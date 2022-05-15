---
layout: post
title:  "Deploying Phoenix Apps to Production"
date:   2022-05-15 22:00:00 +0530
categories: elixir phoenix
---

Phoenix is one of my favorite web application frameworks. There are some excellent platforms
for deploying Phoenix apps to production, including [Fly][fly] and [Gigalixir][gigalixir]. But
there are times when you need to deploy your app to a VM. Here's my recipe for deploying a
Phoenix app to production and doing continuous deployment with Github Actions.

### Packaging the Code
Elixir Releases make it trivial to package the code and ship to production. Phoenix has great
[documentation][deploying] on deploying with releases.

All you have to do here is execute

```shell
mix phx.gen.release
```
which will generate some files to prepare your application for release. The actual release will
be done with Github Actions, so this is all we need here.

### Github Actions

We take an Ubuntu machine and build the Elixir release on it. 

{% raw %}
```yaml
name: Elixir CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    name: Build and Deploy
    runs-on: ubuntu-latest
    env:
      MIX_ENV: prod

    steps:
    - uses: actions/checkout@v2
    - name: Set up Elixir
      uses: erlef/setup-elixir@885971a72ed1f9240973bd92ab57af8c1aa68f24
      with:
        elixir-version: '1.12.3' # Define the elixir version [required]
        otp-version: '24.1' # Define the OTP version [required]
    - name: Restore dependencies cache
      uses: actions/cache@v2
      with:
        path: deps
        key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
        restore-keys: ${{ runner.os }}-mix-
    - name: Install dependencies
      run: mix deps.get
    - name: NPM
      run: cd assets && npm install
    - name: Deploy assets
      run: mix assets.deploy
    - name: Create Release
      run: mix release
```
{% endraw %}
Now we'll make some configurations on the server before transferring the release.

### Server Setup

The first thing to do is to setup a unix user with sudo permission and disable root login.
We then setup the web server and application server.

I like to use Nginx as the web server that forwards the requests to the [Cowboy][cowboy]
server running locally. Here's the Nginx configuration,

```
upstream phoenix {
  server 127.0.0.1:4000 max_fails=5 fail_timeout=60s;
}

server {
  server_name <example.com>;

  location / {
    allow all;

    # Proxy headers
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_set_header X-Cluster-Client-Ip $remote_addr;

    # The Important Websocket Bits!
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_pass http://phoenix;
  }
}
```

With this, Nginx will forward the requests to the Cowboy server listening on port 4000. Its
also important to add SSL, which is super easy to do with [Let's Encrypt][lets-encrypt].

### Phoenix App

I use systemd to run my phoenix app, which makes it run in a managed fashion. Here's my
systemd config, `/etc/systemd/system/<app_name>.service`

```
[Unit]
Description=<App Description>

[Service]
Environment=PHX_SERVER=true
Environment=PHX_HOST=<host>
Environment=PORT=4000
Environment=SECRET_KEY_BASE=<secret_key>
Environment=DATABASE_URL=<db_url>
User=<unix_user>
ExecStart=/home/<unix_user>/_build/prod/rel/<app_name>/bin/<app_name> start

[Install]
WantedBy=default.target
```

It's important to note that we have to provide all the environment variables needed for our
Phoenix app in the service file itself.

### Deployment

Now we can transfer the release built in our Github action to the server. We'll create an
SSH key pair, add the public key to the `authorized_keys` on the server, and store the private
key in Github secrets.

We'll also need to allow our user to be able to start and stop the service from Github Actions.
For this, we'll need to create a file `/etc/sudoers.d/<app_name>`

```
%<unix_user> ALL= NOPASSWD: /bin/systemctl start <app_name>.service
%<unix_user> ALL= NOPASSWD: /bin/systemctl stop <app_name>.service
%<unix_user> ALL= NOPASSWD: /bin/systemctl restart <app_name>.service
```

Also, we'll need to add our Phoenix app environment variables to the file `/etc/environment`

```
PHX_SERVER=true
PHX_HOST=<host>
PORT=4000
SECRET_KEY_BASE=<secret_key>
DATABASE_URL=<db_url>
```

With this, we can now update our Github Action to deploy the release to our server. Add the
following steps to the github action file,  

{% raw %}
```yaml
    - name: Copy files to server
      uses: appleboy/scp-action@master
      with:
        host: ${{ secrets.PROD_HOST }}
        username: ${{ secrets.PROD_USERNAME }}
        key: ${{ secrets.PROD_SSH_KEY }}
        source: "_build/prod/rel/<app_name>"
        target: "~/"
    - name: Run migrations
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.PROD_HOST }}
        username: ${{ secrets.PROD_USERNAME }}
        key: ${{ secrets.PROD_SSH_KEY }}
        script: ~/_build/prod/rel/<app_name>/bin/<app_name> eval "<AppName>.Release.migrate"
    - name: Restart Server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.PROD_HOST }}
        username: ${{ secrets.PROD_USERNAME }}
        key: ${{ secrets.PROD_SSH_KEY }}
        script: sudo systemctl restart <app_name>.service
```
{% endraw %}

And that's it. With this, every time we push the code to the main branch, it is packaged
into a release, transferred to our server and deployed.

[deploying]: https://hexdocs.pm/phoenix/releases.html
[fly]: https://fly.io
[gigalixir]: https://gigalixir.com/
[cowboy]: https://hex.pm/packages/cowboy
[lets-encrypt]: https://letsencrypt.org/
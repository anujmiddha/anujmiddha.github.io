---
layout: post
title:  "Replace DOM elements in Phoenix LiveView"
date:   2022-05-12 10:00:00 +0530
categories: elixir phoenix
---

Phoenix LiveView is an amazing tool to build real-time applications with server
rendered HTML. When the state changes, LiveView takes care of re-rendering the changes
and pushing them to the browser.

For one of my recent projects, I needed to show a list of `<video>` elements on a page.
Initialising the video required a call to

```
videoManager.attachVideo(videoTrack, element)
```

This is easily achieved via `phx-hook`. I simply add the `videoTrack` as a data attribute
to the element,

```
<video id={video_id}
       data-video-track={video_track}
       phx-hook={PlayVideo}/>
```

and initialise in the callback.

```
// app.js
const videoManager = ...
let Hooks = {}
Hooks.PlayVideo = {
  mounted() {
    videoManager.attachVideo(this.el.getAttribute("data-video-track"), this.el)
  }
  updated() {
    videoManager.attachVideo(this.el.getAttribute("data-video-track"), this.el)
  }
}
let liveSocket = new LiveSocket("/live", Socket, {hooks: Hooks, ...})
```

Great! This should _initialise_ and _update_ the video whenever the track changes in the
LiveView. Except, the update doesn't work.

The problem is, every time the track changes, `videoManager` requires a brand new
video element. Re-attaching to an existing `<video>` doesn't work. Now LiveView,
being efficient, only updates the attribute of the existing element, not create a new one.

The solution I implemented, thanks to the advice from Ben Wilson on the Elixir slack,
was attaching the hook to the video’s parent DOM element, and recreating the video element
in the callback. So the new HTML is

```
<div id={video=id}
     data-video-track={video_track}
     phx-hook={PlayVideo}>
  <video ...>
</div>
```

And the app.js is now

```
const videoManager = ...
let Hooks = {}
Hooks.PlayVideo = {
  mounted() {
    const video = this.el.children[0]
    videoManager.attachVideo(this.el.getAttribute("data-video-track"), video)
  }

  updated() {
    const oldVideo = this.el.children[0]
    const newVideo = oldVideo.cloneNode(false)
    videoManager.attachVideo(this.el.getAttribute("data-video-track"), newVideo)
    this.el.replaceChild(newVideo, oldVideo)
  }
}
let liveSocket = new LiveSocket("/live", Socket, {hooks: Hooks, ...})
```

The important piece here is that we clone the old video to get the new video. It
ensures that LiveView recognises it and doesn't replace it.
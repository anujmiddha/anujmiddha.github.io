---
layout: post
title:  "Single Table Inheritance in Rails"
date:   2021-06-30 10:00:00 +0530
categories: ruby rails
---

Many a times we come across situations where we have slight variations in the logic for
different use cases of essentially the same model.

For example, lets say we are building an online book store. This store sells both e-books
and physical books, and supports multiple shipping methods.

{:refdef: style="text-align: center;"}
![image](/assets/images/sti/online_book_store.png)
{: refdef}

The supported shipping methods for e-books are `Email` and for physical books are
`Pickup` and `Post`. We could model this using just 2 tables, `Item` and `ShippingMethod`
with a lot of `if ... else ...` statements

```
class Book
  def supported_shipping_methods
    if ebook?
      [:email]
    else
      [:post, :pickup]
    end
  end  
end

class ShippingMethod  
  def process
    case type
    when :email
      send_email
    when :post
      send_post
    when :pickup
      send_for_pickup
    end
  end
end
```
As you can guess, this quickly devolves into a mess. One powerful technique we can use
to organize such code is [Single Table Inheritance][1], or STI for short.

Rails makes it really simple to implement STI. For the above example, we’ll have a table
named `books` with a column named `type`

```
create_table :books do |t|
  t.string :name
  t.integer :price
  t.string :type, null: false

  t.timestamps
end
```

And for our models, we will have

```
class Book < ActiveRecord::Base
  # All the common functionality can go here
end

class EBook < Book
  # Only methods that are specific to the type of book go here
  def supported_shipping_methods
    [:email]
  end
end

class PhysicalBook < Book
  def supported_shipping_methods
    [:post, :pickup]
  end
end
```

Now when I do any query on EBook or PhysicalBook, ActiveRecord automatically appends
the `type` query for me. For example, when doing `EBook.create(attrs)`, we do not need
to add the `type` attribute. Similarly, we can query different models,

```
Book.all          # Lists all books
EBook.all         # Lists all books with type: EBook
PhysicalBook.all  # Lists all books with type: PhysicalBook
```

We can even create associations with the specific sub classes.

```
class Store < ActiveRecord::Base
  has_many :books
  has_many :e_books
  has_many :physical_books
end
```
and Rails will handle adding the `type` to the query itself !!

P.S. Rails 6 introduced an interesting new concept called [Delegated Types][2]. While it solves a different use case than STI, do give it a look before deciding on implementing STI in your apps.

[1]: https://guides.rubyonrails.org/association_basics.html#single-table-inheritance-sti
[2]: https://github.com/rails/rails/pull/39341
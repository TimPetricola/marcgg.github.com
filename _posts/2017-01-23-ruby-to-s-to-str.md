---
layout: post
title: "The Difference Between to&#95;s & to&#95;str In Ruby"
description: "A lot of people assume that to_s and `to_str` are the same methods, but the difference between them is actually quite major."
blog: true
category: blog
tag: Ruby
---

If you ever looked at the available methods on some objects in Ruby, you might have noticed that there seems to be two difference ways to cast an object to string: `to_s` and `to_str`. However, most people only use `to_s`... is it because it's shorter or is `to_str` functionnally different?

Short answer: they are indeed different.

_Note that this article focuses on `to_s` vs  `to_str`, but the logic applies as well to `to_int`, `to_ary`, `to_hash` and `to_sym`._

## Difference In Scope Of Definition

First of, `to_s` is defined on `BasicObject` which is quite a big deal since `BasicObject` is the [parent class of all classes in Ruby][1]. This means that all classes in Ruby implement `to_s`. 

	> BasicObject.to_s
	 => "BasicObject"

To demonstrate this, let's create a new class `Demo` and call `to_s` on a new instance of this class:

{% highlight ruby %}
class Demo
end

puts Demo.new.to_s
{% endhighlight %}

This returns:

	#<Demo:0x007fc49b05a408>

We could also call `to_s` directly on the class and get a valid result as well since the `Class` [class is also an object][2].

However if we try to do the same with `to_str`, it won't work because it's not defined on a higher level class:

{% highlight ruby %}
puts Demo.new.to_str
{% endhighlight %}

This will return `undefined method to_str for #<Demo:0x007fea8204e290> (NoMethodError)`, meaning that we have to define it ourselves when creating a new class.

## Difference In Behavior

Alright, so the methods are not defined in the same way... but this is just a minor detail compaired to the main difference between the two: 

- `to_s` returns a string representation of an object
- `to_str` is actually stating that the object behaves like a string!

_Note that both `to_s` and `to_str` will return an instance of `String`, even when subclassing `String`._

### String Representation Of An Object

When calling `to_s`, it will return some form of string representation of the object. Something easy to display.

When creating a new class you can keep the default behaviour or build your own. For instance here's how `to_s` behaves on an integer:

{% highlight ruby %}
100.to_s(2) # returns "1100100"
100.to_s # returns "100"
100.to_s(8) # returns"144"
{% endhighlight %}

Here's what it could look like if you were defining it yourself:

{% highlight ruby %}
class User
  def initialize(name)
   @name = name
  end

  def to_s
   "<User: #{@name}>"
  end
end
{% endhighlight %}

It's basically just a way to have a quick and nice way to display your objects that is going to be called when needed, for instance when using `puts` or concatenating with  `#{}`:

{% highlight ruby %}
puts "Here is #{User.new("Bob")}" # Returns "Here is <User: Bob>"
{% endhighlight %}

### Behaving Like A String

Calling `to_str` should return a string-like object, behaving just like a `String`.

On the other hand when an object implements `to_str`, it has way more consequences as it means that it will return something that behaves like a string. It's basically saying that the class is not necesseraly a `String`, but it can be used in the same way. 


Because of this, the only class in Ruby core implementing `to_str` is `String`:

{% highlight c %}
rb_str_to_s(VALUE str)
{
 if (rb_obj_class(str) != rb_cString) {
    return str_duplicate(rb_cString, str);
 }
 return str;
}
{% endhighlight %}

[Exception][3] used to implement `to_str` as well but is was removed in Ruby 1.9, which is why it's often mentionned as an example. The Ruby documentation was even wrong at the time of writing this article, so I wrote [a PR to fix it][4] that was merged.

There are a lot of discussions regarding if a class should implement `to_str` or not,  since it's a strong signal that the class is really similar to a string and should behave as such. If this sounds interesting, you should take a look at this [Symbol#to_str][5] discussion on the Ruby core tracker, or at [this example in Rails][6] of when `to_str` is useful by [Aaron Patterson][7].

#### Example 1: Fixnum

`Fixnum` does not define `to_str`, so when we do this:

{% highlight ruby %}
puts "150" + 42
{% endhighlight %}

We get `'+': no implicit conversion of Fixnum into String (TypeError)`. This makes sense, but what would happen if we were to define `to_str` on the `Fixnum` class?

{% highlight ruby %}
class Fixnum
  def to_str
   self.to_s
  end
end
 {% endhighlight %}

The when we call:

{% highlight ruby %}
puts "150" + 42
{% endhighlight %}

We get `"15042"`, which is quite surprising! It's because `42` was implicitely converted to a `String`.

#### Example 2: User

Going back to our other example, we could add `to_str`  to the `User` class and get the ability to use `+` to concatenate an instance of `User` with a string:

{% highlight ruby %}
class User
  def initialize(name)
   @name = name
  end

  def to_str
   @name
  end
end

puts "Say hello to " + User.new("Bob") # Displays "Say hello to Bob"
{% endhighlight %}

## Summing It Up

That's a lot of information, but let me sum it up quickly before finishing the article:

- `to_s` and `to_str` are **very different**.
- `to_s` is defined on most objects and returns a string representation of this object.
- Defining `to_str` on an object is very much like saying "this object behaves like a string".
- Calling `to_str` should return a string-like object, not juste a representation of the object as a string.


[1]:	https://ruby-doc.org/core-2.2.0/BasicObject.html
[2]:	https://ruby-doc.org/core-2.2.0/Class.html
[3]:	http://apidock.com/ruby/Exception/to_str
[4]:	https://github.com/ruby/ruby/pull/1517
[5]:	https://bugs.ruby-lang.org/issues/7849
[6]:	https://github.com/rails/rails/commit/188cc90af9b29d5520564af7bd7bbcdc647953ca
[7]:	https://twitter.com/tenderlove

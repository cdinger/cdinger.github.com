---
layout: default
title: jquery-objectdiff — a jQuery plugin for comparing objects
lead: "I recently worked on a javascript project that involved a simple workflow where an unprivileged user could request changes to certain bits of enterprise data. The requested changes would then route to a privileged user who would either make the requested changes or deny the request. I wanted to provide a some kind of diff to the privileged user so they didn’t have to compare two sets of data and manually find the changes. It’s 2011—no human should *ever* have to manually perform a diff!"  
---
## jquery-objectdiff — a jQuery plugin for comparing objects

I recently worked on a javascript project that involved a simple workflow
where an unprivileged user could request changes to certain bits of enterprise
data. The requested changes would then route to a privileged user who would
either make the requested changes or deny the request. I wanted to provide a
some kind of diff to the privileged user so they didn’t have to compare two
sets of data and manually find the changes. It’s 2011—no human should *ever*
have to manually perform a diff!

In Rails, I’d be able to reflect on the [ActiveRecord::Dirty::changes](http://ar.rubyonrails.org/classes/ActiveRecord/Dirty.html#M000291)
hash to see which attributes were modified and their before and after values. I
love how it works and I searched for a javascript diff’er that did something
similar. There are a few out there, but none worked quite like I wanted, so I
decided to roll my own. objectDiff() will recursively compare two javascript
objects and return an ActiveRecord-style changes object. Check it:

### Usage

Suppose you have two javascript objects (like serialized form data):

{% highlight javascript %}
var before = {
  "id": 123,
  "name": {
    "first": "Johnny",
    "last":"Johnson"
  }
};

var after = {
  "id": 123,
  "name": {
    "first": "John",
    "last": "Johnson"
  },
  "age": 30
};
{% endhighlight %}

A call to `objectDiff(before, after)` would return an object of only the changes:

{% highlight javascript %}
{"name": {"first": ["Johnny", "John"]}, "age": [null, 30]}
{% endhighlight %}

I hope this is useful to someone. It’s wrapped up in a jQuery plugin on Github:
[jquery-objectdiff](https://github.com/cdinger/jquery-objectdiff).

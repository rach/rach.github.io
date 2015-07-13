---
title:  "Virtualenvwrapper plugins for gems and npms"
date:   2015-07-10
description: Plugins to make it easy and transparent to keep gems and npms inside your virtualenv  
share: true
---

    
In a previous [post]({% post_url 2015-07-09-keep-your-gems-and-npms-inside-your-virtualenv %}), we've seen how to modify your activate script to keep inside a virtualenv the packages from Node and Ruby called gems and npms. I find useful to be able to try gems or npms inside a virtualenv without contaminate your setup.  

As virtualwrapper is offering a way to extend it via plugins.
I wrote two small plugins for handling the env variables in a nicer way and doing some cleaning when you deactivate the virtualenv. 

This post assume that you are using [virtualenvwrapper](http://www.doughellmann.com/projects/virtualenvwrapper/), otherwise I advice you to check it first.

Ironicaly, this packages cannot be installed inside a virtualenv but as to be installed globally as virtualenvwrapper.

##virtualenvwrapper gem plugin   

There is a lot of very useful ruby packages like foreman, jekyll, sass ... 

###Download and install

I encourage you to install it via `pip` :

{% highlight sh %}
pip install virtualenvwrapper.gem 
{% endhighlight %}



###How to use it

{% highlight sh %}
mkvirtualenv test
workon test #if not already inside
gem install sass
which sass # should refer to $VIRTUAL_ENV/lib/gems/bin/sass
{% endhighlight %}

 
##virtualenvwrapper plugin for gem 

Node has some amazing tools to handle assets like webassets, gulp, ...


###Download and install:

I encourage you to install it via `pip`: 

{% highlight sh %}
pip install virtualenvwrapper.npm
{% endhighlight %}


###How to use it


{% highlight sh %}
mkvirtualenv test
workon test #if not already inside
npm install -g gulp
which gulp # should refer to $VIRTUAL_ENV/bin/gulp
{% endhighlight %}





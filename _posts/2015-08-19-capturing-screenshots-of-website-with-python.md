---
title:  "Capturing screenshots of website with Python"
date:   2015-08-19
description: There are solutions depending on PyQt, this post describe an easier one depending of PhantomJS.      
---

If you google how to do a screenshot of a website then you will probably end 
with an answer depending on PyQT and WebKit. Sadly, PyQt is not a python package
that you can install easily via `pip` and you will few others dependencies
to be able to generate a screenshot of a site.

This post describes an easier solution depending only on Selenium and PhantomJS. 
Selenium can easily be installed by `pip` and phantomJS via your package manager or `npm`. On OSX, it's just a matter of doing:

{% highlight sh %}

brew install phantomjs # or npm -g phantomjs 
pip install selenium

{% endhighlight %}

At the opposite of installing PyQT, it should be without problems.
Once you have Selenium and PhantomJS installed then it becomes simple 
to write the small python code required to generate the screenshot of a website. 
{% highlight python %}

from selenium import webdriver
depot = DepotManager.get()
driver = webdriver.PhantomJS()
driver.set_window_size(1024, 768) # set the window size that you need 
driver.get('https://github.com')
driver.save_screenshot('github.png')

{% endhighlight %}

The code above is all you need to generate a screenshot of a website. Webdriver offers others API like `get_screenshot_as_png()` which return a binary and can be useful to create an image in memory if you want to manipulate before saving it. 


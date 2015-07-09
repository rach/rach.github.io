---
title:  "Keep your gems and npms inside your virtualenv"
date:   2015-07-09
description: Nobody want having their gems or npms wondering around. Keep them tidy inside your virtualenv.  
---

Today, when building web application it becomes necessary to embracce multiple technologies to use the best tool for the task. Eg: Node for the assets pipeline.  

If you are using Python then you are probably used to work with [virtualenv](virtualenv). When working with virtualenv then the last thing that you want is installing dependencies globaly and you want to contain them per project.

Taming your gems and npms to force them of existing only in your virtualen is very straightforward as `gem` and `npm` support environment variable to tell them where to find things.

To do so, copy the lines below at the end of the ‘activate’ script of the virtualenv. If you are using [virtualenvwrapper](virtualenvwrapper) then you copy it inside the ‘postactivate’ script.

{% highlight sh %}
    
export GEM_HOME="$VIRTUAL_ENV/gems"
export GEM_PATH=""
export PATH="$GEM_HOME/bin:$PATH"
export npm_config_prefix=$VIRTUAL_ENV

{% endhighlight %}

And now inside your virtualenv, you can do :

{% highlight sh %}
    
gem install <package>
npm -g install <package>

{% endhighlight %}

And all yours gems/npms will be installed in your vitualenv and will be deleted with it.

If you are using [direnv](direnv) then you can copy the script into your `.envrc`.

This is an updated version of an older post from my previous blog but the infos is still actual as I'm still using a similar technique when jungling between different projects. 

This technique as the trade off of not doing any tidying after itself when deactivating the virtualenv and keeping the environment variables. I'm using virtualenwrapper so I developed 2 small plugins: [virtualenv.npm](virtualenv-npm), [virtualenv.npm](virtualenv-npm)); which allows to do the same but also handle the cleaning when you deactivate the virtualenv.

[virtualenv]:   https://pypi.python.org/pypi/virtualenv 
[virtualenvwrapper]:   https://pypi.python.org/pypi/virtualenvwrapper 
[virtualenv-npm]:    https://pypi.python.org/pypi/virtualenv.npm
[virtualenv-gem]:    https://pypi.python.org/pypi/virtualenv.gem
[direnv]:   http://direnv.net/ 

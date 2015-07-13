---
title:  "Using Git Subtree to create branches from a directory"
date:   2015-07-11
description: It's sometimes usefull to have an asymetric branch which is represent an output folder  
share: true
---

This post is about using ``git subtree merge`` to have a branch that is a subset of another branch. 
Github will be a use case and act as a way to illustrate this feature, there are probably other
ways to accomplish the same result.

Github has a way for users to host and serve static web pages
which is very pratical if you need to create a page for your open source project
or create a blog like the one that you are reading now!

If you have never heard about Github Pages then the first place to start is, of course,
the official docs: `Github Pages <http://pages.github.com/>`_.
But if you are interested in building a custom page, I found most of my answers
on their `help page <https://help.github.com/categories/20/articles>`_.

A quick summary ...

There is 3 types of Github page: *User* page, *Organisation* page and *Project* page

- The User and Organisation page share the same behavior :

        The static page is read from the ``master`` branch in the repo called ``username.github.com`` 
        and for organisation ``organisation.github.com``.

        Which means you should have some something like this for the repo url for a User Page:

        ::

                [SSH]     git@github.com:username/username.github.com.git 
                [HTTPS]   https://github.com/username/username.github.com.git

        For organisation:

        ::

                [SSH]     git@github.com:organisation/organisation.github.com.git 
                [HTTPS]   https://github.com/organisation/organisation.github.com.git

        The website will be accesible via ``username.github.com`` or ``organisation.github.com`` and
        it will read the index.html inside the root folder.

        eg: You can find the repo for this blog on https://github.com/rach/rach.github.com and view it as web page via http://rach.github.com/

- The Project Page is slightly different :

        Static pages are read from the ``gh-pages`` branch in your ``projectname`` repo and access to the website is via ``username.github.com/projectname``.


Now we have established the problem, maybe for some reason you don't want to have
your static website in the root folder. In my case, I'm using `Pelican <http://pelican.notmyidea.org/>`_ to build this blog 
and I want to have the static website in a directory that I specify.
I won't be specific to `Pelican <http://pelican.notmyidea.org/>`_ so if you are using anything to generate your pages and you don't want to expose it directly (Markdown, Less, Rst, Template), this should remain informative.

Let's assume for the following that we are building a User Github Page and
you have a source and output folder as below:

::

        (root)
           |-> output
           |-> source
        

Here, we would like to have the master branch focused on the output folder so it will be served by Github Pages. A perfect use case for subtree merge. 
The doc on git subtree merge say:

| *"There are situations where you want to include contents in your project
  from an independently developed project. 
 You can just pull from the other project as long as there are no conflicting paths.*
| ...
| *What you want is the subtree merge strategy, which helps you in such a situation."*
  `ref <http://www.kernel.org/pub/software/scm/git/docs/howto/using-merge-subtree.html>`_

  
  
In our case we are not going to pull from another project, but the idea is the same. 
Or in other words, we want to have the root of ``master`` branch pointing the ``output`` 
of the ``source`` branch and for it to be easy to update.
Hence, we are going to merge the ``output`` folder of the ``source`` branch of same repository into the ``master``.

Here is the command sequence I used: 

.. code-block:: none

        (source)$ git checkout --orphan master
        (master)$ git rm -rf .
        (master)$ git merge -s ours --no-commit origin/source
        (master)$ git read-tree -m -u source:output
        (master)$ git commit -m "Merge folder output of branch source"


1. Create an orphan branch, which is a branch without a parent.
2. Remove any previously tracked element.
3. Prepare for the later step which will record the result as a merge.
4. Read folder ``output`` of branch ``source`` into the current directory.
5. Commit the merge.

Enjoy your website on ``http://username.github.com``

.. note:: For maintaining the result,  merges using "subtree" 
      
        ::

                 git pull -s subtree origin source

.. note::  Alternative to create an empty branch

        ::

                true | git mktree | xargs git commit-tree | xargs git branch master

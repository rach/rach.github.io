---
title:  "Handling Go workspace with direnv"
date:   2015-06-28
description: Bringing some sanity into your go project workspaces 
share: true
---

When I started to do some [Go](go-lang) I quickly hit my first hurdle: [The Go Workspace](go-workspace). The go tool is designed to work with code maintained in public repositories using the (FQDB)[FQDN] and path as namespace and package name. Eg: `github.com/rach/project-x` where `github.com/rach` is kind of namespace enfore by a directory structure and `project-x` is the package name also enforce by directory structure.

Coming from Python, I was suprised that there were not a solution as simple as [virtualenv](virtualenv). Go does offer a solution but it requires a bit more code gymnastic. 

In this post, I'm going to describe how I made my life easier to work with Go with a bit of shell script and using [direnv](direnv) to automate workspace switching. I didn't know much about go when I wrote this post so feel free to shed some light on any of my mistakes.   


##Workspaces

Go project must be kept inside a workspace. A workspace is a directory hierarchy with few directories:

- `src` contains Go source files organized into packages (one package per directory),
- `pkg` contains package objects, and
- `bin` contains executable commands. 

The go tool builds source packages and installs the resulting binaries to the pkg and bin directories.

The `src` subdirectory typically contains multiple version control repositories (such as for Git or Mercurial) that track the development of one or more source packages.

To give you an idea of how a workspace looks in practice, here's an example:

    bin/
        hello                          # command executable
        outyet                         # command executable
    pkg/
        linux_amd64/
            github.com/golang/example/
                stringutil.a           # package object
    src/
        github.com/golang/example/
            .git/                      # Git repository metadata
          	hello/
          	    hello.go               # command source
          	outyet/
          	    main.go                # command source
          	    main_test.go           # test source
          	stringutil/
          	    reverse.go             # package source
          	    reverse_test.go        # test source

The problem that I hit was: 

- how do you work on multiple different projects?
- how should specify which workspace that I working on?

It's when the `GOPATH` enter to define the workspace location.   

##The GOPATH environment variable

The GOPATH environment variable specifies the location of your workspace. To get started, create a workspace directory and set GOPATH accordingly. Your workspace can be located wherever you like.

{% highlight sh %}
$ mkdir $HOME/go
$ export GOPATH=$HOME/go
{% endhighlight %}

To be able to call the binary build inside your workspace, add bin subdirectory to your PATH:

{% highlight sh %}
$ export PATH=$PATH:$GOPATH/bin
{% endhighlight %}


For you project can choose any arbitrary path name, as long as it is unique to the standard library and greater Go ecosystem. It's the convention to use a FQDN and path as your folder structure which will behave as namespaces.

We'll use `github.com/rach/project-x` as our base path. Create a directory inside your workspace in which to keep source code:

{% highlight sh %}
$ mkdir -p $GOPATH/src/github.com/rach/project-x
{% endhighlight %}

##Update the GOPATH automaticaly with direnv

Direnv is an environment switcher for the shell. It load or unload environment variables depending on the current directory. This allows to have project-specific environment variables. direnv works with bash, zsh, tcsh and fish shell. Direnv checks for the existence of an ".envrc" file in the current and parent directories. If the file exists, the variables declared in `.envrc` are made available in the current shell. When you leave the directy or sub-directory where .envrc is present the variable are unloaded. It also works well with updating existing environment variable.

To install direnv on OSX using zsh, you can follow this steps: 

{% highlight sh %}
$ brew update
$ brew install direnv
$ echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
{% endhighlight %}

Using direnv, it becomes easy to have multiple workspaces and switch between them. Simply create a `.envrc` file at the location of your workspace and export the appropriate variable:

{% highlight sh %}
$ mkdir $HOME/new-workspace
$ cd $HOME/new-workspace
$ echo 'export GOPATH=$(PWD):$GOPATH' >> .envrc
$ echo 'export PATH=$(PWD)/bin:$PATH' >> .envrc 
$ direnv allow
{% endhighlight %}

With the code above we now have a workspace which enable itself when you enter it. 
Having multiple workspace help to experiment with libs/package that you want to test in the same way you can install a python lib just for a one-time use.

Assuming we will be writing a lot of go projects, will not be nice of a having an helper to 
create this automated workspaces following the suggested structure.

###Automate workspace creation

Now that we know how a workspace should look like and how to make switching them easier. Let's automate the creation new project with workspaces to avoid mistakes, for that I wrote a small `zsh` function to do it for me. 

{% highlight sh %}
function mkgoproject {
  TRAPINT() {
    print "Caught SIGINT, aborting."
    return $(( 128 + $1 ))
  }
  echo 'Creating new Go project:'
  if [ -n "$1" ]; then
    project=$1
  else
    while [[ -z "$project" ]]; do 
      vared -p 'what is your project name: ' -c project; 
    done
  fi
  namespace='github/rach'
  while true; do 
    vared -p 'what is your project namespace: ' -c namespace 
    if [ -n "$namespace" ] ; then 
       break
    fi
  done
  mkdir -p $project/src/$namespace/$project
  git init -q $project/src/$namespace/$project
  main=$project/src/$namespace/$project/main.go
  echo 'export GOPATH=$(PWD):$GOPATH' >> $project/.envrc
  echo 'export PATH=$(PWD)/bin:$PATH' >> $project/.envrc
  echo 'package main' >> $main 
  echo 'import "fmt"' >> $main
  echo 'func main() {' >> $main
  echo '    fmt.Println("hello world")' >> $main 
  echo '}' >> $main
  direnv allow $project
  echo "cd $project/src/$namespace/$project #to start coding"
}
{% endhighlight %}

If you are using zsh then you should be able to copy/paste this function into 
your zshrc and after reloading it then you be able to call `mkgoproject`. 
If you call the function with an argument then it will consider it being 
the project name and it will ask you for a namespace (eg: github.com/rach), otherwise it will ask you for both: project name (package) and namespace. 
The create a new worspace with an envrc and helloword ready to build within 
the correct directory structure.

{% highlight sh %}

$ mkgoproject test
Creating new Go project:
what is your project namespace: github/rach
cd test/src/github/rach/test #to start coding

{% endhighlight %}

I hope this post will help you into automate the switching between your go project and the creation of them.  


[go-workspace]:   https://golang.org/doc/code.html#Workspaces 
[go-lang]:   https://golang.org/ 
[FQDN]: https://en.wikipedia.org/wiki/Fully_qualified_domain_name

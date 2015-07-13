---
title:  "Handling Go workspace with direnv"
date:   2015-06-28
description: Bringing some sanity into your go projects 
share: true
---

You'll find this post in your `_posts` directory - edit this post and re-build (or run with the `-w` switch) to see your changes!
To add new posts, simply add a file in the `_posts` directory that follows the convention: YYYY-MM-DD-name-of-post.ext.

Jekyll also offers powerful support for code snippets:

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


[go-project]:    https://blog.golang.org/organizing-go-code

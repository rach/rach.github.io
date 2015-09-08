---
title:  "Embedding static assets in a Go project"
date:   2015-09-07
description: If you want to ship a Go webapp as a binary, this post shows how to embed static assets (images, html, ...)     
---

I wanted to build a small self-contained web application in Go, 
at the opposite of an usual webapp where the assets will be serve separatly 
via a CDN or HTTP server like nginx. 
But if performance doesn't matter or it's aimed for a small traffic then having a 
self-contained application makes it easier to deploy and to distribute as it's simply a binary to run.  

There is few packages which allow to embed assets into a Go application:

- [Rice][rice]
- [Statik][statik]
- [Bindata][bindata]

I'm not going to go into the details of each library but I prefered the approach of bindata, simpler to get started, and actively maintained. Also the step to embed is very explicit in the code.


First, you will need to have a Go workspace setup.
If you are unfamilliar with Go then you can refere to the [documentation][go-workspace] or my previous [post]({% post_url 2015-06-28-handling-go-workspace-with-direnv %}) on the topic.

Once you got a Go workspace and project setup then let's create an `index.html` inside a `frontend/` directory within your project. A simple _'Hello World'_ will do: 

{% highlight html %}
<!-- frontend/index.html -->
<html>
  <body>
    Hello, World!
  </body>
</html>
{% endhighlight %}

Now that we have a project setup and a static asset to test, let's install [bindata][bindata] via:

    go get -u github.com/jteeuwen/go-bindata/...

We are good to start the webapp backend code. Create `main.go` file and copy the following code:

{% highlight go %}
package main

import (
	"bytes"
	"io"
  "net/http"
)

//go:generate go-bindata -prefix "frontend/" -pkg main -o bindata.go frontend/...

func static_handler(rw http.ResponseWriter, req *http.Request) {
  var path string = req.URL.Path
  if path == "" {
    path = "index.html"
  }
  if bs, err := Asset(path); err != nil {
    rw.WriteHeader(http.StatusNotFound)
  } else {
    var reader = bytes.NewBuffer(bs)
    io.Copy(rw, reader)
  }
}

func main() {
  http.Handle("/", http.StripPrefix("/", http.HandlerFunc(static_handler)))
	http.ListenAndServe(":3000", nil)
}

{% endhighlight %}

The important line in this code is: 

    //go:generate go-bindata -prefix "frontend/" -pkg main -o bindata.go frontend/...

 the line above allow us to run the `go-bindata` command line when we are calling `go generate`.
From Go 1.4, it's allowed to run custom commands during a [generate][generate] step. It's just a matter of adding `//go:generate command argument...` inside your go file.   

The command line `go-bindata` have multiple options so check the [documentation][bindata-usage] on how to use it. In our case, we are telling bindata few things:

   -  `-prefix "frontend/"` to define a part of a path name to be stripped off   
   -  `-pkg main` to define the package name used in the generated code 
   -  `-o bindata.go` to define the name of the generated file

Once the `go generate` command has run, you should see a file called `bindata.go` being generated. Your project's structure should look like this:

    .
    │ 
    ├── bindata.go (auto-generated file)
    ├── frontend
    │   └── index.html
    └── main.go

The logic to serve the static asset is in the `static_handler` function, 
which receive a request and check if the path matchs an asset. The check is done using the function `Asset` automatically exported by `bindata.go`. If the asset doesn't exist then we return a 404 otherwise we return the content of the asset.


The rest of the code is to create the web application and associate our `static_handler` with a pattern `/` to match all entering requests. If you've got problems understanding this code then check the `http` package [documentation][go-http].

As quick reminder about how Go handle packages, all identifiers will be automatically exported to other packages of the same name if the first letter of the identifier name starts with an uppercase letter. 

Base on this rule, `bindata.go` file expose an `Asset` function to the `main` package. This loads and returns the asset for a given name. 
It returns an error if the asset could not be found or could not be loaded.

This post doens't go into advanced details but I may write other posts as I'm progressing in my application.

[statik]:   https://github.com/rakyll/statik
[rice]:    https://github.com/GeertJohan/go.rice
[bindata]:    https://github.com/jteeuwen/go-bindata
[generate]:    http://golang.org/cmd/go/#hdr-Generate_Go_files_by_processing_source
[go-workspace]:   https://golang.org/doc/code.html#Workspaces 
[bindata-usage]:   https://github.com/jteeuwen/go-bindata#usage
[go-http]:   http://golang.org/pkg/net/http/

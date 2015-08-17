---
title:  "Embedding static assets in Go project with Go bindata"
date:   2015-07-19
description: In the case you want to ship a small webapp as a binary, it's useful to know how to embed static assets (images, html, ...)     
---

I wanted to build a small self-contained web application in Go. At the opposite of an usual webapp where the assets will will be serve separatly. Having a self-contained application makes it easy to deploy and distribute as it's simply a binary to run.  

There is few packages which offer solution for embedding assets into a go application:

- Rice
- Statik
- Go bindata

I'm not going to go into the details of each solution but I found the approach of go bindata being simple and the way of converting the assets being explicit.

From go 1.4, it's allow to run custom command line during a specific step (** link to go doc about `go:` )   

{% highlight go %}
package main

import (
	"bytes"
	"io"
	"net/http"
  "html"
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



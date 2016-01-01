---
title:  "Introducing Suma"
date:   2015-12-31
description: Suma is microservice to manage, control, preview links to external urls within your project.
share: true
---

I'm excited to announce the release of the first version of Suma. Suma stands for **S**hort **U**RL **M**anagment **A**pp.
The role of Suma to manage external links and extract data from them. Suma is a small web service to do easily the followings:

- Creating short URL for external link within your application
- Extracting Title
- Capturing Screenshot from URL 
- Blocking URL's
- Collecting clicks

If your application needs to display external links then it's probably important that you don't redirect the user directly to the URL so you can fight spam, phishing attacks or inappropriate links.


##Use cases

If you don't understand directly what Suma is for, let's illustrate it with few use cases:

- Public Feeds (eg: Twitter or FB like app) which allow user to post link publicly
- Reviews or comments allowing external links
- Display link title or screenshot to preview an external link within your application (eg: slack like app)

To summarize: if your application allows external links from users, then Suma can be useful.

##Technology

Suma is built in Python using [Pyramid](http://www.pylonsproject.org/). If you used to deploy a WSGI app then it should not take you more than few minutes to run it. Otherwise you may have to do some digging until I improve the documentation on deployment.

##What's next

There are a few things that I would like to add to this project:

- Download the favicon
- Customize the behavior of 404 (eg: redirect to a specific url)
- Add option to choose a 403 when a link has been banned
- Add alternative to Goose for extracting content using [diffbot](https://www.diffbot.com/) or [libextract](https://github.com/datalib/libextract)
- Supporting css selector on the html via the API


##License

Suma is licensed under Apache V2 license, the full license text can be found [here](https://github.com/rach/suma/blob/master/LICENSE)

##Motivations

I had the need for a similar project in past and also recently for a new side project. 

Suma is at an early stage of development, but the goal of this project is to provide a microservice which covers the basic need for a company to protect their users from external links within their products. You can read more about the project in the [README](https://github.com/rach/suma)

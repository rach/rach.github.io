---
title:  "Introducing Pome"
date:   2015-12-31
description: Pome is a PostgreSQL Metrics Dashboard to keep track of the health of your db.
share: true
---

I'm excited to announce that I just released the first version of [Pome](https://github.com/rach/pome) (version 0.1.0).
Pome stands for **Po**stgreSQL **Me**trics. Pome is a PostgreSQL Metrics Dashboard to keep track of the health of your database.

PostgreSQL is incredibly stable, especially with small databases. There are a lot PostgreSQL databases in the wild without a DBA to take care of them. In some cases, this could mean things are slowly getting worse. Luckily, a lot of things can be analyzed within postgres to get a health status, but it seemed to miss simple tool to run, which takes no time to install or without dependencies.

## Goals

This project follows 3 principles: Simplicity, Opinionated, Batteries included. 

**Simplicity**, the project aims to be easy to deploy and run. This is why Pome can run as a binary. The project also aimed to feel like the `psql` command and use common arguments. 

**Opinionated**, Pome has the goal to be pre-configured and analyse commonly useful metrics. We want the project to have sensible defaults. In the future, the tool will allow some level of configuration but without compromising Simplicity. 

**Batteries Included**, Pome is built to be accessed via a web interface. The web app is shipped within the binary and Pome takes care of serving the assets (HTML, js, CSS). Pome is not built to be a public facing tool so performance into delivering assets was not a concern. It should be possible to run the frontend individually if it is of concern to you. Pome tries to not to rely on any dependency which cannot be shipped with the binary, and this is one of the reasons why Pome is stateless right now.


##Motivations 

Aside from the goals described above I wanted to start this project for few personal reasons:

- PostgreSQL needs better and easier tooling. People may disagree but tooling is an important part of the success of any technology. Looking around I didn't find a tool sharing the same goals. A tool doesn't need to be perfect to start being useful but they need to be easy and enjoyable to use.
- Learning more about Go. I went with Go because it was suited to ship the tool as a binary. This project gave me a chance to dabble in Go for the first time and have a project to keep improving using this language.
- Improving my experience using React. I used React a while ago but I always mainly work in the backend world. I wanted a project to bring me up to date on the topic.
- Experimenting with D3. I had a few others project for which I was planning to use D3, so it felt best that I iron my skills out on a smaller project first.

##License

Pome is licensed under Apache V2 license, the full license text can be found [here](https://github.com/rach/pome/blob/master/LICENSE)

##What's next

There are a few features that I would like to add to make Pome more useful:

- Allow to configure when the metrics are collected
- Add few others metrics

One of the big things that I will start to work on soon is a new UI mock up, to ensure that tool stay user friendly.
You can read more about the future of Pome in the [README](https://github.com/rach/pome) or within
the opened [Issues](https://github.com/rach/pome/issues).

This project is at a very early stage, and there are a lot of missing features, but I'm using PostgreSQL a lot so I should be able to keep improving the project based on my own need at the start. I'm also hoping that few people will get excited about having a project with these goals, will try it and/or start contributing.

---
title: "Migrating to Hugo"
date: 2021-10-23T08:31:15+01:00
draft: true
tags: ['']
author: "Dave"
toc: true
featured_image: "image/title/image_name.png"
---

Whilst deploying a WordPress server gave me something to demostrate using terraform with AWS , it was a bit overkill for a website hosting static content. This type of blog site does not really need to be backed by a database. 

Based on a colleague setting up a static site using Jekyll I thought I would take a look at migrating this site off WordPress and onto something more suited and if possible, cheaper. In addition to this I figured there must be a more devops way to do it.

The end result was hosting the site on gitpages, content is held in github as markdown. github actions are used to deploy a hugo container, generate the static content from the markdown and upload to a dedicated branch. This branch is then used by git pages. The ssl cert is now also handled by git pages, and its all free!

Options:
There are a number of different opensource projects which are advertised as static site generators. For me I thought I would work through them all but started on hugo and it was pretty straightforward, before I knew it I had a site up and running which was easy to post. There was no real value in me trying others.

I was unable to locate a theme with all the features I required so ended up forking off a theme and making the ammendments I required.



Reading:
Whilst researching the migration I came across a number of useful sources of great content which I have linked below;
https://gohugo.io/tools/migrations/
https://github.com/SchumacherFM/wordpress-to-hugo-exporter

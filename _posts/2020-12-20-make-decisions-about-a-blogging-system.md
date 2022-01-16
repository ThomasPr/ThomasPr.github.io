---
title:  "Make decisions about a blogging system"
date:   2020-12-20 22:43:50 +0100
excerpt: "It was a challenging task to decide which software I want to use for this blog."
---

I'm a person who really thinks twice about the future development of his decisions, especially when it comes to a system that I probably use for years. It took me ages to set up a blog like this, but in the end it was just a matter of minutes. In this very first blog post I want to walk through the rationale that led me to the current setup.

## My Requirements

- Having the content under version control (git)
- No vendor lock-in
- No burden for recurring maintence like security updates
- Use my own domain
- Fully customizable layout and content to fit my needs
- No WYSIWYG-editor, the HTML code looks always a bit creepy


## The Alternatives

There are many services and software out there to get almost the same result. Basically they can be grouped into four categories:


### Software as a Service

Providers who offer a fully-featured system hosted and managed by them. You only need to sign up for their service and can start writing your first post.

The most popular blogging provider is workpress.com who provides a hosting instance of WordPress. Please be aware about the difference of the provider WordPress.com and the software WordPress. Other well-known providers are wix.com or jimdo.com, but there are many more out there.


### Software installed on your own server

Software to be installed and managed by your own on your own server. Mostly, you need a php-powered webspace with a database and upload the software to this webspace. After entering some configuration options you can start blogging. The setup is quite easy and many webspace providers offer a one-click installation where they automated the hassle of installing and configuring the software. However, it's very uncommon to install software updates and offer migrations in a one-click fashion. WordPress is the the most popular software, but only one out of a huge list.


### Static site generators

Since I do not plan to offer any dynamic content like comments, contact forms or user login, the blog itself can be run by static pages only. A static site generator where I create the content locally, generate the web page and upload that to web server is a valid option for my requirements. The generated pages can be uploaded to AWS S3 or even simpler just to GitHub Pages. Static site generators are not that common, but some of the known ones are Jekyll, Hugo or Middleman.


### Develop the software by my own

This feels like re-inventing the wheel once again, but this will be the most flexible and most customizable solution. Since I'm a professional software developer, I have quite some experience to develop my own software.

## My Decision

It took ages from the idea to blog by myself until I finally was able to write this blog post you're reading right now. Let me show you my opinion about the options:

- Software as a Service: This seems to be the worst option, I'm hardly able to move my content, have to use a WYSIWYG editor and be unable to fully customize my blog. Therefore, I won't go for that.
- Software installed on my own server: Thats a much better option, at least I'm able to access the content in the database. But even if I can modify the software and therefore my blog therotically, I don't have much experience with PHP which is used by many blog engines such as WordPress.
- Static site generators: Seems to be a good option, I don't have to deal with security updates, they're usually simple enough for easy layout changes and depending on the hosting I'm able to use my own domain.
- Developing my own software: A very flexible but very time-consuming option. Since I don't want to focus on blogging itself, not on the blogging software, this option is not my preference. Additionally, I still have to deal with security updates, even worse I have to find and fix them by my own.

In the end, I just decided to go for a static site generator: I really like the simplicity and the broad hosting options. Since I already spend so many time to for that decision, I just took the one that I alredy knew from the past: **Jekyll**. For the hosting part I'll go for GitHub Pages who offers an incredible easy way to host static pages and even more actually generate the static pages based of the Jekyll content.

Check out my public repository on GitHub to see final result:
  - The [master](https://github.com/ThomasPr/ThomasPr.github.io/tree/master) branch contains the Jekyll content
  - The [gh-pages](https://github.com/ThomasPr/ThomasPr.github.io/tree/gh-pages) branch contains the generated static pages which are visible in the blog

Comments are welcome on [Twitter](https://twitter.com/TheThomasPr/status/1340778103929524234) or [LinkedIn](https://www.linkedin.com/posts/thomas-preissler_make-decisions-about-a-blogging-system-activity-6857657519179931648-4RHN).


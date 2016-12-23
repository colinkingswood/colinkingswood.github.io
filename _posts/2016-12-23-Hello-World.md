---
layout: post
title: My first blog post! Why Jekyll? 
---

Ok, not really as I tried google blogger, and wordpress before, but didn't like them much.
This is however my first blog post using Jekyll - a static site generator. 

As a proagmatic software engineer, I usually have good reasons behind the choices I make (and I dislike the faddish nature of the software industry where something is all hyped up one month, and another product the next month).

## So why Jekyll?


Jekyll is a static site generator - the opposite of what I usually make - database driven applications. Static sites were popular in the early days of the web but gradually got overtaken by database backed sites and applicatioins of a more dynamic nature. 
Static sites do have a number of advantages however. 

### No database or other "moving parts" on the server. 
This gives a number of advantages

#### simple
 - less to set up and configure and keep maintained, so simpler overall.

#### secure
 - less security risks - with less moving parts there is a smaller [attack surface](https://en.wikipedia.org/wiki/Attack_surface) for hackers to potentially exploit. [SQL injection](https://en.wikipedia.org/wiki/SQL_injection) is an example of something that application developers need to protect against. With no database this is one less problem to worry about. For me this is the main appeal of a static site generator.


### It works with github pages - which offeres free hosting for Jekyll sites. 

Can't complian about that.

### Fast

Database backed sites can be badly written or configured which leads them to run slowly. To be fair a well written application should load quickly but it takes time and effort to tune more complex sites so that they do. 
As this is only serving static files, there is no need for database calls (i.o. is often the bottleneck in we applications) and there is a lot less computing going on in the background. 
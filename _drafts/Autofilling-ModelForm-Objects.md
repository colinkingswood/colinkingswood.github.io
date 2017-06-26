---
layout: post
title: Django CreateView with automatically filled fields.
---

I have a model with *required* fields and I want to use it with Django's generic CreateView.

In my exampe I am creating a todo list where a task object belongs to a logged in user. 
However I have two fields that I want filled automatically - the user should be the logged in user, and the created dat should be the time when the form is saved. 


I use class based views when possible, as it cuts down on a lot of the code needed, they are well writen and have probably covered more edge cases than I have considered. 

So in this case I am using a *CreateView* 

So with optional fields this is relatively easy. Let the form do its things, add the auto-filled fields after and save a second time. 

But that isn't going to work for the use case above, as when form tries to ssave the model it will complai nthat the required fields aren't available.

The ModelForm we have created does not know about the user or the request. 



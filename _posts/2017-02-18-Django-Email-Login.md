---
layout: post
title: Django login using email 
---

Using an email adress rather than username for logging in to a Django application. 

## Django auth login views

Building the new Matchstaff jobsite for European Bartender School, I wanted to be able to login users using their email address (rather than their username which is the Django default way).

Django already provides views for logging in and out, so rather than reinvent the wheel we use those. Those use the templates from the Django admin by default, but we want to use the custom templates from our designer. This is easy enough, by passing the template_name argument to the view.  


in urls.py we add the following lines for login and logout views 
    
https://en.wikipedia.org/wiki/Content_delivery_networkfrom django.contrib import admin, auth

urlpatterns = [
    ...
    ...
        
    url(r'^login/$',  auth.views.login,    {'template_name': 'jobsite/login.html'} , name='login'),
    url(r'^logout/$', auth.views.logout ,  {'template_name': 'jobsite/logout.html'}, name='logout'),

    ..
]
</code>
</pre>



I created a file backends.py and added the followin to it. 
This is where Django hooks in and uses our new class / function to check the email for logging in.

ModelBackend is the default Django authentication backend, so we subclass that, and override the authenticate method.

<pre>
<code class="language-python">
from django.contrib.auth.hashers import check_password
from django.contrib.auth import get_user_model
from django.contrib.auth.backends import ModelBackend


class UserModelEmailBackend(ModelBackend):

    def authenticate(self, username="", password="", **kwargs):

        try:
            user = get_user_model().objects.get(email__iexact=username)
            if check_password(password, user.password):
                return user
            else:
                return None

        except get_user_model().DoesNotExist:
            # No user was found, return None - triggers default login failed
            return None
</code>
</pre>

In addition to that we need to tell Django to use the new UserModelEmailBackend that we have created. 

In settings.py I added the following:

<pre>
<code class="language-python">
AUTHENTICATION_BACKENDS =['jobsite.backends.UserModelEmailBackend'  ,
                          'django.contrib.auth.backends.ModelBackend']
</code>  
</pre>


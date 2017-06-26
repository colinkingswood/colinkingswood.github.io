---
layout: post
title: Ansible, git setup for private repos.  
---

I wanted to set up Ansible for deployment of our new website (we currently have some Capistrano scripts but I don't know the Ruby eccosystem well, so I preffered a python based system). 
We have the code hosted in a private github repo, but most of the documentation for the git module is relevant to public repositories.  

I wanted to 

  * Create a directory owned by our deploy user
  * Clone the private github repository into that directory (as the deploy user)

First thing that I needed to do was add my ssh id_rsa.pub to github (the ones on my local machine, where I am running Ansible). You can see [instructions on github](https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/) 
  
Most of the Stack Overflow and other answers I found on the internet told me to add the following lines to my ansible.cfg file or variations of this:

<pre>
<code>
[ssh_connection]
ssh_args = -o ForwardAgent=yes
</code>
</pre>


This worked in that I could cone the repository as the root user (that hadn't been working before I added that). 

I wanted to use the deploy user that I had set up. 
so I added the following:

<pre>
<code>
become: true
become_user: deploy
<pre>
<code>

Unfortunately that didn't work, as changing users drops the key forwarding.






a
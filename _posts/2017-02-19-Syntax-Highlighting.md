---
layout: post
title: Syntax Highlightighting using Prism 
---

Writin a blog about coding, its useful to be able to display code in a readable way. 

With HTML we can use the <code>&lt;pre&gt;</code> tag for preformatting things, but it doesn't do syntax highlighting. I had a look around at other code blogs did a few searches and found [Prism](http://prismjs.com/). 

You need to download a css file and a JavaScript library. I would have preferred pure css, but I am not even sure if that is possible without adding a lot of classes to the HTML. It took me less than 20 minutes to get it working, so that's pretty good in terms of ease of use. 

### Download Prism libraries. 

This was about the only part that caused me a problem. The original tutorial I was looking at had Prism example using HTML, and CSS and was served from a [CDN](https://en.wikipedia.org/wiki/Content_delivery_network). 
I copied the CDN location and added it to my own HTML, and it formatted the code nicely, but didn't do the syntax highlighting that I was looking for.  

Then looking at the Prism website, there are a large number of languages with checkboxes beside them. The python checbox wasn't ticked. 

![Prism download page]({{ site.url }}/images/prism-download.png){: .center-image }


So it turns out that you need to specify the languages you want to use and download a custom copy of Prsim (unless yoo are using the default languages). It also gives you an option to choose a different theme, so I went for a dark background. 

Anyway, I added Python and a couple of other languages and downloaded my custom copy.

### Update  your HTML

I added the Prism js and css files to the head of my HTML. 
 
<pre>
<code class="language-html">
&lt;link rel=&quot;stylesheet&quot; href=&quot;{{ site.baseurl }}/css/prism.css&quot;&gt;
&lt;script src=&quot;{{ site.baseurl }}/js/prism.js&quot;&gt;&lt;/script&gt;
</code>
</pre>


Then its a case of wrapping the code in two tags, pre and code. The class on the code tag should be whatever language yoyu need to highlight - in my case python. 

<pre>
<code class="language-html">
&lt;pre&gt;
&lt;code class=&quot;language-python&quot;&gt;
from django.contrib.auth.hashers import check_password
..
.
&lt;/code&gt;
&lt;/pre&gt;

</code>
</pre>

## escaping HTML

One thing worth noting (as I discovered writing this post) is that code for HTML (and I assume for JS and CSS) needs to be escaped, otherwise the broswer will try to render it. 

I used this [site](http://www.freeformatter.com/html-escape.html#ad-output), cutting and pasting the small piece of example code that you see above to get this, which went into the file for creating this blog post. 
<pre>
<code class="language-html">
&amp;lt;pre&amp;gt;
&amp;lt;code class=&amp;quot;language-python&amp;quot;&amp;gt;
from django.contrib.auth.hashers import check_password
..
.
&amp;lt;/code&amp;gt;
&amp;lt;/pre&amp;gt;
</code>
</pre>


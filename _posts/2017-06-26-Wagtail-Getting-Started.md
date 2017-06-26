---
layout: post
title: Getting started with Wagtail (for Django developers)  
---

I have recently set up [Wagtail](https://wagtail.io/) for a simple blog. Wagtail is bulit on top of Django and uses many of the same features, but there are a few differences in the way you use it, so I thought it was worth writing it down so other devs familiar with Django can get up to speed a bit quicker than I did. All the information is in the [documentaion](http://docs.wagtail.io), but as there is quite a lot of it, hopefully this will save you some time.  


## Pages are represented by Django models. 

The first and probably main concept that you need to know is that you have page types which are represented with PageModels in models.py. These are pretty similar to Django models, but inherit from the PageModel class rather than Django's Model class.  Each page has a set of fields which are standard Django model fields e.g. IntegerField, with a couple extra additions - RichTextField and StreamField for adding more complicated stuff than text and numbers. 

In my example I have two types of page (at present). BlogIndexPage and BlogArticlePage. As the names suggest, the BlogArticle page is for every blog article and the index displays a list of these. 
Here is my BlogArticlePage example:

```python
class BlogArticlePage(Page):
    hero_image      = ImageField(null=True, upload_to="blog_post_heros")
    preview_image   = ImageField(null=True, upload_to="blog_post_previews" , help_text="Preview of hero image for blog index")
    preview_text    = models.TextField(null=True, blank=True ,               help_text="Text that will appear on the blog index (first 200 chars of body will be taken otherwise)")
    text_1          = RichTextField(blank=True ,                             help_text="Text that goes above the main image")
    main_image      = ImageField(null=True, upload_to='blog_post_images',    help_text="Image in the middle of the blog post")
    text_2          = RichTextField(blank=True  ,                            help_text="Text below the main image")
    author_desc     = RichTextField(blank=True, null=True ,                  help_text="Description of the blog post author (optional)"   )
    tags = ClusterTaggableManager(through=BlogPageTag, blank=True)
    
    content_panels = Page.content_panels + [
        FieldPanel('hero_image'),
        FieldPanel('preview_image'),
        FieldPanel('preview_text'),
        FieldPanel('text_1'),
        FieldPanel('main_image'),
        FieldPanel('text_2'),
        FieldPanel('author_desc'),
        FieldPanel('tags'),
    ]
```

The top half (above "content_panels") are all the fields - these get added as database fields when you run a migration. 

The content panels section is configuring the wagtail admin pages. 

![Wagtail admin screenshot]({{ site.url }}/images/wagtail-admin.png)

So if you wanted to display the hero_image at the bottom of the admin page, you would have that as the last entry in the content_panels list rather than the first. 



## Django view functions are done in models. 

I am used to using Django's class based generic views for most stuff these days. It takes a lot of boilerplate out of the coding. 

I have managed to get a [blog](http://www.ebsmatchstaff.com/blog) up and running  and I didn't touch views.py. So far all the functionality that is usually in views.py is now moved to the models in Models.py

For example my BlogIndex model doesn't really contain any data of its own (technically it does, but nothing that is relevant to this example). The idea is that it takes the most recent blog posts and displays a snippet of text and image from those.  In a normal Django application we would probably be using a [ListView](http://ccbv.co.uk/projects/Django/1.11/django.views.generic.list/ListView/) for this. In ```get_queryset``` we would probably add something like 

```python
return BlogArticlePage.objects.all()[0:10]
```

But so far I haven't used views.py and everything is handled for us automatically. So how do we get a list of blog articles to send to the template? 

We add a method ```get_context()``` to the PageModel and the queryset in there, which appears to be the equivalent of ```get_context_data()``` in Django's class based views. 

```python
 def get_context(self, request):
        context = super(BlogIndexPage, self).get_context(request)
        context['posts'] = BlogArticlePage.objects.descendant_of(self).live().filter(preview_image__isnull=False).order_by('-first_published_at')
        return context
```


## Templates

I haven't found out how to specify templates (yet). 

The default wagtail way of doing things is to use a template based on the name of the PageModel but with camel case converted to snake case. 

This is what my directory structure looks like, "blog" is Django app (jobsite is another Django app - all standard Django). 
```
├── blog
│   ├── migrations
│   └── templatetags
├── jobsite
│   ├── fixtures
│   ├── management
│   ├── migrations
│   ├── rest
│   └── templatetags
├── matchstaff
│   ├── settings
│   ├── static
│   └── templates
│       ├── blog
│       └── jobsite
├── media
```

In my models.py I have two main pages for the blog - BlogIndexPage and BlogArticlePage.

So in the ```matchstaff/templates``` directory, I have a folder blog, with the tenplates in there. 
Based on the model names BlogIndexPage and BlogArticlePage, I have the templates in there named ```blog_index_page.html``` and ```blog_article_page```. 

 
## URLs

Like the views.py file you don't really need to touch the urls.py file. I have a blog up and running without needing to touch anything in urls.py except for adding the wagtail urls as an include in the main urls.py 

```python
url(r'^blog/', include(wagtail_urls)),
```

When editing a page, there is a tab "Promote Page", which has a slug field that will be used for the url. It uses the tree structure (described below), so we would have blog/blog-article-slug for our blog posts. 


![Wagtail promote page screenshot]({{ site.url }}/images/promote-tab.png)
 

Now not using urls.py leads to some new problems. I added tagging to the blog articles so admins can add tags based on subjects and users can filter the list of blog articles on these tags. 
 
In this case we use a [RoutablePageMixin](http://docs.wagtail.io/en/latest/reference/contrib/routablepage.html). 


So we add in the tags model in models.py
 
```python

class BlogPageTag(TaggedItemBase):
    content_object = ParentalKey('BlogArticlePage', related_name='tagged_items')
```


You will see that the BlogArticlePage above already had this listed in the content panels - this gives us a nice autocomplete in the BlogArticlePage editor:


![Wagtail promote page screenshot]({{ site.url }}/images/tag-autocomplete.png)


Now we want to filter the blog index on these tags. 


```python
class BlogIndexPage(RoutablePageMixin, Page):

    .. model definition here ..

    @route('^tags/$', name='tag_archive')
    @route('^tags/([\-\w]+)/$', name='tag_archive')
    def tag_archive(self, request, tag=None):

        try:
            tag = Tag.objects.get(slug=tag)
        except Tag.DoesNotExist:
            if tag:
                msg = 'There are no blog posts tagged with "{}"'.format(tag)
                messages.add_message(request, messages.INFO, msg)
            return redirect(self.url)

        posts = self.get_posts(tag=tag)
        context = {
            'tag': tag,
            'posts': posts
        }
        return render(request, 'blog/blog_index_page.html', context)
```

So the blog index is usually reached at [http://ebsmatchstaff.com/blog/](http://ebsmatchstaff.com/blog/). Now if we want to filter by tags, we add new routes to the url [http://www.ebsmatchstaff.com/blog/tags/health/](http://www.ebsmatchstaff.com/blog/tags/health/). This will then use the tag archive function to return the blog posts filtered by the 'health' tag. 


## Tree Structure

This is the part that confused me the most, (especially as I had messed up my install and deleted the two default pages from the database).

All Wagtail pages pages are part of a tree structure. 

When installing wagtail, it adds two default pages - "root" and a "welcome to wagtail" page. Everything is added as a child page of one of these. Wagtail doesn't seem offer an option to add child pages to root, so all new pages end up being a child of "welcome to wagtail" (you can move the page to be a child of root afterwards, but you can't add it there directly). 

But I don't want "welcome to wagtail" as my top level page. That is fine. 

You need to set up a "site" to get everything up and running, and here you can specify your top level page for the site. So everything is in a tree structure, but your default page can be as far down the heirarchy as you want it. 

![Wagtail admin sites screenshot]({{ site.url }}/images/wagtail-sites.png)



So thats what I have learned getting a blog up and running in Wagtaill. Looking at the docs, I only seemed to have scratched the surface and it seems like a decent CMS system.

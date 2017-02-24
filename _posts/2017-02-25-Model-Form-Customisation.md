---
layout: post
title: Django ModelForm customization - an example with Class Based Views.
---

Django model forms are great and easy to use if you are using them in the standard way - i.e. a one to one mapping between fields in the form and fields in the model, not doing anything unusual with them. When you want a ModelForm to do something a bit different it gets a little trickier, but it is possible to do a fair bit of cusomization and still get the benefits of using Class Based Views with ModelForms. 

I needed to do a fair bit of customization recently, so I thought it would be a good idea to write it up as an example. 

It's worth pointing out that if you are working with Django's class based views, then the [Classy Class-Based Views](http://ccbv.co.uk/) site is great in helping to understanding them.


## Class based generic views, the basics

Using ModelForms with class based views you can generate a form with very little code on the Django side. 

<pre>
<code class="language-python">
class MyModelCreateView(CreateView):
    model = MyModel 
    template_name = 'my_template.html'
</code>  
</pre>

This will take the model you supplied and generate a model form from it, and do all the validation and saving when you submit the form. 

You still need to write the template (Django provides shortcuts for this `as_p`, and `as_table`). You need to add the   `<form>` elements but thats enough code to get a working form for creating a model in your Django site. 

<pre>
<code class="language-html">
    &lt;form action=&quot;&quot; method=&quot;post&quot; &gt;
        {% raw %}{% csrf_token %}
        {{ form.as_p }}{% endraw%}
        &lt;input type=&quot;submit&quot; class=&quot;button blue&quot; value=&quot;Send&quot;&gt;
    &lt;/form&gt;
</code>  
</pre>



## Customization

When you want to do something that doesn't fit the standard way of using it, then it gets a bit trickier, but it's possible to do quite a lot of customization. 

Here are some things that I wanted to customize on my form. 

  * use a different widget from the dafult for that field type.
  * add some extra classes to the form elements for styling.
  * Add a foreign key object (VenueContact) based on the logged in user.
  * Give a *required* field a default value. 
  * Display two fields location and address (not on the Venue model).
  * create an Address object based on the above fileds. 

I have been building a new JobSite for European Bartender School to relpace the old one. We want to take the data from the previous site and populate the new site with that data, which is fine, except the data in the old version isn't particularly clean. 

So in my models I have a Venue model with a related Address object. Address is also used by the jobseekers, which is why it got it's own table (complete with lots of nulls to deal with the data from the old site). 

       graduate >--- address ---< venue >---< venue_contacts 


Here's what my models looks like (well a cut down version)

<pre>
<code class="language-python">
class Address(models.Model):
    address1        = models.CharField(max_length=255, null=True,verbose_name='Address 1')
    address2        = models.CharField(max_length=255, null=True, blank=True,verbose_name='Address 2')
    world_city      = models.ForeignKey('WorldCity',   null=True, blank=True)
    city            = models.CharField(max_length=255, null=True,verbose_name='City')
    zipcode         = models.CharField(max_length=50,  null=True,verbose_name='Postcode')
    country         = models.CharField(max_length=100, null=True)


class Venue(models.Model):
    name    = models.CharField(max_length=125) 
    address = models.OneToOneField(Address)
    ...
    ..
    .
class VenueContact(models.Model):
    auth_user   = models.OneToOneField(AuthUser)     # name, username, email stored here
    phonenumber = models.CharField(max_length=20, validators=[phone_regex], blank=True , null=True)  # validators should be a list
    
class Graduate(models.Model):

    auth_user       = models.OneToOneField(AuthUser)
    address         = models.OneToOneField(Address)
    phonenumber     = models.CharField(max_length=20, validators=[phone_regex], blank=True, null=True)  #
    ...
    ..
    .
</code>  
</pre>


## Adding extra fields, and customizing widgets. 

Lets start with the easiest one, adding some classs for css styling and a different widget. I have a URL field, but the validation in the browser expected the url to have "http" at the start. That is likely to cause problems for the users, and we are doing server side validation anyway, so I changed it to a text widget. 
In addition to that I need some exta classes for css styling on some widgets. 


So the first thing to do is to create a custom from class, rather than using the auto generated form from class based views. 
It is required that you give the ModelForm a model and  either a `fields` or `excludes` list so that it knows which fields to use (or exclude). 
In forms.py


<pre>
<code class="language-python">
class VenueForm(forms.ModelForm):
    class Meta:
            model = Venue
            fields = [ 'name', 'address', 'email', 'mobile'  , 'venuetypes',  'logo' ,'short_description', 'long_description', 'website' ]
</code>
</pre>


To add css classes you need to give the widget an attrs dictionary. You can do this in a few ways. The simplest is by adding a widgets dictionary to the Meta class.

<pre>
<code class="language-python">
{% raw %}class VenueForm(forms.ModelForm):
    class Meta:
            model = Venue
            fields = [ 'name', 'address', 'email', 'mobile'  , 'venuetypes',  'logo' ,'short_description', 'long_description', 'website' ]
            widgets = { 'website' : forms.Textwidget(), 
                        'venuetypes' : forms.Select(queryset=Venuetypes.objects.all, 
                                                    attrs={'class' : 'venue_type_select'}
                                                    )
                       } 
            }    
{% endraw %}
</code>
</pre>

The widgets dictionary allows you to map fileds to the widgets that should be used. I have chaned the URL field to use a standard text widget - so no browser validation.

Then the venuetypes filed needs the `venue-type-select` so that it works with the JavaScript on that page. In this case we have to specify the same Select widget and querset that would have been generated beforee, but we add the attrs dictionary. 


## Overriding the __init__method

The other place that you can specify the widgets is in the \__init__ method. The first line of our \__init__ calls the parent classes method, and that actually builds the form for us. Then we can specify the widgets or attributes using `self.fields['field_name']`. 
The code below has the same effect as the previous example for the `venuetypes` and `website` fields. In addition i have added two extra fields that aren't part of the Venue model. These are for the adress object that we will save later. 

<pre>
<code class="language-python">
{% raw %}class VenueForm(forms.ModelForm):

    def __init__(self, *args, **kwargs):
        super(VenueForm, self).__init__(*args, **kwargs)
        self.fields['venuetypes'].widget.attrs = {'class': 'venue-type-select',}
        self.fields['website'].widget = forms.TextInput()


        self.fields['address'] = forms.CharField(required=True)
        self.fields['location'] = LocationChoiceField(queryset=WorldCity.objects.all(),
                                                      widget=forms.Select(attrs={'class': 'location-select'}),
                                                 )

    class Meta:
            model = Venue
            fields = [ 'name', 'address', 'email', 'mobile'  , 'venuetypes',  'logo' ,'short_description', 'long_description', 'website' ]

{% endraw %}
</code>
</pre>


## Adding a data based on the logged in user

In my data model every venue has a contact, which is linked to a Django auth user. Django knows who the user is so I shouldn't need to put this in the form (a hidden field would have been one option, but there is always the possibilty someone can tamper with the data being posted). 
 
The form is responsible for saving the object, so the functionality belongs in the form. But the form does not have access to the logged in user (or the request). 

The way to get around this is to pass the user to the form when it is initialised. In our view, we can use the `get_form_kwargs` method to pass in the user. 

In my `CreateView`
<pre>
<code class="language-python">
def get_form_kwargs(self):
    kwargs = {'user' : self.request.user , }
    return kwargs
</code>
</pre>

If you try running this now you will get an error: 
`__init__() got an unexpected keyword argument 'user'`

The way to fix this, and to make use of the user is in the form's `__init__` method again. We pop the user from kwargs *before* calling the parent classes `__init__`. We now have a form that has a **user** attribute.

<pre>
<code class="language-python">
class VenueForm(forms.ModelForm):

    def __init__(self, *args, **kwargs):
        user = kwargs.pop('user')
        self.logged_user = user
        super(VenueForm, self).__init__(*args, **kwargs)
         .... continue with form customization below....   
</code>
</pre>

We need to do two things in our form with the user. 1) Give the venue the user's email address as a default if the POST data doesn't contain the email address. 2) Create a VenueContact based on the user. 

To use the logged user's email as a default, use the clean_email method (each form field gets it's own clean method).
<pre>
<code class="language-python">
    def clean_email(self):
        email =self.cleaned_data['email']
        if not email:
            email = self.logged_user.email
        return email
</code>
</pre>

Creating a logged user is a bit more complicated as it relies on two fields, address and location (location will contain City names). this can be done in the view, after `form_valid`, save the form, add the extra fields, then save the object again. This wouldn't work in my case as the `VenueContact` is a required field, so the form.save would give an error. 

The place to do this is in the form's `clean()` method, as the docs say: 

*This method does any cleaning that is specific to that particular attribute, unrelated to the type of field that it is.*

So in our form again, add the clean method

<pre>
<code class="language-python">
    def clean(self):
        address_str = self.cleaned_data.pop('address')
        location = self.cleaned_data.pop('location')
        address = Address.objects.create(address1=address_str,
                               world_city=location,
                               city=location.city,
                               country=location.country)
        logged_user = self.logged_user
        contact, created = VenueContact.objects.get_or_create(auth_user=logged_user)
        self.cleaned_data.update({'address' : address, 'contact' : contact})
</code>
</pre>

So now when the form is validated, it gets / creates the extra objects that are not part of the model, and adds them to the cleaned data dictionary. 

Here is the final form:
<pre>
<code class="language-python">
class VenueForm(forms.ModelForm):

    def __init__(self, *args, **kwargs):
        user = kwargs.pop('user')
        self.logged_user = user
        super(VenueForm, self).__init__(*args, **kwargs)
        self.fields['location'] = LocationChoiceField(queryset=WorldCity.objects.all(),
                                                      widget=forms.Select(attrs={'class': 'location-select'}),
                                                 )

        self.fields['address'] = forms.CharField(required=True)
        self.fields['name'].widget.attrs ={'class': 'characters-remaining' ,
                                           'maxlength' :55 ,
                                           }
        self.fields['short_description'].widget = forms.Textarea(attrs = {'class': 'characters-remaining',
                                                                           'maxlength' : 155,
                                                                           'rows' : 4
                                                                            })
        self.fields['venuetypes'].widget.attrs = {'class': 'venue-type-select',}
        self.fields['website'].widget = forms.TextInput()

    class Meta:
            model = Venue
            fields = [ 'name', 'address', 'email', 'mobile'  , 'venuetypes',  'logo' ,'short_description', 'long_description', 'website' ]

    def clean_email(self):
        email =self.cleaned_data['email']
        if not email:
            email = self.logged_user.email
        return email

    def clean(self):
        address_str = self.cleaned_data.pop('address')
        location = self.cleaned_data.pop('location')
        address = Address.objects.create(address1=address_str,
                               world_city=location,
                               city=location.city,
                               country=location.country)
        logged_user = self.logged_user
        contact, created = VenueContact.objects.get_or_create(auth_user=logged_user)
        self.cleaned_data.update({'address' : address, 'contact' : contact})
</code>
</pre>

and the final view
<pre>
<code class="language-python">
{%raw%}@method_decorator(loggedin_decorators, name='dispatch')
class VenueCreateView(NamedFormsetsMixin, CreateWithInlinesView):
    model = Venue
    form_class = VenueForm
    template_name='jobsite/venue_logged.html'

    def get_success_url(self):
        return reverse('venue_update' , kwargs={ 'pk' : self.object.id})

    def get_form_kwargs(self):
        kwargs = super(VenueCreateView, self).get_form_kwargs()
        kwargs.update({'user' : self.request.user})
        return kwargs{% endraw %}
</code>
</pre>


(If the fields in Address have been confusing you , it's probably  because I have a `city` and `country` field for legacy data, and I want to replace this with a foreign key to the `WorldCity` table, which is prepopulated with cities, countries and their locations. When I get a chance I want to ensure that all addresses have the WordCity foreign key, and eliminate the other two fields). 

The functionality is almost all contained in the form (only added the `get_form_kwargs` method to the view)- this made the update view very easy to write, as the same form worked for that as well. If the save functionality was split between form and view, the extra code in the Views would need repeated between Create and Update views.


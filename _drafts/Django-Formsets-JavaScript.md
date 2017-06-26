---
layout: post
title: Django Formsets and JavaScript.  
---

Two things that you might find useful in this blog post:

* How to use the logged in user with ModelForms for foreign key (the correct way). 
* How to use Django formsets with JavaScript from the Django admin 

Django is great up until a point then it leaves you on your own. That's pretty much how it works when it comes to JavaScript. Which makes it quite painful when using Formsets, which basically require some JavaScript ot be functional. 
Formsets are for adding extra objects to the many side of a one to many relation. For the jobsite I am building I want the user to be able to upload multiple images when they are on the edit page of a venue. 
 
The annoying thing is that the JavaScript has clearly been written, as it is avaliable  in the Django admin (admin inlines are done using formsets).
 
// Last time I basically pulled the admin js apart, took the bits I wanted and needed, tweaked them and put it back together. 
// 
// This time I wanted to try and leave the JS alone and get my templates right so that they would work with it (I notice that the admin inlines.js doesn't seem to have changed since the last time I used it despite there being a few Django versions released in that time - seems failry stable to me).
This will show you how to create templates that are compatible with Django admin's inlines.js. 

So here is how to do it. 

I like Django's class based generic views, but those don't cover formsets. There is a package [django-extra-views](https://github.com/AndrewIngram/django-extra-views) that addds the same functionality as the other views but for formsets. 

The relevant models in my application are Venue and VenueImage, which has a foreign key to Venue - so one venue can have multiple images associated with it. 

I already had extra-views installed. Next was to create an inline. This is the equivalent of creating a TabularInline, or StackedInline in the Django admin. The VenueImage model only has one field - the image upload field. Setting the model on InlineFormset will generate the formset based on that. 

<pre>
<code class='language-python'>
class VenueImageFormSet(InlineFormSet):
    model = VenueImage
    extra = 0
    max_num = 10
    prefix = 'image_set'
    can_delete = True
    fields = ['upload']
</code>
</pre>


Next I added a the views. I have both a CreateWithInlinesView And An UpdateWithInlines view (both from the extra-vies package). They are pretty similar, the difference being that you pass an id / slug field in the URL to an update view, while a create view you don't and a new object is created.  You see the create view has the Venue has the Venue as the main object and the VenueImageInline added for the associated image uploads. I have overridden the get_form_kargs method to pass the user to the form  - as I want to link the user to the venue when it is created. The user is already logged in, so use than rather than passing it back and forward as a hidden form element.   


<pre>
<code class='language-python'>
@method_decorator(loggedin_decorators, name='dispatch')
class VenueCreateView(NamedFormsetsMixin, CreateWithInlinesView):
    model = Venue
    form_class = VenueForm
    template_name= 'jobsite/venue_edit.html'
    inlines = [VenueImageFormSet]
    inlines_names = ['venue_image_inline']

    def get_success_url(self):
        return reverse('job_create' , kwargs={'venue_id' : self.object.id})

    def get_form_kwargs(self):
        kwargs = super(VenueCreateView, self).get_form_kwargs()
        kwargs.update({'user' : self.request.user})
        return kwargs

</code>
</pre>

So one problem I had is that the venue object is associated with a user, and it should only be the logged in user that is able to create or edit the venue object. 

There were three ways I could achive this
* The easiest way to do that with a model form would be to send it in the form as a hidden field, but there is always the possibility of someone tampering with the form before it is posted back to the server so I didn't like that approach.
* Another option would be to excluse the user from the ModelFrom, then use the CreateView to create the venue object, then once the objects is saved by the form, update the venue with the user it and save again. 
* Pass the user to the form, and use it there. The ModelForm is alreasdy responsibe for validating the data then saving the venue object, so it seems like the correct place to put the logic for adding the user.

The problem is that the form doesn't know about the user or the request, so we need to pass it in. For that we can use the get_form_kwargs() method in the CreateView and pass the user in the kwrgs. 
Doing that on its own will give an error as a ModelForm doesn'tr expect a user argument (when it has been excluded from the fields). 

We can override the __init_() method to cope with this. Pop the user argument as the first thing we do, add it to self, so we can use it later, then call the super method to create the form as it normal. Now that the user has been poped from kwargs, it won't give an error. 
Also we need to override the clean method to create the related object before the model is saved. That way the form is responsible for creating the object and teh logic isn't splt between forms and views. 

<pre>
<code class='language-python'>
class VenueForm(forms.ModelForm):
    def __init__(self, *args, **kwargs):
        user = kwargs.pop('user')
        self.logged_user = user
        super(VenueForm, self).__init__(*args, **kwargs)

    def clean(self):
        # we want to turn the location and address str into an address object,
        # and use the logged user into a venue contact
        logged_user = self.logged_user
        contact, created = VenueContact.objects.get_or_create(auth_user=logged_user)
        self.cleaned_data.update('contact' : contact})
</code>
</pre>

So now when we save the form, the logged user is aded as a VenueContact. 

*Now back to Formsets*
 
 So our views will create  the form and relted formset for a venue, and venue uploads. 
 
 So how to get the template to work with the admin templates. 
 
 
 
<div  id="image_set-group" class="row form-field field-upload-photos has-border js-inline-admin-formset" data-inline-type="stacked" data-inline-formset="{&quot;name&quot;: &quot;#{{ venue_image_inline.prefix }}&quot;, &quot;options&quot;: {&quot;addText&quot;: &quot;Add another Venue image&quot;, &quot;deleteText&quot;: &quot;Remove&quot;, &quot;prefix&quot;: &quot;{{ venue_image_inline.prefix }}&quot;}}" >

    <div class="small-12 medium-4 columns field-label">
        <span class="text-right middle">Venue photos</span>
    </div>

    <!--  Management form -->
    {{ venue_image_inline.non_form_errors }}
    {{venue_image_inline.management_form }}

    <div class="small-12 medium-8 with-select-2 columns inline-related">
        {%  for image_form in venue_image_inline %}

            <img src="{% if image_form.instance.upload %}{{ image_form.instance.upload.url }}{% endif %}" alt="photo-name" class="uploaded-photo">
            <label for="venue-photos-upload" class="button blue-empty"><i class="fa fa-camera"></i>  Upload</label>
            {{ image_form.id }}
            {{ image_form.upload.errors }}
            {{ image_form.upload }}
            clear {{ image_form.DELETE}}
            <br>
            <hr>
        {%  endfor %}

        <!-- empty form -->
        <div class="empty-form {{ venue_image_inline.prefix }}"  id="{{ venue_image_inline.prefix }}-empty" >
        {{ venue_image_inline.empty_form }}
        </div>
        <!-- empty form end -->
        <p class="help-text">Max 5. Recommended dimensions: 800px x 600px.</p>

    </div>
</div>
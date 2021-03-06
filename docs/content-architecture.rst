====================
Content Architecture
====================

Mezzanine primarily revolves around the models found in two packages,
``mezzanine.core`` and ``mezzanine.pages``. This section describes these
models and how to extend them to create your own custom content for a
Mezzanine site.

The ``Page`` Model
==================

The foundation of a Mezzanine site is the model
``mezzanine.pages.models.Page``. Each ``Page`` instance is stored in a
hierarchical tree to form the site's navigation, and an interface for
managing the structure of the navigation tree is provided in the admin
via ``mezzanine.pages.admin.PageAdmin``. All types of content inherit from
the ``Page`` model and Mezzanine provides a default content type via the
``mezzanine.pages.models.RichTextPage`` model which simply contains a WYSIWYG
editable field for managing content.

.. _creating-custom-content-types:

Creating Custom Content Types
=============================

In order to handle different types of pages that require more structured
content than provided by the ``RichTextPage`` model, you can simply create
your own models that inherit from ``Page``. For example if we wanted to have
pages that were photo galleries::

    from django.db import models
    from mezzanine.pages.models import Page

    # The members of Page will be inherited by the Gallery model, such as
    # title, slug, etc. In this example the Gallery model is essentially a
    # container for GalleryImage instances.
    class Gallery(Page):
        notes = models.TextField("Notes")

    class GalleryImage(models.Model):
        gallery = models.ForeignKey("Gallery")
        image = models.ImageField(upload_to="galleries")

You'll also need to create an admin class for the Gallery model that
inherits from ``mezzanine.pages.admin.PageAdmin``::

    from copy import deepcopy
    from django.contrib import admin
    from mezzanine.pages.admin import PageAdmin
    from models import Gallery, GalleryImage

    gallery_extra_fieldsets = ((None, {"fields": ("notes",)}),)

    class GalleryImageInline(admin.TabularInline):
        model = GalleryImage

    class GalleryAdmin(PageAdmin):
        inlines = (GalleryImageInline,)
        fieldsets = deepcopy(PageAdmin.fieldsets) + gallery_extra_fieldsets

    admin.site.register(Gallery, GalleryAdmin)

In this example we build ``GalleryAdmin.fieldsets`` by copying
``PageAdmin.fieldsets`` and appending to it. This is so we can make use
of the fieldsets already defined by ``PageAdmin`` which would be lost if
we were to define our own ``fieldsets`` from scratch. It's also worth noting
that if we weren't to define ``fieldsets``, the fields defined for our
``Gallery`` model would not be automatically shown in the admin, as would
normally be the case with Django. This is because ``fieldsets`` has already
been defined by ``PageAdmin.fieldsets``.

By using an admin class that inherits from ``PageAdmin`` the admin class
won't be listed in the admin index page, instead being made available as
a type of ``Page`` when creating new pages from the navigation tree.

Page Permissions
================

The navigation tree in the admin where pages are managed will take
into account any permissions defined using `Django's permission system
<http://docs.djangoproject.com/en/dev/topics/auth/#permissions>`_. For
example if a logged in user doesn't have permission to add new
instances of the ``Gallery`` model from our previous example, it won't
be listed in the types of pages that user can add when viewing the
navigation tree in the admin.

In conjunction with Django's permission system, the ``Page`` model also
implements the methods ``can_add``, ``can_change`` and ``can_delete``.
These methods provide a way for custom page types to implement their
own permissions by being overridden on subclasses of the ``Page`` model.

Each of these methods takes a single argument which is the current
request object. This provides the ability to define custom permission
methods with access to the current user as well.

.. note::

    The ``can_add`` permission in the context of an existing page has
    a different meaning than in the context of an overall model as is
    the case with Django's permission system. In the case of a page
    instance, ``can_add`` refers to the ability to add child pages.

For example, if our ``Gallery`` content type should only contain one
child page at most, and only be deletable when added as a child page
(unless you're a superuser), the following permission methodss could
be implemented::

    class Gallery(Page):
        notes = models.TextField("Notes")

        def can_add(self, request):
            return self.children.count() == 0

        def can_delete(self, request):
            return request.user.is_superuser or self.parent is not None

Displaying Custom Content Types
===============================

When creating models that inherit from the ``Page`` model, multi-table
inheritance is used under the hood. This means that when dealing with the
page object, an attribute is created from the subclass model's name. So
given a ``Page`` instance using the previous example, accessing the
``Gallery`` instance would be as follows::

    >>> Gallery.objects.create(title="My gallery")
    <Gallery: My gallery>
    >>> page = Page.objects.get(title="My gallery")
    >>> page.gallery
    <Gallery: My gallery>

And in a template::

    <h1>{{ page.gallery.title }}</h1>
    <p>{{ page.gallery.notes }}</p>
    {% for galleryimage in page.gallery.galleryimage_set.all %}
    <img src="{{ MEDIA_URL }}{{ galleryimage.image }}" />
    {% endfor %}

The ``Page`` model also contains the method ``Page.get_content_model`` for
retrieving the custom instance without knowing its type beforehand::

    >>> page.get_content_model()
    <Gallery: My gallery>

Page Templates
==============

The view function ``mezzanine.pages.views.page`` handles returning a
``Page`` instance to a template. By default the template ``pages/page.html``
is used, but if a custom template exists it will be used instead. The check
for a custom template will first check for a template with the same name as
the ``Page`` instance's slug, and if not then a template with a name derived
from the subclass model's name is checked for. So given the above example
the templates ``pages/my-gallery.html`` and ``pages/gallery.html`` would be
checked for respectively.

Page Processors
===============

So far we've covered how to create and display custom types of pages, but
what if we want to extend them further with more advanced features? For
example adding a form to the page and handling when a user submits the form.
This type of logic would typically go into a view function, but since every
``Page`` instance is handled via the view function
``mezzanine.pages.views.page`` we can't create our own views for pages.
Mezzanine solves this problem using *Page Processors*.

*Page Processors* are simply functions that can be associated to any custom
``Page`` models and are then called inside the
``mezzanine.pages.views.page`` view when viewing the associated ``Page``
instance. A Page Processor will always be passed two arguments - the request
and the ``Page`` instance, and can either return a dictionary that will be
added to the template context, or it can return any of Django's
``HttpResponse`` classes which will override the
``mezzanine.pages.views.page`` view entirely.

To associate a Page Processor to a custom ``Page`` model you must create the
function for it in a module called ``page_processors.py`` inside one of your
``INSTALLED_APPS`` and decorate it using the decorator
``mezzanine.pages.page_processors.processor_for``.

Continuing on from our gallery example, suppose we want to add an enquiry
form to each gallery page. Our ``page_processors.py`` module in the gallery
app would be as follows::

    from django import forms
    from django.http import HttpResponseRedirect
    from mezzanine.pages.page_processors import processor_for
    from models import Gallery

    class GalleryForm(forms.Form):
        name = forms.CharField()
        email = forms.EmailField()

    @processor_for(Gallery)
    def gallery_form(request, page):
        form = GalleryForm()
        if request.method == "POST":
            form = GalleryForm(request.POST)
            if form.is_valid():
                # Form processing goes here.
                redirect = request.path + "?submitted=true"
                return HttpResponseRedirect(redirect)
        return {"form": form}

The ``processor_for`` decorator can also be given a ``slug`` argument rather
than a Page subclass. In this case the Page Processor will be run when the
exact slug matches the page being viewed.

The ``Displayable`` Model
=========================

The abstract model ``mezzanine.core.models.Displayable`` and associated
manager ``mezzanine.core.managers.PublishedManager`` provide common features
for items that can be displayed on the site with their own URLs (also known
as slugs). Some of these features are:

  * Fields for title and WYSIWYG edited content.
  * Auto-generated slug from the title.
  * Draft/published status with the ability to preview drafts.
  * Pre-dated publishing.
  * Meta data.

Content models that do not inherit from the ``Page`` model described earlier
should inherit from the ``Displayable`` model if any of the above features
are required. An example of this can be found in the ``mezzanine.blog``
application, where ``BlogPost`` instances contain their own URLs and views
that fall outside of the regular URL/view structure of the ``Page`` model.

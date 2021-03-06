.. _usage:

=====
Usage
=====

This document covers the usage of django-user-accounts. It assumes you've
read :ref:`installation`.

django-user-accounts has very good default behavior when handling user
accounts. It has been designed to be customizable in many aspects. By default
this app will:

* enable username authentication
* provide default views and forms for sign up, log in, password reset and
  account management
* handle log out with POST
* require unique email addresses globally
* require email verification for performing password resets

The rest of this document will cover how you can tweak the default behavior
of django-user-accounts.


Customizing the sign up process
===============================

In many cases you need to tweak the sign up process to do some domain specific
tasks. Perhaps you need to create a profile for the new user or something else.
The built-in ``SignupView`` has hooks to enable just about any sort of
customization during sign up. Here's an example of a custom ``SignupView``
defined in your project::

    import account.views
    
    
    class SignupView(account.views.SignupView):
        
        def after_signup(self, user, form):
            self.create_profile(user, form)
            super(SignupView, self).after_signup(user, form)
        
        def create_profile(self, user, form):
            profile = user.get_profile()
            profile.some_attr = "some value"
            profile.save()

You can define your own form class to add fields to the sign up process::

    # forms.py
    
    from django import forms
    from django.forms.extras.widgets import SelectDateWidget
    
    import account.forms
    
    
    class SignupForm(account.forms.SignupForm):
        
        birthdate = forms.DateField(widget=SelectDateWidget(years=range(1910, 1991)))
    
    # views.py
    
    import account.views
    
    import myproject.forms
    
    
    class SignupView(account.views.SignupView):
       
       form_class = myproject.forms.SignupForm
       
       def after_signup(self, user, form):
           self.create_profile(user, form)
           super(SignupView, self).after_signup(user, form)
       
       def create_profile(self, user, form):
           profile = user.get_profile()
           profile.birthdate = form.cleaned_data["birthdate"]
           profile.save()

To hook this up for your project you need to override the URL for sign up::

    from django.conf.urls import patterns, include, url
    
    import myproject.views
    
    
    urlpatterns = patterns("",
        url(r"^account/signup/$", myproject.views.SignupView.as_view(), name="account_signup"),
        url(r"^account/", include("account.urls")),
    )

.. note::

    Make sure your ``url`` for ``/account/signup/`` comes *before* the
    ``include`` of ``account.urls``. Django will short-circuit on yours.

Using email address for authentication
======================================

django-user-accounts allows you to use email addresses for authentication
instead of usernames. You still have the option to continue using usernames
or get rid of them entirely.

To enable email authentication do the following:

1. check your settings for the following values::
   
       ACCOUNT_EMAIL_UNIQUE = True
       ACCOUNT_EMAIL_CONFIRMATION_REQUIRED = True
   
   .. note::
   
       If you need to change the value of ``ACCOUNT_EMAIL_UNIQUE`` make sure your
       database schema is modified to support a unique email column in
       ``account_emailaddress``.
   
   ``ACCOUNT_EMAIL_CONFIRMATION_REQUIRED`` is optional, but highly
   recommended to be ``True``.

2. define your own ``LoginView`` in your project::
   
       import account.forms
       import account.views
       
       
       class LoginView(account.views.LoginView):
           
           form_class = account.forms.LoginEmailForm

3. ensure ``"account.auth_backends.EmailAuthenticationBackend"`` is in ``AUTHENTICATION_BACKENDS``

If you want to get rid of username you'll need to do some extra work:

1. define your own ``SignupForm`` and ``SignupView`` in your project::
   
       # forms.py
       
       import account.forms
       
       
       class SignupForm(account.forms.SignupForm):
           
           def __init__(self, *args, **kwargs):
               super(SignupForm, self).__init__(*args, **kwargs)
               del self.fields["username"]
       
       # views.py
       
       import account.views
       
       import myproject.forms
       
       
       class SignupView(account.views.SignupView):
           
           form_class = myproject.forms.SignupForm
           
           def generate_username(self, form):
               # do something to generate a unique username (required by the
               # Django User model, unfortunately)
               # If we have unique email, use this construction for generate unique username.
               # username=self.cleaned_data['username'].split('@',1)[0].
               # simple <magic> ^_^
               username = "<magic>"
               return username

2. many places will rely on a username for a User instance.
   django-user-accounts provides a mechanism to add a level of indirection
   when representing the user in the user interface. Keep in mind not
   everything you include in your project will do what you expect when
   removing usernames entirely.
   
   Set ``ACCOUNT_USER_DISPLAY`` in settings to a callable suitable for your
   site::
   
       ACCOUNT_USER_DISPLAY = lambda user: user.email
   
   Your Python code can use ``user_display`` to handle user representation::
   
       from account.utils import user_display
       user_display(user)
   
   Your templates can use ``{% user_display request.user %}``::
   
       {% load account_tags %}
       {% user_display request.user %}


Allow non-unique email addresses
================================

If your site requires that you support non-unique email addresses globally
you can tweak the behavior to allow this.

Set ``ACCOUNT_EMAIL_UNIQUE`` to ``False``. If you have already setup the
tables for django-user-accounts you will need to migrate the
``account_emailaddress`` table::

   ALTER TABLE "account_emailaddress" ADD CONSTRAINT "account_emailaddress_user_id_email_key" UNIQUE ("user_id", "email");
   ALTER TABLE "account_emailaddress" DROP CONSTRAINT "account_emailaddress_email_key";

``ACCOUNT_EMAIL_UNIQUE = False`` will prevent duplicate email addresses per
user.


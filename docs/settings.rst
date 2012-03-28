.. _settings:



For the send email you must set some variable in settings.py

EMAIL_HOST = 'localhost'
EMAIL_PORT = 1025
EMAIL_HOST_USER = ''
EMAIL_HOST_PASSWORD = ''
EMAIL_USE_TLS = False

In the debug mode you make run debug mail server for get message from django-user-accounts:
python -m smtpd -n -c DebuggingServer localhost:1025

Where is EMAIL_HOST in settings.py is a localhost, EMAIL_PORT in settings.py is 1025.

How send email for signup user.
DEFAULT_FROM_EMAIL = 'mail@mail.ss' #example you must set her you mail

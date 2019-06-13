# Django Blacklist

Blacklist users and hosts in Django. Automatically blacklist rate-limited clients.


## Overview

Django Blacklist allows you to block specific users and IP addresses/networks from accessing your application.
Clients can be blocked manually from the admin interface, or automatically after exceeding a request rate limit.
The blacklist rules are applied for a specific duration.


## Installation

To install the package, run:
```
$ pip install django-blacklist
```

Add the `blacklist` application to `INSTALLED_APPS`:
```
INSTALLED_APPS = [
    ...
    'blacklist'
]
```

Add the `blacklist_middleware` middleware after `AuthenticationMiddleware`:
```
MIDDLEWARE = [
    ...
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'blacklist.middleware.blacklist_middleware',
    ...
]
```

Apply the blacklist database migrations:
```
$ python manage.py migrate blacklist
```


## Usage

You can manage the blacklist rules from the admin. Changes take effect after restarting the server.
A rule can target a user or an IP address.
You can also target IP networks (ranges) by specifying the optional prefixlen field (number of network prefix bits).
Each rule has a specific duration. After that duration passes, rules expire automatically, without a restart.
When a request is rejected due to a matching rule, an response with HTTP status 400 (bad request) is returned,
and an error is output from logger `django.security`.

### Removing Expired Rules

Expired rules are not automatically removed from the database.
They can be cleaned up with the included management command `trim_blacklist`:
```
$ python manage.py trim_blacklist [-c <created_days>] [-e <expired_days>]
```
The options `-c` and `-e` specify the minimum ages of creation and expiry, respectively.


## Automatic Blacklisting

Clients can be blacklisted automatically, after exceeding a specified request rate limit.
This feature requires [django-ratelimit](https://github.com/jsocol/django-ratelimit).

First, rate-limit a view by applying the `@ratelimit` decorator. Make sure to set `block=False`.
Then, blacklist rate-limited clients by adding the `@blacklist_ratelimited` decorator. Specify the blacklist duration.
For example:
```
from datetime import timedelta
from ratelimit.decorators import ratelimit
from blacklist.ratelimit import blacklist_ratelimited

@ratelimit(key='user_or_ip', rate='50/m', block=False)
@blacklist_ratelimited(timedelta(minutes=30))
def index(request):
    ...
```

Automatic rules take effect immediately, without a restart.
If the request comes from an authenticated user, the rule will target that user.
Otherwise, it will target their IP address.
***
Note: The client IP address is taken from the `REMOTE_ADDR` value of `request.META`.
If your application is behind one or more reverse proxies, this will, by default,
always be the address of the nearest proxy.
To avoid blacklisting all clients, you can set `REMOTE_ADDR` from the `X-Forwarded-For` header in middleware.
However, keep in mind that this header can be forged to bypass the rate limits.
To counter that, you can use the last address in that header.
If you are behind two proxies, use the second to last, etc.
***

`@blacklist_ratelimited` accepts two arguments: `(duration, block=True)`.
* `duration` can be a `timedelta` object, or a tuple of two separate durations
(for user-based and IP-based rules).
* `block` specifies if the request should be rejected immediately, or passed to the view.

Automatic rules will have a comment that contains the ID of the request, which triggered the creation of the rule,
and the "request line".
The request ID is added only if available. Django does not generate request IDs.
For that purpose, you can install [django-log-request-id](https://github.com/dabapps/django-log-request-id).

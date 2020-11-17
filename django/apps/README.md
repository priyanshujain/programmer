## Be careful about “applications”
Django has this concept of “applications,” which are vaguely described in the documentation as “a Python package that provides some set of features.” While they do make sense as reusable libraries that can be plugged into different projects, their utility as far as organizing your main application code is less clear.

There are some implications from how you define your “apps” that you should be aware of. The biggest one is that Django tracks model migrations separately for each app. If you have ForeignKey’s linking models across different apps, Django’s migration system will try to infer a dependency graph so that migrations are run in the right order. Unfortunately, this calculation isn’t perfect and can lead to some errors or even complex circular dependencies, especially if you have a lot of apps.

We originally organized our code into a bunch of separate “apps” to organize different functionality, but had a lot of cross-app ForeignKey’s. The migrations we had checked in would occasionally wind up in a state where they would run okay in production, but not in development. In the worst case, they would not even play back on top of a blank database. Each system may have a different permutation of migration states for different apps, and running manage.py migrate may not work with all of them. Ultimately, we found that having all these separate apps led to unnecessary complexity and headaches.

We quickly discovered that if we had these ForeignKey’s crossing different apps, then perhaps they weren’t really separate apps to begin with. In fact, we really just had one “app” that could be organized into different packages. To better reflect this, we trashed our migrations and migrated everything to a single “app.” This process wasn’t the easiest task to accomplish (Django also uses app names to populate ContentType’s and in naming database tables — more on that later) but we were happy we did it. It also meant that our migrations all had to be linearized, and while that came with downsides, we found that they were outweighed by the benefit of having a predictable, stable migration system.

To summarize, here are our suggestions for any developers starting a Django project:

If you don’t really understand the point of apps, ignore them and stick with a single app for your backend. You can still organize a growing codebase without using separate apps.
If you do want to create separate applications, you will want to be very intentional about how you define them. Be very explicit about and minimize any dependencies between different apps. (If you are planning to migrate to microservices down the line, I can imagine that “apps” might be a useful construct to define precursors to a future microservice).

## Organize your apps inside a package

While we’re on the topic of apps, let’s talk a little bit about package organization. If you follow Django’s “Getting started” tutorial, the manage.py startapp command will create an “app” at the top-level of the project directory. For instance, an app called foo would be accessible as import foo.models… . We would strongly advise you to actually put your apps (and any of your Python code) into a Python package, namely the package that is created with django-admin startproject.

In Django’s tutorial example, instead of:

```s
mysite/
    mysite/
        __init__.py
polls/
    __init__.py
We’d suggest:

mysite/
    mysite/
        __init__.py
    polls/
        __init__.py
```

This is a small and subtle change, but it prevents namespace conflicts between your app and third party Python libraries. In Python, top-level modules go into a global namespace and need to be uniquely named. As an example, the Python library for a vendor we used, Segment, is actually named analytics. If we had an analytics app defined as a top-level module, there would be no way to distinguish between the two packages in your code.





### Reference
https://doordash.engineering/2017/05/15/tips-for-building-high-quality-django-apps-at-scale/
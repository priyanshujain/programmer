### The SSL error: decryption failed or bad record mac

This occurs either when the certificate is invalid or the message hash value has been tampered; in our case it’s because of the latter. Django creates a single database connection when it tries to query for the first time. Any subsequent calls to the database will use this existing connection until it is expired or closed, in which it will automatically create a new one the next time you query. The PostgreSQL engine in Django uses psycopg to talk to the database; according to the document it is level 2 thread safe. Unfortunately, the timeout() method is using multiprocessing module and therefore tampers the SSL MAC. There are different ways to fix this. We can either (1) use basic threads instead of spawning a new process or (2) use a new database connection in the timeout() method. We can also (3) scrap the timeout() method altogether and handle the async task properly via Celery.
The timeout() method is simple, generic, and a quick implementation to limit the execution time of a method. We can use this for anything that takes time, not only for those that queries databases. I think most of the time spawning a new process is safer than using threads. At this point in time, I also don’t want to introduce and maintain another moving part like Celery yet; so we’re choosing to use a new database connection to fix the issue! Unfortunately, it seems it wouldn’t be as pretty. In order to create a new database connection, you just have to close the existing one and Django will create a new one. But this doesn’t work if you are inside a database transaction — it will fail InterfaceError: connection already closed. There’s also no easy way to pass a new database connection when using the models. The closest would be the .using() method where you specify a database alias. You can probably overload the cached property django.db.connections.databases and add your alias with new connection but I wouldn’t really do that! Haha! So we’re left with executing custom SQL directly:

A sample rawSQL query using a new database connection.
The issue we talked about could probably be detected earlier if I just use PostgreSQL, like in Production, instead of SQLite in memory when running the integration tests. I also could’ve tested it works properly in Dev environment where it uses proper PostgreSQL. Silly me! Overall, the debugging process have kept me busy in this quarantine period while working at home. I think it’s also a little fun and a good learning experience for becoming a better Software Engineer.
Keep safe folks! Let’s come back stronger after this pandemic.



References:

1. https://virtualandy.wordpress.com/2019/09/04/a-fix-for-operationalerror-psycopg2-operationalerror-ssl-error-decryption-failed-or-bad-record-mac/

2. https://medium.com/@philamersune/fixing-ssl-error-decryption-failed-or-bad-record-mac-d668e71a5409

3. https://www.psycopg.org/docs/usage.html#thread-safety

4. https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNECT

5. https://stackoverflow.com/questions/57933533/postgres-sqlalchemy-and-multiprocessing


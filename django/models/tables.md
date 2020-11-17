## Explicitly name your database tables
Your database is more important, more long-lived, and harder to change after the fact than your application. Knowing that, it makes sense that you should be very intentional about how you are designing your database schema, rather than allowing a web framework to make those decisions for you.

While you largely do control the database schema in Django, there are a few things it handles by default that you should know about. For example, Django automatically generates a table name for your models, with the pattern of `<app_name>_<model_name_lowercased>`. Instead of relying on these auto-generated names, you should consider defining your own naming convention and naming all of your tables manually, using Meta.db_table.

```py
class Foo(Model):
    class Meta:
        db_table = 'foo'
```

The other thing to watch for is ManyToManyFields. Django makes it easy to generate many-to-many-relationships using this field and will create the join table with automatically-generated table and column names. Instead of doing that, we strongly suggest you always create and name the join tables manually (using the through keyword). It’ll make it a lot easier to access the table directly, and frankly, we found that it’s just annoying to have hidden tables.

These may seem like minor details, but decoupling your database naming from Django implementation details is a good idea because there are going to be other things that touch your data besides the Django ORM, such as data warehouses. This also allows you to rename your model classes later on if you change your mind. Finally, it’ll simplify things like breaking out tables into separate services or transitioning to a different web framework.

## Avoid GenericForeignKey
If you can help it, avoid using GenericForeignKey’s. You lose database query features like joins (`select_related`) and data integrity features like foreign key constraints and cascaded deletes. Using separate tables is usually a better solution, and you can leverage abstract base models if you are looking for code-reuse.

That said, there are situations where it can still be helpful to have a table that can point to different tables. If so, you would be better off doing your own implementation and it isn’t that hard (you just need two columns, one for the object ID, the other to define the type). One think we dislike about GenericForeignKey is that they have a dependency on Django’s ContentTypes framework, which stores identifiers for tables in a mapping table named `django_contenttypes`.

That table is not a lot of fun to deal with. For starters, it uses the name of your app (app_label) and the Python model class (model) as columns to map a Django model to an integer id, which is then stored inside the table with the GFK. If you ever move models between apps or rename your apps, you’re going to have to do some manual patching on this table. More importantly, having a common table hold these GFK mapping will greatly complicate things if you ever want to move your tables into separate services and databases. Similar to the earlier section about explicitly naming your tables — you should own and define your own table identifiers as well. Whether you want to use an integer, string, or something else to do this, any of these is better than relying on an arbitrary ID defined in some random table.

## Keep migrations safe
If you are using Django 1.7 or later and are using a relational database, you are probably using Django’s migration system to manage and migrate your schema. As you start running at scale, there are some important nuances to consider about using Django migrations.

First of all, you will need to make sure that your migrations are safe, meaning they will not cause downtime when they are applied. Let’s suppose that your deployment process involves calling manage.py migrate automatically before deploying the latest code on your application servers. An operation like adding a new column will be safe. But it shouldn’t be too surprising that deleting a column will break things, as existing code would still be referencing the nonexistent column. Even if there are no lines of code that reference the deleted field, when Django fetches an object (e.g. Model.objects.get(..), under the hood it performs a SELECT on every column that is defined in the model. As a result, pretty much any Django ORM access to that table will raise an exception.

You could avoid this issue by being sure to run the migrations after the code is deployed, but it does mean deployments have to be a bit more manual. It can get tricky if developers have committed multiple migrations ahead of a deploy. Another workaround is to make these and other dangerous migrations into “no-op” migrations, by making migrations purely “state” operations. You will then need to perform the DROP operations after the deploy.

```py
class Migration(migrations.Migration):
    state_operations = [ORIGINAL_MIGRATIONS]
    operations = migrations.SeparateDatabaseAndState(
        state_operations=state_operations
    )

```
Of course, dropping columns and tables aren’t the only operation you will want to watch out for. If you have a large production database, there are many unsafe operations that may lock up your database or tables and lead to downtime. The specific types of operations will depend on what variant of SQL you are using. For example, with PostgreSQL, adding columns with an index or that are non-nullable to a large table can be dangerous. Here’s a pretty good article from BrainTree summarizing some of the dangerous migrations on PostgreSQL.

## Squash your migrations
As your project evolves and accumulates more and more migrations, they will take longer and longer to run. By design, Django needs to incrementally play back every single migration starting from the first one in order to construct its internal state of the database schema. Not only will this slow down production deploys, developers will also have to wait when they initially set up their local development database. If you have multiple databases, this process will take even longer, because Django will play all migrations on every single database, regardless of whether the migration affects that database.

Short of avoiding Django migrations entirely, the best solution we’ve come up with is to just do some periodic spring cleaning and to “squash” your migrations. One option is to try Django’s built-in squashing feature. Another option, which has worked well for us, is to just do this manually. Drop everything in the `django_migrations` table, delete existing migration files, and run manage.py makemigrations to create fresh, consolidated migrations.

```sh
python manage.py squashmigrations <appname> <squashfrom> <squashto>

```
#### Alternate 
- delete migration files from project directory
- remove migration entries from the django_migrations database table
- run makemigrations - Django creates new migration files
- run migrate --fake as you should already have the tables in the database



## Reduce migration friction
If many dozens of developers are working on the same Django codebase, you may frequently run into race conditions with merging in database migrations. For example, consider a scenario where the current migration history on master looks like:

```
0001_a
0002_b
```
Now suppose engineer A generates migration 0003_c on his local branch, but before he is able to merge it, engineer B gets there first and checks in migration 0003_d. If engineer A now merges his branch, anyone that tries to run migrations after pulling the latest code will run into the error “Conflicting migrations detected; multiple leaf nodes in the migration graph: (0003_c, 0003_d).”

At a minimum, this results in the migrations having to be manually linearized or creating a merge migration, causing friction in the team’s development process. The Zenefits engineering team discusses this problem in more detail in a blog post, from which we derived inspiration to improve upon this.

In less than a dozen lines of code, we were able to solve a more general form of this problem in the case where we have multiple Django applications. We did this by overriding the handle() method of our makemigrations command to generate a multiple-application migration manifest:

```py
def handle(self, *app_labels, **options):
  # Generate a migrations manifest with latest migration on each app
  super(Command, self).handle(*app_labels, **options)
 
  loader = MigrationLoader(None, ignore_no_migrations=True)
  apps = sorted(loader.migrated_apps)
  graph = loader.graph
  
  with open('latest_migrations.manifest', 'w') as f:
    for app_name in apps:
      leaf_nodes = graph.leaf_nodes(app_name)
      if len(leaf_nodes) != 1:
        raise Exception('App {} has multiple leaf migrations!'.format(app_name))  
      f.write('{}: {}\n'.format(app_name, leaf_nodes[0][1]))

```

Applying this to the above example, the manifest file would have one entry doordash: `0002_b` for our app. If we generate a new migration file `0003_c` off HEAD, the diff on the manifest file will apply cleanly and can be merged as is:

```
- doordash: 0002_b
+ doordash: 0003_c
```
However, if the migrations are outdated, such as if an engineer only has `0001_a` locally and generates a new migration `0002_d`, the manifest file diff will not apply cleanly and thus Github would declare that there are merge conflicts:

```
- doordash: 0001_a
+ doordash: 0002_d
```

The engineer would then be responsible for resolving the conflict before Github will allow the pull request to be merged. If you have integration tests on which code merges are gated (which any company of that size should), this is also another motivation to keep the test suite fast!




## Avoid Fat Models
Django’s promotes a “fat model” pattern where you put the bulk of your business logic inside model methods. While this is what we used initially, and it can even be pretty convenient, we realized that it does not scale very well. Over time, model classes become bloated with methods and get extremely long and difficult to read. Mixins are one way to mitigate the complexity a bit, but do not feel like an ideal solution.

This pattern can be kind of awkward if you have some logic that doesn’t really need to operate on a full model instance fetched from the database, but rather just needs the primary key or a simplified representation stored in the cache. Additionally, if you ever wish to move off the Django ORM, coupling your logic to models is going to complicate that effort.

That said, the real intention behind this pattern is to keep the API/view/controller lightweight and free of excessive logic, which is something we would strongly advocate. Having logic inside model methods is a lesser evil, but you may want to consider keeping models lightweight and focused on the data layer. To make this work, you will need to figure out a new pattern and put your business logic in some layer(service/usecase layer) that is in between the data layer and the API/presentational layer.

## Be careful with signals
Django’s signals framework can be useful to decouple events from actions, but one use case that can be troublesome are pre/post_save signals. They can be useful for small things (e.g. checking when to invalidate a cache) but putting too much logic in signals can make program flow difficult to trace and read. Passing custom arguments or information through a signal is not really possible. It is also very difficult, without the use of some hacks, to disable a signal from firing on certain conditions (e.g. if you want to bulk update some models without triggering expensive signals).

Our advice would be to limit your use of these signals, and if you do use them, avoid putting other than simple and cheap logic inside them. You should also keep these signals organized in a predictable and consistent place (e.g. close to where the models are defined), to make your code easy to read.





## Avoid using the ORM as the main interface to your data
If you are directly creating and updating database objects from many parts of your codebase with calls to the Django ORM interface `Model.objects.create()` or `Model.save())`, you may want to revisit this approach. We found that using the ORM as the primary interface to modify data has some drawbacks.

The main problem is that there isn’t a clean way to perform common actions when a model is created or updated. Suppose that every time ModelA is created, you really want to also create an instance of ModelB. Or you want to detect when a certain field has changed from its previous value. Apart from signals, your only workaround is to overload a lot of logic into Model.save(), which can get very unwieldy and awkward.

One solution for this is to establish a pattern in which you route all important database operations (create/update/delete) through some kind of simple interface that wraps the ORM layer(DBAPI). This gives you clean entry points to add additional logic before or after database events. Additionally, decoupling your application code a bit from the model interface will give you the flexibility to move off the Django ORM in the future.







## Don’t cache Django models
If you are working on scaling your application, you are probably taking advantage of a caching solution like Memcached or Redis to reduce database queries. While it can tempting to cache Django model instances, or even the results of entire Querysets, there are some caveats you should be aware of.

If you migrate your schema (add/change/delete fields from your model), Django actually does not handle this very gracefully when dealing with cached instances. If Django tries to read a model instance that was written to the cache from an earlier version of the schema, it will pretty much die. Under the hood, it’s deserializing a pickled object from the cache backend, but that object will be incompatible with the latest code. This is more of an unfortunate Django implementation detail than anything else.

You could just accept that you’ll have some exceptions after a deploy with a model migration, and limit the damage by setting reasonably short cache TTL’s. Better yet, avoid caching model’s altogether as a rule. Instead, only cache the primary keys, and look up the objects from the database. (Typically, primary key lookups are pretty cheap. It’s the SELECT queries to find those IDs that are expensive).

Taking this a step further to avoid database hits entirely, you can still cache Django models safely if you only maintain one cached copy of a model instance. Then, it is pretty trivial to invalidate that cache upon changes to the model schema. Our solution was to just create a unique hash of the known fields and adding that to our cache key (e.g. Foo:96f8148eb2b7:123). Whenever a field is added, renamed or deleted, the hash changes effectively invalidate the cache.

# Conclusion
Django is definitely a powerful and feature-filled framework for getting started on your backend service, but there are subtleties to watch out for that can save you headaches down the road. Defining Django apps carefully and implementing good code organization up front will help you avoid unnecessary refactoring work later. Meanwhile, by taking full control over your database schema and being deliberate about how you use Django features like GenericForeignKey’s and the ORM, you can ensure that you aren’t too coupled to the framework and and migrate to other technologies or architectures in the future.

By thinking about these things, you can maintain the flexibility to evolve and scale your backend in the future. We hope that some of the things we’ve learned about using Django will help you out in building your own apps!



### Reference
https://doordash.engineering/2017/05/15/tips-for-building-high-quality-django-apps-at-scale/
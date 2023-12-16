# Database transactions

Django gives you a few ways to control how database transactions are managed.

## Managing database transactions

### Django’s default transaction behavior

Django’s default behavior is to run in autocommit mode. Each query is immediately committed to the database, unless a transaction is active. See below for details.

Django uses transactions or savepoints automatically to guarantee the integrity of ORM operations that require multiple queries, especially delete() and update() queries.

Django’s `TestCase` class also wraps each test in a transaction for performance reasons.

---
### Tying transactions to HTTP requests

A common way to handle transactions on the web is to wrap each request in a transaction. Set `ATOMIC_REQUESTS` to `True` in the configuration of each database for which you want to enable this behavior.

It works like this. Before calling a view function, Django starts a transaction. If the response is produced without problems, Django commits the transaction. If the view produces an exception, Django rolls back the transaction.

You may perform subtransactions using savepoints in your view code, typically with the `atomic()` context manager. However, at the end of the view, either all or none of the changes will be committed.

> **Warning**
> 
> While the simplicity of this transaction model is appealing, it also makes it inefficient when traffic increases. Opening a transaction for every view has some overhead. The impact on performance depends on the query patterns of your application and on how well your database handles locking.

> **Per-request transactions and streaming responses**
> 
> When a view returns a `StreamingHttpResponse`, reading the contents of the response will often execute code to generate the content. Since the view has already returned, such code runs outside of the transaction.
> 
> Generally speaking, it isn’t advisable to write to the database while generating a streaming response, since there’s no sensible way to handle errors after starting to send the response.

In practice, this feature wraps every view function in the `atomic()` decorator described below.

Note that only the execution of your view is enclosed in the transactions. Middleware runs outside of the transaction, and so does the rendering of template responses.

When `ATOMIC_REQUESTS` is enabled, it’s still possible to prevent views from running in a transaction.

`non_atomic_requests`(using=None)
: This decorator will negate the effect of `ATOMIC_REQUESTS` for a given view:

```python
from django.db import transaction


@transaction.non_atomic_requests
def my_view(request):
    do_stuff()


@transaction.non_atomic_requests(using="other")
def my_other_view(request):
    do_stuff_on_the_other_database()
```

It only works if it’s applied to the view itself.

---
### Controlling transactions explicitly

Django provides a single API to control database transactions.

`atomic`(using=None, savepoint=True, durable=False)
: Atomicity is the defining property of database transactions. `atomic` allows us to create a block of code within which the atomicity on the database is guaranteed. If the block of code is successfully completed, the changes are committed to the database. If there is an exception, the changes are rolled back.

`atomic` blocks can be nested. In this case, when an inner block completes successfully, its effects can still be rolled back if an exception is raised in the outer block at a later point.

It is sometimes useful to ensure an `atomic` block is always the outermost `atomic` block, ensuring that any database changes are committed when the block is exited without errors. This is known as durability and can be achieved by setting `durable=True`. If the `atomic` block is nested within another it raises a `RuntimeError`.

`atomic` is usable both as a decorator:

```python
from django.db import transaction


@transaction.atomic
def viewfunc(request):
    # This code executes inside a transaction.
    do_stuff()
```

and as a context manager:

```python
from django.db import transaction


def viewfunc(request):
    # This code executes in autocommit mode (Django's default).
    do_stuff()

    with transaction.atomic():
        # This code executes inside a transaction.
        do_more_stuff()
```

Wrapping `atomic` in a try/except block allows for natural handling of integrity errors:

```python
from django.db import IntegrityError, transaction


@transaction.atomic
def viewfunc(request):
    create_parent()

    try:
        with transaction.atomic():
            generate_relationships()
    except IntegrityError:
        handle_exception()

    add_children()
```

In this example, even if `generate_relationships()` causes a database error by breaking an integrity constraint, you can execute queries in `add_children()`, and the changes from `create_parent()` are still there and bound to the same transaction. Note that any operations attempted in `generate_relationships()` will already have been rolled back safely when `handle_exception()` is called, so the exception handler can also operate on the database if necessary.

> **Avoid catching exceptions inside `atomic`!**
> 
> When exiting an `atomic` block, Django looks at whether it’s exited normally or with an exception to determine whether to commit or roll back. If you catch and handle exceptions inside an `atomic` block, you may hide from Django the fact that a problem has happened. This can result in unexpected behavior.
> 
> This is mostly a concern for `DatabaseError` and its subclasses such as `IntegrityError`. After such an error, the transaction is broken and Django will perform a rollback at the end of the `atomic` block. If you attempt to run database queries before the rollback happens, Django will raise a `TransactionManagementError`. You may also encounter this behavior when an ORM-related signal handler raises an exception.
>
>The correct way to catch database errors is around an `atomic` block as shown above. If necessary, add an extra `atomic` block for this purpose. This pattern has another advantage: it delimits explicitly which operations will be rolled back if an exception occurs.
>
>If you catch exceptions raised by raw SQL queries, Django’s behavior is unspecified and database-dependent.

> **You may need to manually revert app state when rolling back a transaction.**
> 
> The values of a model’s fields won’t be reverted when a transaction rollback happens. This could lead to an inconsistent model state unless you manually restore the original field values.
> 
> For example, given `MyModel` with an `active` field, this snippet ensures that the `if obj.active` check at the end uses the correct value if updating `active` to `True` fails in the transaction:
> 
> ```python
> from django.db import DatabaseError, transaction
> 
> obj = MyModel(active=False)
> obj.active = True
> try:
>     with transaction.atomic():
>         obj.save()
> except DatabaseError:
>     obj.active = False
> 
> if obj.active:
>     ...
> ```
> 
> This also applies to any other mechanism that may hold app state, such as caching or global variables. For example, if the code proactively updates data in the cache after saving an object, it’s recommended to use transaction.on_commit() instead, to defer cache alterations until the transaction is actually committed.

In order to guarantee atomicity, `atomic` disables some APIs. Attempting to commit, roll back, or change the autocommit state of the database connection within an `atomic` block will raise an exception.

`atomic` takes a `using` argument which should be the name of a database. If this argument isn’t provided, Django uses the `"default"` database.

Under the hood, Django’s transaction management code:

- opens a transaction when entering the outermost `atomic` block;
- creates a savepoint when entering an inner `atomic` block;
- releases or rolls back to the savepoint when exiting an inner block;
- commits or rolls back the transaction when exiting the outermost block.

You can disable the creation of savepoints for inner blocks by setting the `savepoint` argument to `False`. If an exception occurs, Django will perform the rollback when exiting the first parent block with a savepoint if there is one, and the outermost block otherwise. Atomicity is still guaranteed by the outer transaction. This option should only be used if the overhead of savepoints is noticeable. It has the drawback of breaking the error handling described above.

You may use `atomic` when autocommit is turned off. It will only use savepoints, even for the outermost block.

> **Performance considerations**
> 
> Open transactions have a performance cost for your database server. To minimize this overhead, keep your transactions as short as possible. This is especially important if you’re using `atomic()` in long-running processes, outside of Django’s request / response cycle.

---
## Autocommit

### Why Django uses autocommit

In the SQL standards, each SQL query starts a transaction, unless one is already active. Such transactions must then be explicitly committed or rolled back.

This isn’t always convenient for application developers. To alleviate this problem, most databases provide an autocommit mode. When autocommit is turned on and no transaction is active, each SQL query gets wrapped in its own transaction. In other words, not only does each such query start a transaction, but the transaction also gets automatically committed or rolled back, depending on whether the query succeeded.

PEP 249, the Python Database API Specification v2.0, requires autocommit to be initially turned off. Django overrides this default and turns autocommit on.

To avoid this, you can deactivate the transaction management, but it isn’t recommended.

---
### Deactivating transaction management

You can totally disable Django’s transaction management for a given database by setting `AUTOCOMMIT` to `False` in its configuration. If you do this, Django won’t enable autocommit, and won’t perform any commits. You’ll get the regular behavior of the underlying database library.

This requires you to commit explicitly every transaction, even those started by Django or by third-party libraries. Thus, this is best used in situations where you want to run your own transaction-controlling middleware or do something really strange.

---
## Performing actions after commit

Sometimes you need to perform an action related to the current database transaction, but only if the transaction successfully commits. Examples might include a background task, an email notification, or a cache invalidation.

`on_commit()` allows you to register callbacks that will be executed after the open transaction is successfully committed:

`on_commit`(func, using=None, robust=False)
: Pass a function, or any callable, to on_commit():

```python
from django.db import transaction


def send_welcome_email():
    ...


transaction.on_commit(send_welcome_email)
```

Callbacks will not be passed any arguments, but you can bind them with `functools.partial()`:

```python
from functools import partial

for user in users:
    transaction.on_commit(partial(send_invite_email, user=user))
```

Callbacks are called after the open transaction is successfully committed. If the transaction is instead rolled back (typically when an unhandled exception is raised in an `atomic()` block), the callback will be discarded, and never called.

If you call `on_commit()` while there isn’t an open transaction, the callback will be executed immediately.

It’s sometimes useful to register callbacks that can fail. Passing `robust=True` allows the next callbacks to be executed even if the current one throws an exception. All errors derived from Python’s `Exception` class are caught and logged to the `django.db.backends.base` logger.

You can use `TestCase.captureOnCommitCallbacks()` to test callbacks registered with `on_commit()`.

> **Changed in Django 4.2:**
> The robust argument was added.
# django-cheat-sheet
> A cheat-sheet to make your Django apps better.

This reposetory contains my notes on Django concepts which I learn from various sources.

## Contents

- [Designing Models](#designing-models)
- [Database optimization](#database-optimization)
- [Managers](#managers)
- [Middleware](#middleware)
- [Signals](#signals)
- [Security](#security)

## Designing Models

- Use singular model name. E.g `class Organization(models.Model)` and not `class Organizations(models.Model)`

- Use snake case for model field names. E.g. `first_name` and not `FirstName`

- [Many-to-one relationships](https://docs.djangoproject.com/en/3.0/topics/db/examples/many_to_one/): Use  ForeignKey to define ManyToOne relationship. E.g.
  If you want to design models for team and player where Many Player can belong to One Team do:

  ```python

    class Team(models.Model):
        name = models.CharField(max_length=100)

    class Player(models.Model):
        name = models.CharField(max_length=100)

        # Many Player can belong to one team
        team = models.ForeignKey(Team, ...)
  ```

- Reverse Relationships: Use `related_name` if you want to get all the players for a team.

```python
    class Player(models.Model):
        team = models.ForeignKey(Team, related_name='players', ...)

    # Now you can do:
    team1 = Team.objects.get(name='team1')

    # Get all the players for this team
    players = team1.players.all()
```

- Use `related_query_name` if you want get all the teams that has a Player named `Krishna`

```python
    class Player(models.Model):
        team = models.ForeignKey(
            Team,
            related_name='players',
            related_query_name='player',
            ...
        )

    # Now you can do:
    teams = Team.objects.filter(player__name='Krishna')
```

- [One-to-one relationships](https://docs.djangoproject.com/en/3.0/topics/db/examples/one_to_one/) Use OneToOne Field when defining One-To-One Relationships. E.g. Suppose that each Team has a Coach. And one Coach can only belong to one Team.

  ```python
    class Team(models.Model):
        name = models.CharField(max_length=100)

    class Coach(models.Model):
        name = models.CharField(max_length=100)
        ...
        ...
        team = OneToOneField(Coach, ...)
  ```

  There is one confusion that you might think of is that in which Model you should add the OneToOneField. In that case one rule that I follow is "Don't disturbe the existing models if not needed.":
  Suppose that you added `coach` field to `Team` model instead of the above.

  - It will create two migrations
  - You will have to update team exclusively while creating Coach.

    ```python
        # Before
        team1 = Team.objects.get(name='team1')
        Coach.objects.create(name='tt', team=team1)

        # After
        team1 = Team.objects.get(name='team1')
        coach = Coach.objects.create(name='tt')
        team1.coach = coach
        team1.save()
    ```

- ForeignKey and OneToOneField accept one required parameter `on_delete` which decides what will happen to your model when the data it is linked to gets deleted. For example look at the following:

    ```python
    class Team(models.Model):
        name = models.CharField(max_length=100)

    class Player(models.Model):
        name = models.CharField(max_length=100)

        # Many Player can belong to one team
        team = models.ForeignKey(Team, on_delete=models.CASCADE)
    ```

    In the above example `on_delete` is set to `CASCADE` which means that delete all the Players in the team, when the team they are linked to gets deleted. But if it is set to `SET_NULL` it means that keep they players and set the `player.team` to `None` when the team is getting deleted.

- Any changes in your models requires you to run `python manage.py makemigrations` and `python manage.py migrate`

- Models should include all relevant domain/business logic as it's property or methods. For example for a player you might want to check if he is a captain or not:

    ```python
        class Player(models.Model):
            name = models.CharField(max_length=100)

            def is_captain(self):
                if self is captain:
                    return True
                return False
    ```

- Blank and Null Fields:

  - `Null` is database-related. Defines if a given database column will accept null values or not.

  - `Blank` is validation-related. It will be used during forms validation, when calling form.is_valid().

  - Do not use `null=True` for text/char based field. Otherwise, you will end up having two possible values for "no data" that is: `None` and `an empty` string. Having two possible values for "no data" is redundant The Django convention is to use the empty string, not NULL.

    ```python
        class Player(models.Model):
            name = models.CharField(max_length=100)

            # null=True is not required as `empty string` will be saved to the database.
            bio = models.TextField(blank=True)

            # here you may add `null=True`
            birth_date = models.DateField(null=True, blank=True)
    ```

  - Do not use `null=True` or `blank=True` for `BooleanField`. It is better to specify default values for such fields. If you realise that the field can remain empty, you need `NullBooleanField`.

- [`Class Meta`](https://docs.djangoproject.com/en/3.0/ref/models/options/)
  - can be used to set default order of a list of objects. E.g. if you want to list all players in alphabetical order this is how you do it:

    ```python
        class Player(models.Model):
            name = models.CharField(max_length=100)

            class Meta:
                ordering = ['-name']
    ```

    Now `Player.objects.all()` will return queryset of players in an alphabetical order but [ordering is expensive](https://docs.djangoproject.com/en/3.0/topics/db/optimization/#don-t-order-results-if-you-don-t-care) so you should not use it when you don't need it.

    You might want to add an index to your database which may help to improve ordering performance:

    ```python
        class Player(models.Model):
            name = models.CharField(max_length=100)

            class Meta:
                indexes = [models.Index(fields=['name'])]
                ordering = ['-name']
    ```

  - can also be used to set abstract model by setting `abstract=True` in the `meta class`. This model will then not be used to create any database table. Instead, when it is used as a base class for other models, its fields will be added to those of the child class.

## Database optimization

- Use `QuerySet.explain()` to understand how specific QuerySets are executed by your database. Also checkout [django-debug-toolbar](https://github.com/jazzband/django-debug-toolbar/)

- Use indexes to fields that you frequently query by using `filter()`, `exclude()`, `order_by` etc.

- Use indexed field to query:

    ```python
        # Faster
        player = Player.objects.get(id=10)

        # Slower. Assuming the name field is not indexed.
        player = Player.objects.get(name='Krishna')
    ```

- Use `filter()` and `exclude()` when needed. In case you want to perform some operation on only certain number of Players.

- [Use iterator](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#iterator) when dealing with `QuerySet` with large amount of data that you only need to access once.

- Use [`F expression`](https://docs.djangoproject.com/en/3.0/ref/models/expressions/#f-expressions) to update fields in the same model.

    ```python
    # Don't
    for player in Player.objects.all():
        player.rating += 1
        player.save()

    # Do
    Player.objects.update(rating=F('rating') + 1)
    ```

- Use `QuerySet.count()` if you only want the count, rather than `len(queryset)`.

- Use `QuerySet.exists()` if you only want to find out if at least one result exists, rather than `if queryset`

- Use `delete()` and `update()` when needed.:

  ```python
     # bulk update
     Player.objects.filter(name='Krishna').update(rating=10)

     # bulk delete
     Player.objects.filter(name='Krishna').delete()
  ```

- Use `ForeignKey` values directly

    ```python
       player = Player.objects.get(id=1)

       # Don't
       team_name = player.team.name

       # Do
       team_name = player.team_name
    ```

- Don't [order results if not needed](https://docs.djangoproject.com/en/3.0/topics/db/optimization/#don-t-order-results-if-you-don-t-care).

- Use [Bulk methods](https://docs.djangoproject.com/en/3.0/topics/db/optimization/#use-bulk-methods):
  - `bulk_create` Suppose that you have to create a lot of Players:

    ```python

        # Creating all the players at once is better
        Player.objects.bulk_create([
            Player(name='dummy player 1'),
            Player(headline='dummy player 2'),
            ...
        ])

        # Than creating one by one
        Player.objects.create(name='dummy player 1')
        Player.objects.create(name='dummy player 2')
        ...
    ```

  - `bulk_update` You can also update in bulk:

    ```python
        players = Player.objects.bulk_create([
            Player(name='dummy player 1'),
            Player(headline='dummy player 2'),
            ...
        ])

        players[0].rating = 4
        players[1].rating = 3
        ...

        # Updating all the players rating at once is better than
        Player.objects.bulk_update(players, ['rating'])

        # Updating them one by one
        players[0].save()
    ```

- Use [`select_related`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#select-related) and [`preftech_related`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#prefetch-related): These methods are used to improve speed by minimising the number of database query when dealing with related objects.

  - `select_related` is used when you're dealing with ForeignKey and OneToOne Relations. Use when you want to get detail of one related object.

  ```python
    # Each player will be related to a team so why not get the team information also
    players = Player.objects.all().select_releted('team')
  ```

  - `prefetch_related` is used when you're dealing with ManyToMany Relations. Use when you want to get details of many related object.

  ```python
    # Each player can belong to multiple groups so why not get those groups also.
    players = Player.objects.all().prefecth_relted('groups')
  ```

## Managers

- Managers are the interface through which database query operations are provided to Django models.

- At least one `Manager` should exist for every model in a Django application.

- `objects` is the default manager that is added to every model.

- You can define a manager for a model by calling `models.Manager()`:

    ```python
    from django.db import models

    class Player(models.Model):
        #...
        people = models.Manager()
    ```

    Now you can do `Player.people.all()` instead of `Player.objects.all()`

- You can also have custom managers:
  - Helpful when you need extra methods to your manager:

    ```python
    class CaptainManager(models.Manager):
        def get_captain(self):
            return captain

    class Player(models.Model):
        ...
        objects = PollManager()
    ```

    Now you can do `Player.objects.get_captain()` with all the other methods on `objects`(`.all()`, `filter()`..etc.)

  - Helpful in modifying the intial queryset that manager returns:

   ```python
    class HighRatedPlayerManager(models.Manager):
        def get_queryset(self):
            return super().get_queryset().filter(rating='5')

    class Player(models.Model):
        title = models.CharField(max_length=100)
        author = models.CharField(max_length=50)

        objects = models.Manager()
        five_stars = HighRatedPlayerManager()
   ```

   With `Player.objects.all()` you can get all the players but with `Player.five_starts.all()` you only get players with rating 5.

## Middleware

- Middleware are hooks to modify Django request and response object.
  - You can use middleware to modify the request object which is sent to the view. E.g. every time you do `request.user` you get the user object. This functionality is provided via `AuthenticationMiddleware` which modifies the request object to add user to the request before it is sent to the view.

  - You can also use middleware to modify the response object which is returned from the view.

  - Ordering of middlewares in your settings is very important. While processing request object middlware works from top to bottom and while processing response object middleware works from bottom to top.

  - Think of it like an onion where your view is the core of the onion. So when request comes, it will be passed through each of middlware from top to bottom. And then the view will be executed. After that the response of the view will be passed through each of the middleware from bottom to top.
  
  - Here is how you define a custom middleware:

  ```python
  class SimpleMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        # One-time configuration and initialization.

    def __call__(self, request):
        # Code to be executed for each request before
        # the view (and later middleware) are called.

        response = self.get_response(request)

        # Code to be executed for each request/response after
        # the view is called.

        return response
  ```

  To modify the request do something before the line `response = self.get_response(request)` and then that request will be passed to the next middleware(from top to bottom) or if no middleware is there then it will be passed to your views.

  Similarly to modify the response do something after the line `response = self.get_response(request)` and then your modified response will be passed to the next middleware from bottom to top.

  - To activate a custom middleware you need to add it in your settings file in middleware section.

  - To know more about ordering of a middlware checkout [1](https://docs.djangoproject.com/en/3.0/ref/middleware/#middleware-ordering).

## Signals

- The Django Signals is a strategy to allow decoupled applications to get notified when certain events occur.
  - Let's say you want to send some message to slack everytime instance of a model is updated, but there are several place in your code base that this model can be updated. Now you don't want to call the send_message method everytime you're updating the model. You can do that via a signal hook which will get triggered every time this specific model's save method is triggered.

  - Signals are often misused, here are few tips to verify that you really need a signal:
    - Use when many pieces of the applications are interested in the same event.
    - Use when you are working with decoupled applications:
      - Django core model.
      - A model defined by third party app.
    - Don't use signals when you have both sender and reciever at one place.
      - Don't use signals in your views to send email.

  - Here is how you can implement signals in your django application:

    - Let's say I want to send a message everytime my `User` model is getting updated.

        ```python
        # Put this in your app's `signals.py`
        @receiver(post_save, sender=User)
        def send_message_to_slack(sender, **kwargs):
            send_message(sender)

        # Now you must ensure that the receiver code is imported when the projects is started
        # in your apps.py do

        class ProfileConfig(AppConfig):
            name = 'myproject.profile'

            def ready(self):
                import myproject.profile.signals # noqa

        # And in your __init__.py put:
        # Note that this is not required if you're already referring to your AppConfig
        # in the `INSTALLED_APPS` settings.
        default_app_config = 'myproject.profile.apps.ProfileConfig'
        ```

    - In the above example we used Django's built-in signal `post_save`, [there are many more](https://docs.djangoproject.com/en/3.0/topics/signals/) built-in signals that you can use and also define your own if needed.

## Security

- **Know your version and use a secure one**

    What version of Django are you using? The choice of version determines what known vulnerabilities are present, and potentially exploitable, in your application. Have a look at the archive of all the [security issues](https://docs.djangoproject.com/en/3.0/releases/security/) and closely follow Django's security release cycle.

- **Throttle user authentications**

    Django provides a lot of security features baked in, but the authentication system does not inherently protect against brute force attacks. It is important to write your own code to fix this, or use one of many open source solutions. (Like [django-defender](https://github.com/jazzband/django-defender))

- **Use raw queries and custom SQL with caution**

    While it may be tempting to write raw sql queries and custom SQL, doing so may open the door for an attack. A user attempting to perform an sql injection(execute arbitrary sql on a database) is going to find it much harder, if you always use the ORM.

- **Watch you headers**

    When the site is served via https, the referer request header is utilized by Django to help prevent cross site request forgery(CSRF) attacks. If you're too strict with your referer-policy header, you disable the functionality of Django's CSRF protection. In the end you need to weigh the privacy benefits of using a strict referer-policy header with the benefits of the CSRF protection.

- **Be careful with your cookies**

    Some cookies are more secure than others -- the default cookie behavior is to connect over http. However, since we already established that you need to use https, you want to make sure your cookies are only being sent over https as well. To prevent leaking cookies, be sure to set your SESSION_COOKIE_SECURE and CSRF_COOKIE_SECURE settings to True.

- **Carefully handle user uploads**

    If your web application allows users to upload files, you're opening yourself to an attack vector and the upload logic should therefore be handled carefully. It is important to validate all uploaded files to be sure they are what you expect.

- **Understand all your dependencies**

    Always keep checking at the release cycle of your dependencies and upgrading them accordingly. If one of your dependencies package is not maintained for a long time, look for alternatives or help maintaing the package.

- **Change the admin URL**

    Change the default admin URL from `/admin/` to something else. This is so common anyone will be looking for `youdomain/admin` and trying brute force attacks.

    ```python
        urlpatterns = [
            path('my-special-admin-login/', admin.site.urls),
        ]
    ```

- **Never run `debug=True` in Production.**

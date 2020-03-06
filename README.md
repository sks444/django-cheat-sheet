# django-cheat-sheet
> A cheat-sheet to make your Django apps better.

## Contents

- [Designing Models](#designing-models)
- [Caching](#caching)
- [Database optimization](#database-optimization)
- [Security](#security)
- [Deployments](#deployments)
- [Authentication](#authentication)
- [Authorization](#authorization)

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

- [`Class Meta`](https://docs.djangoproject.com/en/3.0/ref/models/options/) can be used to set default order of a list of objects. E.g. if you want to list all players in alphabetical order this is how you do it:

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

## Caching

- ToDo

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

- Do not perform operations on all the objects all the time, use `filter()` and `exclude()` when needed.

- [Use iterator](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#iterator) when dealing with `QuerySet` with large amount of data that you only need to access once.

- Use [`F expression`](https://docs.djangoproject.com/en/3.0/ref/models/expressions/#f-expressions) to update fields in the same model.

    ```python
    # Don't
    for player in Player.objects.all():
        player.rating += 1
        player.save()

    # Do
    Entry.objects.update(rating=F('rating') + 1)
    ```

## Security

- ToDo

## Deployments

- ToDo

## Authentication

- ToDo

## Authorization

- ToDo

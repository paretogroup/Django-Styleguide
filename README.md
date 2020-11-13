# Pareto - Django Styleguide

Django styleguide used in [HackSoft](https://hacksoft.io) projects.

Expect often updates as we discuss & decide upon different things.

**Table of contents:**

<!-- toc -->

- [Examples](#examples)
- [Overview](#overview)
- [Cookie Cutter](#cookie-cutter)
- [Models](#models)
  * [Custom validation](#custom-validation)
  * [Properties](#properties)
  * [Methods](#methods)
  * [Testing](#testing)
- [Services](#services)
- [Selectors](#selectors)
- [APIs & Serializers](#apis--serializers)
  * [An example list API](#an-example-list-api)
  * [An example detail API](#an-example-detail-api)
  * [An example create API](#an-example-create-api)
  * [An example update API](#an-example-update-api)
  * [Nested serializers](#nested-serializers)
- [Exception Handling](#exception-handling)
  * [Raising Exceptions in Services / Selectors](#raising-exceptions-in-services--selectors)
  * [Handle Exceptions in APIs](#handle-exceptions-in-apis)
- [Testing](#testing-1)
  * [Naming conventions](#naming-conventions)
  * [Example](#example)
    + [Example models](#example-models)
    + [Example selectors](#example-selectors)
    + [Example services](#example-services)
  * [Testing services](#testing-services)
  * [Testing selectors](#testing-selectors)
- [Inspiration](#inspiration)

<!-- tocstop -->

## Examples

Most of the examples are taken from HackSoft's Learning Management System - Odin - <https://github.com/HackSoftware/Odin>

## Overview

**In Django, business logic should live in:**

* Model properties (with some exceptions).
* Model `clean` method for additional validations (with some exceptions).
* Services - functions, that take care of code written to the database.
* Selectors - functions, that take care of code taken from the database.

**In Django, business logic should not live in:**

* APIs and Views.
* Serializers and Forms.
* Form tags.
* Model `save` method.

**Model properties vs selectors:**

* If the model property spans multiple relations, it should better be a selector.
* If a model property, added to some list API, will cause `N + 1` problem that cannot be easily solved with `select_related`, it should better be a selector.

## Cookie Cutter

We recommend starting every new project with [`cookiecutter-django`](https://github.com/pydanny/cookiecutter-django)

Once this is done, depending on the context, remove everything that's not needed.

The usual list is:

* `allauth`
* templates
* Settings for things that are not yet required (always add settings when necessary)

## Models

Lets take a look at an example model:

```python
class Course(models.Model):
    name = models.CharField(unique=True, max_length=255)

    start_date = models.DateField()
    end_date = models.DateField()

    attendable = models.BooleanField(default=True)

    students = models.ManyToManyField(
        Student,
        through='CourseAssignment',
        through_fields=('course', 'student')
    )

    teachers = models.ManyToManyField(
        Teacher,
        through='CourseAssignment',
        through_fields=('course', 'teacher')
    )

    slug_url = models.SlugField(unique=True)

    repository = models.URLField(blank=True)
    video_channel = models.URLField(blank=True, null=True)
    facebook_group = models.URLField(blank=True, null=True)

    logo = models.ImageField(blank=True, null=True)

    public = models.BooleanField(default=True)

    generate_certificates_delta = models.DurationField(default=timedelta(days=15))

    objects = CourseManager()

    def clean(self):
        if self.start_date > self.end_date:
            raise ValidationError("End date cannot be before start date!")

    def save(self, *args, **kwargs):
        self.full_clean()
        return super().save(*args, **kwargs)

    @property
    def visible_teachers(self):
        return self.teachers.filter(course_assignments__hidden=False).select_related('profile')

    @property
    def duration_in_weeks(self):
        weeks = rrule.rrule(
            rrule.WEEKLY,
            dtstart=self.start_date,
            until=self.end_date
        )
        return weeks.count()

    @property
    def has_started(self):
        now = get_now()

        return self.start_date <= now.date()

    @property
    def has_finished(self):
        now = get_now()

        return self.end_date <= now.date()

    @property
    def can_generate_certificates(self):
        now = get_now()

        return now.date() <= self.end_date + self.generate_certificates_delta

    def __str__(self) -> str:
        return self.name
```

Few things to spot here.

**Custom validation:**

* There's a custom model validation, defined in `clean`. This validation uses only model fields and no relations.
* This requires someone to call `full_clean()` on the model instance. The best place to do that is in the `save()` method of the model. Otherwise people can forget to call `full_clean()` in the respective service.

**Properties:**

* All properties, expect `visible_teachers` work directly on model fields.
* `visible_teachers` is a great candidate for a **selector**.

We have few general rules for custom validations & model properties / methods:

### Custom validation

* If the custom validation depends only on the **non-relational model fields**, define it in `clean` and call `full_clean` in `save`.
* If the custom validation is more complex & **spans relationships**, do it in the service that creates the model.
* It's OK to combine both `clean` and additional validation in the `service`.


### Properties

* If your model properties use only **non-relational model fields**, they are OK to stay as properties.
* If a property, such as `visible_teachers` starts **spanning relationships**, it's better to define a selector for that.


### Methods

* If you need a method that updates several fields at once (for example - `created_at` and `created_by` when something happens), you can create a model method that does the job.
* Every model method should be wrapped in a service. There should be no model method calling outside a service.

### Testing

Models need to be tested only if there's something additional to them - like custom validation or properties.

If we are strict & don't do custom validation / properties, then we can test the models without actually writing anything to the database => we are going to get quicker tests.

For example, if we want to test the custom validation, here's how a test could look like:

```python
from datetime import timedelta

from django.test import TestCase
from django.core.exceptions import ValidationError

from odin.common.utils import get_now

from odin.education.factories import CourseFactory
from odin.education.models import Course


class CourseTests(TestCase):
    def test_course_end_date_cannot_be_before_start_date(self):
        start_date = get_now()
        end_date = get_now() - timedelta(days=1)

        course_data = CourseFactory.build()
        course_data['start_date'] = start_date
        course_data['end_date'] = end_date

        course = Course(**course_data)

        with self.assertRaises(ValidationError):
            course.full_clean()

```

There's a lot going on in this test:

* `get_now()` returns a timezone aware datetime.
* `CourseFactory.build()` will return a dictionary with all required fields for a course to exist.
* We replace the values for `start_date` and `end_date`.
* We assert that a validation error is going to be raised if we call `full_clean`.
* We are not hitting the database at all, since there's no need for that.

Here's how `CourseFactory` looks like:

```python
class CourseFactory(factory.DjangoModelFactory):
    name = factory.Sequence(lambda n: f'{n}{faker.word()}')
    start_date = factory.LazyAttribute(
        lambda _: get_now()
    )
    end_date = factory.LazyAttribute(
        lambda _: get_now() + timedelta(days=30)
    )

    slug_url = factory.Sequence(lambda n: f'{n}{faker.slug()}')

    repository = factory.LazyAttribute(lambda _: faker.url())
    video_channel = factory.LazyAttribute(lambda _: faker.url())
    facebook_group = factory.LazyAttribute(lambda _: faker.url())

    class Meta:
        model = Course

    @classmethod
    def _build(cls, model_class, *args, **kwargs):
        return kwargs

    @classmethod
    def _create(cls, model_class, *args, **kwargs):
        return create_course(**kwargs)
```

## Services

A service is a simple function that:

* Lives in `your_app/services.py` module
* Takes keyword-only arguments
* Is type-annotated (even if you are not using `mypy` at the moment)
* Works mostly with models & other services and selectors
* Does business logic - from simple model creation to complex cross-cutting concerns, to calling external services & tasks.

An example service that creates an user:

```python
def create_user(
    *,
    email: str,
    name: str
) -> User:
    user = User(email=email)
    user.full_clean()
    user.save()

    create_profile(user=user, name=name)
    send_confirmation_email(user=user)

    return user
```

As you can see, this service calls 2 other services - `create_profile` and `send_confirmation_email`


## Selectors

A selector is a simple function that:

* Lives in `your_app/selectors.py` module
* Takes keyword-only arguments
* Is type-annotated (even if you are not using `mypy` at the moment)
* Works mostly with models & other services and selectors
* Does business logic around fetching data from your database

An example selector that list users from the database:

```python
def get_users(*, fetched_by: User) -> Iterable[User]:
    user_ids = get_visible_users_for(user=fetched_by)

    query = Q(id__in=user_ids)

    return User.objects.filter(query)
```

As you can see, `get_visible_users_for` is another selector.

## APIs & Serializers

When using services & selectors, all of your APIs should look simple & the same.

General rules for an API is:

* Do 1 API per operation. For CRUD on a model, this means 4 APIs.
* Use the most simple `APIView` or `GenericAPIView`
* Use services / selectors & don't do business logic in your API.
* Use serializers for fetching objects from params - passed either via `GET` or `POST`
* Serializer should be nested in the API and be named either `InputSerializer` or `OutputSerializer`
  * `OutputSerializer` can subclass `ModelSerializer`, if needed.
  * `InputSerializer` should always be a plain `Serializer`
  * Reuse serializers as little as possible
  * If you need a nested serializer, use the `inline_serializer` util

### An example list API

```python
class CourseListApi(SomeAuthenticationMixin, APIView):
    class OutputSerializer(serializers.ModelSerializer):
        class Meta:
            model = Course
            fields = ('id', 'name', 'start_date', 'end_date')

    def get(self, request):
        courses = get_courses()

        data = self.OutputSerializer(courses, many=True)

        return Response(data)
```

### An example detail API

```python
class CourseDetailApi(SomeAuthenticationMixin, APIView):
    class OutputSerializer(serializers.ModelSerializer):
        class Meta:
            model = Course
            fields = ('id', 'name', 'start_date', 'end_date')

    def get(self, request, course_id):
        course = get_course(id=course_id)

        data = self.OutputSerializer(course)

        return Response(data)
```

### An example create API

```python
class CourseCreateApi(SomeAuthenticationMixin, APIView):
    class InputSerializer(serializers.Serializer):
        name = serializers.CharField()
        start_date = serializers.DateField()
        end_date = serializers.DateField()

    def post(self, request):
        serializer = self.InputSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        create_course(**serializer.validated_data)

        return Response(status=status.HTTP_201_CREATED)
```

### An example update API

```python
class CourseUpdateApi(SomeAuthenticationMixin, APIView):
    class InputSerializer(serializers.Serializer):
        name = serializers.CharField(required=False)
        start_date = serializers.DateField(required=False)
        end_date = serializers.DateField(required=False)

    def post(self, request, course_id):
        serializer = self.InputSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        update_course(course_id=course_id, **serializer.validated_data)

        return Response(status=status.HTTP_200_OK)
```

### Nested serializers

In case you need to use a nested serializer, you can do the following thing:

```python
class Serializer(serializers.Serializer):
    weeks = inline_serializer(many=True, fields={
        'id': serializers.IntegerField(),
        'number': serializers.IntegerField(),
    })
```

The implementation of `inline_serializer` can be found in `utils.py` in this repo.


## Exception Handling

### Raising Exceptions in Services / Selectors

Now we have separation between our HTTP interface & the core logic of our application.

In order to keep this separation of concerns, our services and selectors must not use the `rest_framework.exception` classes because they are bounded with HTTP status codes. 

Our services and selectors must use one of:

* [Python built-in exceptions](https://docs.python.org/3/library/exceptions.html)
* Exceptions from `django.core.exceptions`
* Custom exceptions, inheriting from the ones above.

Here is a good example of service that preforms some validation and raises `django.core.exceptions.ValidationError`:

```python
from django.core.exceptions import ValidationError

def create_topic(*, name: str, course: Course) -> Topic:
    if course.end_date < timezone.now():
       raise ValidationError('You can not create topics for course that has ended.')

    topic = Topic.objects.create(name=name, course=course)

    return topic
```

### Handle Exceptions in APIs

In order to transform the exceptions raised in the services or selectors, to a standard HTTP response, you need to catch the exception and raise something that the rest framework understands.

The best place to do this is in the `handle_exception` method of the `APIView`.

There you can map your exception to DRF exception.

Here is an example:

```python
from rest_framework import exceptions as rest_exceptions

from django.core.exceptions import ValidationError


class CourseCreateApi(SomeAuthenticationMixin, APIView):
    expected_exceptions = {
        ValidationError: rest_exceptions.ValidationError
    }

    class InputSerializer(serializers.Serializer):
        ...

    def post(self, request):
        serializer = self.InputSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        create_course(**serializer.validated_data)

        return Response(status=status.HTTP_201_CREATED)

    def handle_exception(self, exc):
        if isinstance(exc, tuple(self.expected_exceptions.keys())):
            drf_exception_class = self.expected_exceptions[exc.__class__]
            drf_exception = drf_exception_class(get_error_message(exc))

            return super().handle_exception(drf_exception)

        return super().handle_exception(exc)
```

Here's the implementation of `get_error_message`:

```python
def get_first_matching_attr(obj, *attrs, default=None):
    for attr in attrs:
        if hasattr(obj, attr):
            return getattr(obj, attr)

    return default


def get_error_message(exc):
    if hasattr(exc, 'message_dict'):
        return exc.message_dict
    error_msg = get_first_matching_attr(exc, 'message', 'messages')

    if isinstance(error_msg, list):
        error_msg = ', '.join(error_msg)

    if error_msg is None:
        error_msg = str(exc)

    return error_msg
```

You can move this code to a mixin and use it in every API to prevent code duplication. 

We call this `ExceptionHandlerMixin`. Here's a sample implementation from one of our projects:

```python
from rest_framework import exceptions as rest_exceptions

from django.core.exceptions import ValidationError

from project.common.utils import get_error_message


class ExceptionHandlerMixin:
    """
    Mixin that transforms Django and Python exceptions into rest_framework ones.
    without the mixin, they return 500 status code which is not desired.
    """
    expected_exceptions = {
        ValueError: rest_exceptions.ValidationError,
        ValidationError: rest_exceptions.ValidationError,
        PermissionError: rest_exceptions.PermissionDenied
    }

    def handle_exception(self, exc):
        if isinstance(exc, tuple(self.expected_exceptions.keys())):
            drf_exception_class = self.expected_exceptions[exc.__class__]
            drf_exception = drf_exception_class(get_error_message(exc))

            return super().handle_exception(drf_exception)

        return super().handle_exception(exc)
```

Having this mixin in mind, our API can be written like that:

```python

class CourseCreateApi(
  SomeAuthenticationMixin,
  ExceptionHandlerMixin,
  APIView
):
    class InputSerializer(serializers.Serializer):
        ...

    def post(self, request):
        serializer = self.InputSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        create_course(**serializer.validated_data)

        return Response(status=status.HTTP_201_CREATED)
```

All of code above can be found in `utils.py` in this repository.

## Testing

In our Django projects, we split our tests depending on the type of code they represent.

Meaning, we generally have tests for models, services, selectors & APIs / views.

The file structure usually looks like this:

```
project_name
├── app_name
│   ├── __init__.py
│   └── tests
│       ├── __init__.py
│       ├── models
│       │   └── test_some_model_name.py
│       ├── selectors
│       │   └── test_some_selector_name.pyy
│       └── services
│           ├── __init__.py
│           └── test_some_service_name.py
└── __init__.py
```

### Naming conventions

We follow 2 general naming conventions:

* The test file names should be `test_the_name_of_the_thing_that_is_tested.py`
* The test case shoud be `class TheNameOfTheThingThatIsTestedTests(TestCase):`

For example if we have:

```python
def a_very_neat_service(*args, **kwargs):
    pass
```

We are going to have the following for file name:

```
project_name/app_name/tests/services/test_a_very_neat_service.py
```

And the following for test case:

```python
class AVeryNeatServiceTests(TestCase):
    pass
```

For tests of utility functions, we follow a similiar pattern.

For example, if we have `project_name/common/utils.py`, then we are going to have `project_name/common/tests/test_utils.py` and place different test cases in that file.

If we are to split the `utils.py` module into submodules, the same will happen for the tests:

* `project_name/common/utils/files.py`
* `project_name/common/tests/utils/test_files.py`

We try to match the stucture of our modules with the structure of their respective tests.

### Example

We have a demo `django_styleguide` project.

#### Example models

```python
import uuid

from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone

from djmoney.models.fields import MoneyField


class Item(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    name = models.CharField(max_length=255)
    description = models.TextField()

    price = MoneyField(
        max_digits=14,
        decimal_places=2,
        default_currency='EUR'
    )

    def __str__(self):
        return f'Item {self.id} / {self.name} / {self.price}'


class Payment(models.Model):
    item = models.ForeignKey(
        Item,
        on_delete=models.CASCADE,
        related_name='payments'
    )

    user = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='payments'
    )

    successful = models.BooleanField(default=False)

    created_at = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return f'Payment for {self.item} / {self.user}'
```

#### Example selectors

For implementation of `QuerySetType`, check `queryset_type.py`.

```python
from django.contrib.auth.models import User

from django_styleguide.common.types import QuerySetType

from django_styleguide.payments.models import Item


def get_items_for_user(
    *,
    user: User
) -> QuerySetType[Item]:
    return Item.objects.filter(payments__user=user)
```

#### Example services

```python
from django.contrib.auth.models import User
from django.core.exceptions import ValidationError

from django_styleguide.payments.selectors import get_items_for_user
from django_styleguide.payments.models import Item, Payment
from django_styleguide.payments.tasks import charge_payment


def buy_item(
    *,
    item: Item,
    user: User,
) -> Payment:
    if item in get_items_for_user(user=user):
        raise ValidationError(f'Item {item} already in {user} items.')

    payment = Payment.objects.create(
        item=item,
        user=user,
        successful=False
    )

    charge_payment.delay(payment_id=payment.id)

    return payment
```

### Testing services

Service tests are the most important tests in the project. Usually, those are the heavier tests with most lines of code.

General rule of thumb for service tests:

* The tests should cover the business logic behind the services in an exhaustive manner.
* The tests should hit the database - creating & reading from it.
* The tests should mock async task calls & everything that goes outside the project.

When creating the required state for a given test, one can use a combination of:

* Fakes (We recommend using <https://github.com/joke2k/faker>)
* Other services, to create the required objects.
* Special test utility & helper methods.
* Factories (We recommend using [`factory_boy`](https://factoryboy.readthedocs.io/en/latest/orms.html))
* Plain `Model.object.create()` calls, if factories are not yet introduced in the project.

**Lets take a look at our service from the example:**

```python
from django.contrib.auth.models import User
from django.core.exceptions import ValidationError

from django_styleguide.payments.selectors import get_items_for_user
from django_styleguide.payments.models import Item, Payment
from django_styleguide.payments.tasks import charge_payment


def buy_item(
    *,
    item: Item,
    user: User,
) -> Payment:
    if item in get_items_for_user(user=user):
        raise ValidationError(f'Item {item} already in {user} items.')

    payment = Payment.objects.create(
        item=item,
        user=user,
        successful=False
    )

    charge_payment.delay(payment_id=payment.id)

    return payment

```

The service:

* Calls a selector for validation
* Create ORM object
* Calls a task

**Those are our tests:**

```python
from unittest.mock import patch

from django.test import TestCase
from django.contrib.auth.models import User
from django.core.exceptions import ValidationError

from django_styleguide.payments.services import buy_item
from django_styleguide.payments.models import Payment, Item


class BuyItemTests(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(username='Test User')
        self.item = Item.objects.create(
            name='Test Item',
            description='Test Item description',
            price=10.15
        )

        self.service = buy_item

    @patch('django_styleguide.payments.services.get_items_for_user')
    def test_buying_item_that_is_already_bought_fails(self, get_items_for_user_mock):
        """
        Since we already have tests for `get_items_for_user`,
        we can safely mock it here and give it a proper return value.
        """
        get_items_for_user_mock.return_value = [self.item]

        with self.assertRaises(ValidationError):
            self.service(user=self.user, item=self.item)

    @patch('django_styleguide.payments.services.charge_payment.delay')
    def test_buying_item_creates_a_payment_and_calls_charge_task(
        self,
        charge_payment_mock
    ):
        self.assertEqual(0, Payment.objects.count())

        payment = self.service(user=self.user, item=self.item)

        self.assertEqual(1, Payment.objects.count())
        self.assertEqual(payment, Payment.objects.first())

        self.assertFalse(payment.successful)

        charge_payment_mock.assert_called()
```

### Testing selectors

Testing selectors is also an important part of every project.

Sometimes, the selectors can be really straightforward, and if we have to "cut corners", we can omit those tests. But it the end, it's important to cover our selectors too.

Lets take another look at our example selector:

```python
from django.contrib.auth.models import User

from django_styleguide.common.types import QuerySetType

from django_styleguide.payments.models import Item


def get_items_for_user(
    *,
    user: User
) -> QuerySetType[Item]:
    return Item.objects.filter(payments__user=user)
```

As you can see, this is a very straighforward & simple selector. We can easily cover that with 2 to 3 tests.

**Here are the tests:**

```python
from django.test import TestCase
from django.contrib.auth.models import User

from django_styleguide.payments.selectors import get_items_for_user
from django_styleguide.payments.models import Item, Payment


class GetItemsForUserTests(TestCase):
    def test_selector_returns_nothing_for_user_without_items(self):
        """
        This is a "corner case" test.
        We should get nothing if the user has no items.
        """
        user = User.objects.create_user(username='Test User')

        expected = []
        result = list(get_items_for_user(user=user))

        self.assertEqual(expected, result)

    def test_selector_returns_item_for_user_with_that_item(self):
        """
        This test will fail in case we change the model structure.
        """
        user = User.objects.create_user(username='Test User')

        item = Item.objects.create(
            name='Test Item',
            description='Test Item description',
            price=10.15
        )

        Payment.objects.create(
            item=item,
            user=user
        )

        expected = [item]
        result = list(get_items_for_user(user=user))

        self.assertEqual(expected, result)
```

## Inspiration

The way we do Django is inspired by the following things:

* The general idea for **separation of concerns**
* [Boundaries by Gary Bernhardt](https://www.youtube.com/watch?v=yTkzNHF6rMs)
* Rails service objects

## Service layer

The goal of a service is to concentrate the business logic for the operations of an application.
It knows all Models that should be part of the flow and knows the API of those models. It also orchestrate all the side-effects and therefore can make the use of other services.

Adding this layer to your application will make your process easy to test, since you don't need to spin up an entire request to test the flow, and allow you to mock the other layers. Thus, you can test the logic individually.

#### Example 1.1

So, let's imagine a process to registers new users. Whenever a new user register herself, we need to check whether the username already exists or not. In case it doesn't, the process should send a Welcome email and persist the new user. 

If it already exists, the application should raise an error.

A simple model for UserAccount could be something like this.

```py
from django.contrib.auth.models import AbstractUser, UserManager


class UserAccountManager(UserManager):
    
    def find_by_username(self, username):
        queryset = self.get_queryset()
        return queryset.filter(username=username)


class UserAccount(AbstractUser):
    objects = UserAccountManager()

```


Adding test for this custom manager (Assuming we have installed pytest, pytest-mock, and django-mock-queries).



```py
from django_mock_queries.query import MockSet

from account.models import UserAccount


class TestFindByUsername:

    def test_when_username_exist_return_user_account(self, mocker):
        expected_result = [
            UserAccount(
                username='john.smith',
                email='johnsmith@example.com'
            )
        ]
        qs_mock = MockSet(expected_result[0])

        mocker.patch.object(
            UserAccount.objects,
            'get_queryset',
            return_value=qs_mock
        )

        result = list(UserAccount.objects.find_by_username('john.smith'))
        assert result == expected_result

```

Now, let's create a class that will send the Welcome email.

```py
from django.core.mail import send_mail


class WelcomeEmail:
    subject = 'Welcome new user!'
    body = 'We are happy to have you here!'
    from_email = 'welcome@example.com'

    @classmethod
    def send_to(cls, emails):
        send_mail(
            cls.subject,
            cls.body,
            cls.from_email,
            emails
        )

```

adding tests,

```py
from account.emails import WelcomeEmail


class TestSendTo:

    def test_send_email(self, mocker):
        mock_send_mail = mocker.patch('account.emails.send_mail')

        WelcomeEmail.send_to(['john.smith@example.com'])
        assert mock_send_mail.called is True

```

Building the use case now.

```py
from django.utils.translation import gettext as _

from account.models import UserAccount
from account.emails import WelcomeEmail


class UsernameAlreadyExistError(Exception):
    pass


class RegisterUserAccount:

    def __init__(
        self,
        username,
        email,
        password,
        first_name=None,
        last_name=None,
    ):
        # Set the internal state for the operation
        self._username = username
        self._email = email
        self._password = password
        self._first_name = first_name
        self._last_name = last_name
    
    def execute(self):
        self.valid_data()
        user_account = self._factory_user_account()

        self._send_welcome_email_to(user_account)
        return user_account

    def valid_data(self):
        # It is a public method to allow clients of this object to validate
        # the data even before to execute the use case.
        user_account_qs = UserAccount.objects.find_by_username(self._username)
        if user_account_qs.exists():
            # Raise a meaningful error to be catched by the client
            error_msg = (
                'There is an user account with {} username. '
                'Please, try another username'
            ).format(self._username)

            raise UsernameAlreadyExistError(_(error_msg))

        return True
    
    def _send_welcome_email_to(self, user_account):
        WelcomeEmail.send_to([user_account.email])

    def _factory_user_account(self):
        # Factory to create an user account. Ideally, it would be
        # implemented by the UserAccount manager to simplify the
        # API. But it was implemented here for the purpose of this
        # tutorial
        return UserAccount.objects.create_user(
            self._username,
            self._email,
            self._password,
            **{
                'first_name': self._first_name,
                'last_name': self._last_name
            }
        )

```


adding tests,

```py
import pytest

from account.models import UserAccount
from account.usecases import (
    RegisterUserAccount,
    UsernameAlreadyExistError
)


def create_user(username, email, password, **extra_fields):
    # mock for "create_user" method on the UserAccount model
    # to avoind touching the database. Unit tests should not
    # touch databases :)

    return UserAccount(
        username=username,
        email=email,
        password=password,
        **extra_fields
    )


@pytest.fixture
def mock_execute_dependencies(mocker):
    # we don't need to test django again
    mocker.patch.object(UserAccount.objects, 'create_user', side_effect=create_user)
    # already tested on TestValidData
    mocker.patch.object(RegisterUserAccount, 'valid_data', return_value=None)
    # Already tested on test_welcome_email.py file
    mocker.patch.object(WelcomeEmail, 'send_to', return_value=None)


class TestExecute:
    # Test the execute method on RegisterUserAccount use case

    def setup_method(self):
        # setup method will be executed on each test
        self._use_case = RegisterUserAccount(
            username='john.smith',
            email='john.smith@example.com',
            password='P@ssword',
            first_name='John',
            last_name='Smith',
        )

    def test_return_user_account_type(self, mock_execute_dependencies):
        result = self._use_case.execute()
        assert isinstance(result, UserAccount)

    def test_create_user_account_with_first_name(self, mock_execute_dependencies):
        expected_result = 'John'
        user_account = self._use_case.execute()
        assert user_account.first_name == expected_result

    def test_create_user_account_with_last_name(self, mock_execute_dependencies):
        expected_result = 'Smith'
        user_account = self._use_case.execute()
        assert user_account.last_name == expected_result


class TestValidData:
    # Test the valid_data method on RegisterUserAccount use case
  
    def test_when_username_already_exists_raise_username_already_exist_error(self, mocker):
        qs_mock = MockSet(
            UserAccount(
                username='john.smith',
                email='john.smith@example.com'
            )
        )

        # We don't need to test this method again,
        # since it has been tested by the
        # test_user_account_manager.py file.
        mocker.patch.object(
            UserAccount.objects,
            'find_by_username',
            return_value=qs_mock
        )

        with pytest.raises(UsernameAlreadyExistError):
            use_case = RegisterUserAccount(
                'john.smith',
                'john.smith@example.com',
                'P@ssword9'
            )

            use_case.valid_data()

    def test_when_username_does_not_exists_returns_true(self, mocker):
        expected_result = True
        qs_mock = MockSet()

        mocker.patch.object(
            UserAccount.objects,
            'find_by_username',
            return_value=qs_mock
        )

        use_case = RegisterUserAccount(
            'john.smith',
            'john.smith@example.com',
            'P@ssword9'
        )

        result = use_case.valid_data()
        assert result == expected_result

```


Uhu! We've created and tested the whole "business logic" of our application! And we didn't have to implement the view yet. We're are able to validate our flow, run tests against it, and reuse this whenever we want.
However, let's keep moving down the path and implement the rest of the layers.
I will start with the RegisterAccountForm.


```py
from django import forms
from django.utils.translation import gettext as _


class RegisterAccountForm(forms.Form):

    username = forms.CharField(required=True)
    first_name = forms.CharField(required=True)
    last_name = forms.CharField(required=True)
    password = forms.CharField(required=True, widget=forms.PasswordInput())
    confirm_password = forms.CharField(required=True, widget=forms.PasswordInput())

    def clean_confirm_password(self) -> str:
        # note that confirm the password is an input validation.
        # the business logic does not care about confirmation.
        password = self.cleaned_data.get('password')
        confirm_password = self.cleaned_data.get('confirm_password')

        if confirm_password != password:
            raise forms.ValidationError(
                _('Password and confirmation do not match each other')
            )

        return confirm_password

```


Don't forget the tests.


```py
from account.forms.register_account_form import RegisterAccountForm


class TestRequired:

    def test_when_username_is_empty_returns_error_with_username(self):
        data = {
            'username': None,
            'first_name': 'John',
            'last_name': 'Smith',
            'password': 'P@ssword99',
            'confirm_password': 'P@ssword99',
        }

        form = RegisterAccountForm(data=data)
        form.is_valid()
        assert 'username' in form.errors
    
    def test_when_first_name_is_empty_returns_error_with_first_name(self):
        data = {
            'username': 'johnsmith',
            'first_name': None,
            'last_name': 'Smith',
            'password': 'P@ssword99',
            'confirm_password': 'P@ssword99',
        }

        form = RegisterAccountForm(data=data)
        form.is_valid()
        assert 'first_name' in form.errors
       
    def test_when_last_name_is_empty_returns_error_with_last_name(self):
        data = {
            'username': 'johnsmith',
            'first_name': 'John',
            'last_name': None,
            'password': 'P@ssword99',
            'confirm_password': 'P@ssword99',
        }

        form = RegisterAccountForm(data=data)
        form.is_valid()
        assert 'last_name' in form.errors

    def test_when_password_is_empty_returns_error_with_password(self):
        data = {
            'username': 'johnsmith',
            'first_name': 'John',
            'last_name': 'Smith',
            'password': None,
            'confirm_password': 'P@ssword99',
        }

        form = RegisterAccountForm(data=data)
        form.is_valid()
        assert 'password' in form.errors

    def test_when_confirm_password_is_empty_returns_error_with_confirm_password(self):
        data = {
            'username': 'johnsmith',
            'first_name': 'John',
            'last_name': 'Smith',
            'password': 'P@ssword99',
            'confirm_password': None,
        }

        form = RegisterAccountForm(data=data)
        form.is_valid()
        assert 'confirm_password' in form.errors

    def test_when_confirm_password_is_different_from_password_returns_error_with_confirm_password(self):
        data = {
            'username': 'johnsmith',
            'first_name': 'John',
            'last_name': 'Smith',
            'password': 'P@ssword99',
            'confirm_password': 'P@ssword91',
        }

        form = RegisterAccountForm(data=data)
        form.is_valid()
        assert 'confirm_password' in form.errors

    def test_when_all_fields_are_filled_up_and_password_is_equal_to_confirm_password_the_form_is_valid(self):
        password = 'P@ssword99'
        data = {
            'username': 'johnsmith',
            'first_name': 'John',
            'last_name': 'Smith',
            'password': password,
            'confirm_password': password,
        }

        form = RegisterAccountForm(data=data)
        assert form.is_valid() is True
```


and finally the view 


```py
from django.urls import reverse_lazy
from django.views.generic import FormView

from account.forms import RegisterAccountForm
from account.usecases import RegisterUserAccount


class RegisterView(FormView):
    template_name = 'account/register.html'
    form_class = RegisterAccountForm
    success_url = reverse_lazy('summary')

    def post(self, request, *args, **kwargs):
        form = self.get_form()

        if form.is_valid():
           # get the input and delegate to process
           self._run_use_case(form) 
           if not form.errors: 
               return self.form_valid(form) 

        return self.form_invalid(form)

    def _run_use_case(self, form):
        usecase = RegisterUserAccount(
            form.data['username'],
            form.data['email'],
            form.data['password'],
            first_name=form.data['first_name'],
            last_name=form.data['last_name']
        )

        # handle the output
        try:
            usecase.execute()
        except UsernameAlreadyExistError as err:
            form.add_error('username', str(err))

```


As we can see, views take care of the input and delegate the data process. Collect the output and return it to client.
Testing the view.


```py
from django.contrib.auth.models import AnonymousUser
from django.test import RequestFactory

from account.views import RegisterView
from account.usecases import RegisterUserAccount, UsernameAlreadyExistError


class TestPost:
  
    def test_when_form_is_valid_run_register_user_account_usecase(self, mocker):
        mock_execute = mocker.patch.object(
            RegisterUserAccount,
            'execute', 
            return_value=None
        )

        mocker.patch.object(RegisterView, 'form_valid', return_value=None)

        request = self._factory_request(mocker)
        self._run_view(request)
        assert mock_execute.called is True

    def test_when_form_is_valid_and_the_use_case_raise_exception_run_form_invalid(self, mocker):
        def mock_execute_usecase():
            raise UsernameAlreadyExistError

        mock_execute = mocker.patch.object(
            RegisterUserAccount,
            'execute',
            side_effect=mock_execute_usecase
        )

        mock_invalid = mocker.patch.object(
            RegisterView,
            'form_invalid',
            return_value=None
        )

        request = self._factory_request(mocker)
        self._run_view(request)
        assert mock_invalid.called is True

    def test_when_form_is_valid_and_use_case_run_successfuly_run_form_valid(self, mocker):
        mock_execute = mocker.patch.object(
            RegisterUserAccount,
            'execute',
            return_value=True
        )

        mock_valid = mocker.patch.object(
            RegisterView,
            'form_valid',
            return_value=None
        )

        request = self._factory_request(mocker)
        self._run_view(request)
        assert mock_valid.called is True

    def test_form_is_invalid_run_form_invalid(self, mocker):
        mock_invalid = mocker.patch.object(
            RegisterView,
            'form_invalid',
            return_value=None
        )

        request = mocker.Mock(
            POST={
                'username': None,
                'first_name': None,
                'last_name': None,
                'password': None,
                'confirm_password': None,
            }
        )
        self._run_view(request)
        assert mock_invalid.called is True
        
    def _factory_request(self, mocker):
        return mocker.Mock(
            POST={
                'username': 'johnsmith',
                'first_name': 'John',
                'last_name': 'Smith',
                'password': 'P@ssword99',
                'confirm_password': 'P@ssword99',
            }
        )

    def _run_view(self, request):
        view = RegisterView()
        view.request = request
        view.post(request)


class TestGet:

    def test_return_200(self):
        factory = RequestFactory()
        request = factory.get('/account/register/')
        request.user = AnonymousUser()

        response = RegisterView.as_view()(request)
        assert response.status_code == 200

```


Note that it doesn't test the logic here, neither if the Welcome email got sent or not. Also, there's no mention to the UserAccount model. The view doesn't even know that it exists. Therefore, we only need to mock the execute method of the use case.

Wow, it was a quite long journey until here, wasn't it? I want to do a quick recap.

Models handle data, guarantee integrity, and contain the behaviors that change their state.

Use cases execute the business logic and run the side effects.

Views receive the input, delegate the process, return the output.

Templates deal only with the markup that will be processed by the view.

That's how you start to decouple your application and make flows more reusable, testable, and reliable.

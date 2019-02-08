### 로그인 & 회원가입

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
# Create your models here.


class Account(models.Model):

    # django에서 제공하는 user 모델과 1:1 대응
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    phone = models.CharField(max_length=20)
    address = models.CharField(max_length=30)

    def __str__(self):
        return self.user.username
```

```python
# forms.py

from django import forms
from .models import Account
from django.contrib.auth.models import User


# 회원가입 시 유저 모델의 정보를 받기 위한 폼
class UserForm(forms.ModelForm):

    password = forms.CharField(widget=forms.PasswordInput())

    class Meta:
        model = User
        fields = ('username', 'email', 'password',)


# 회원가입 시 추가 정보(연락처, 주소) 정보를 받기 위한 폼
class AccountForm(forms.ModelForm):

    class Meta:
        model = Account
        fields = ('phone', 'address',)


# 로그인을 위한 폼
class LoginForm(forms.ModelForm):

    password = forms.CharField(widget=forms.PasswordInput())

    class Meta:
        model = User
        fields = ('username', 'password',)
```

```python
# views.py
from django.shortcuts import render
from django.shortcuts import redirect
from django.conf import settings
from django.contrib.auth import login, logout, authenticate
from django.http import Http404
from django.contrib.auth.hashers import make_password

from .forms import AccountForm
from .forms import UserForm
from .forms import LoginForm


def signupView(request):

    if request.method == 'POST':
        userForm = UserForm(request.POST)
        accountForm = AccountForm(request.POST)

        if userForm.is_valid() and accountForm.is_valid():
            userFormSave = userForm.save(commit=False)
            """
            이렇게 따로 폼 만들어서 회원가입할 때 비밀번호 해시 작업을 해줘야함
            (make_password 메소드) 안하면 회원가입은 되나, 추후 login() 메소드 이용 불가,
            에러 발생
            """
            userFormSave.password = make_password(userFormSave.password)
            userFormSave.save()

            account = accountForm.save(commit=False)
            account.user = userFormSave
            account.save()

            return redirect('accounts:loginView')

    userForm = UserForm()
    accountForm = AccountForm()

    ctx = {
        'accountForm': accountForm,
        'userForm': userForm,
    }

    return render(request, 'account/signup_template.html', ctx)


def loginView(request):

    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']
        # 로그인 인증
        user = authenticate(username=username, password=password)
        print(user)

        # 인증이 안됐으면 None을 반환하므로 조건에 따른 로그인 처리
        if user is not None:
            login(request, user)
            return redirect(settings.LOGIN_REDIRECT_URL)
        else:
            raise Http404('login failed')

    loginForm = LoginForm()
    return render(request, 'account/login_template.html', {'loginForm': loginForm})


def logoutView(request):
    # 로그아웃 메서드, POST요청이 왔을 때 사용하게끔 하는 것이 바람직할 듯, 테스트를 위해 반영x
    logout(request)
    return redirect('accounts:loginView')
```



> 두 번째 방법



```python
# models.py
from django import forms
from .models import Account
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth.forms import AuthenticationForm

"""
UserCreationForm을 상속 받으면 웬만한 회원가입 시 필요한 기능을 처리해줌. 아이디, 비밀번호 중복체크 등
아이디와 비밀번호 입력만 받기에 다른 필드도 추가
"""
class SignUpForm(UserCreationForm):

    email = forms.EmailField()
    address = forms.CharField(max_length=30)
    phone = forms.CharField(max_length=15)

    def clean_email(self):
        email = self.cleaned_data.get('email')
        if email is None:
            raise forms.ValidationError('Please write your email')
        return email

    def clean_address(self):
        address = self.cleaned_data.get('address')
        if address is None:
            raise forms.ValidationError('Please write your address')
        return address

    def clean_phone(self):
        phone = self.cleaned_data.get('phone')
        if phone is None:
            raise forms.ValidationError('Please write your phone number')
        return phone

    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data['email']
        user.address = self.cleaned_data['address']

        if commit:
            user.save()
        return user


"""
로그인을 위해 사용, 인증 절차도 반영이 된 폼이라 view에서 별도의 로직이 필요 없음
"""
class SignInForm(AuthenticationForm):
    error_messages = {
        'invalid_login': 'wrong id or password',
        'inactive': 'no user',
    }

```

```python
# views.py
from django.shortcuts import render
from django.shortcuts import redirect
from django.conf import settings

from .models import Account
from .forms import SignUpForm
from .forms import SignInForm

def signupView(request):

    signupForm = SignUpForm(request.POST or None)

    if request.method == 'POST':
        if signupForm.is_valid():

            # 아이디, 비밀번호, 이메일 저장
            form = signupForm.save()

            # 추가 정보는 Account 모델에 저장
            address = request.POST.get('address')
            phone = request.POST.get('phone')

            Account.objects.create(
                user_id=form.id,
                address=address,
                phone=phone
                )

            return redirect('accounts:loginView')

    ctx = {
        'signupForm': signupForm,
    }

    return render(request, 'account/signup_template.html', ctx)


def loginView(request):

    if request.method == 'POST':
        """
        AuthenticationForm를 상속 받은 폼은 view에서 사용할 때 첫 번째 파라미터로 request, 두 번째 파라미터로 데이터 값을 인자로 받음
        """
        loginForm = SignInForm(request=request, data=request.POST)
        if loginForm.is_valid():

            # 입력 받은 폼에서 유저 정보 가져옴, authenticate 메소드 안써도 됨
            user = loginForm.get_user()
            print(user)

            # 로그인
            login(request, user)
            return redirect(settings.LOGIN_REDIRECT_URL)

            # if user is not None:
            #     login(request, user)
            #     return redirect(settings.LOGIN_REDIRECT_URL)
            # else:
            #     raise Http404('login failed')

    loginForm = SignInForm()
    return render(request, 'account/login_template.html', {'loginForm': loginForm})
```


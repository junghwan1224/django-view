## view in django

**view 란?**

> view 는 Django 어플리케이션이 일반적으로 특정 기능과 템플릿을 제공하는 웹페이지의 한 종류 (django documentation)

> View는 필요한 데이타를 모델 (혹은 외부)에서 가져와서 적절히 가공하여 웹 페이지 결과를 만들도록 컨트롤하는 역할을 한다.
>
> 일반 MVC Framework에서 말하는 Controller와 비슷한 역할을 한다. 엄연히는 다름
>
> [참고](http://pythonstudy.xyz/python/article/306-Django-%EB%B7%B0-View)

> 뷰(*view*) 는 애플리케이션의 "로직"을 넣는 곳(djangogirls)



**view의 구조**

```python
# views.py
from django.shortcuts import render

def post_list(request):
    posts = Post.objects.all()
    return render(request, 'blog/blog_list.html', {'posts': posts})

def post_detail(request, pk):
    post = Post.objects.get(pk=pk)
```

```python
# urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('/post_list', views.post_list, name='post_list'),
    path('/post_detail/<int:pk>', views.post_detail),
]
# localhost:8000/post_list 로 이동하면 views.py의 post_list 함수 실행
# localhost:8000/post_detail/1
```

일반적인 함수형 view의 형태, 첫 번째 파라미터로 HTTP request를 받는다.

함수의 반환은 HttpResponse 객체를 반환한다.

```python
from django.http import HttpResponse
# def 로직은 생략
return HttpResponse('<p>hello world</p>')
```

```python
from django.http import HttpResponse
from django.template import loader

t = loader.get_template('myapp/index.html')
return HttpResponse(t.render)
```



**request**

view 함수에서의 첫 번째 파라미터인 request는 요청 및 응답에 대한 전반적인 처리를 담당하는 HttpRequest 객체를 의미

```python
def my_view(request):
    print(request.scheme) # usually http or https
    print(request.path) # "/post/list" not including scheme or domain
    
    # request.method
    if request.method == 'GET':
        do_get_something()
    elif request.method == 'POST':
        do_post_something()
```

use with ajax

```javascript
var username = 'jeonghwan';
var data = {
  'name': username
};

$('#button').click(function(e){
  $.post(url, data)
    .done(function(result){
    alert('ajax success');
  })
    .fail(function(){
    alert('ajax fail');
  })
})
```

```python
# views.py

def check_ajax(request):
    name = request.POST.get('name') # 'jeonghwan'
    name = request.POST['name'] # 'jeonghwan'
    """
    차이점?
    .get('name')은 POST로 들어온 값 중에서 'name'이 없으면 return None
    값이 없는 경우 두 번째 파라미터에  디폴트 값을  지정해줄 수 있음.
    name = request.POST.get('name', 'park')
    
    ['name']은 없으면 raise KeyError
    """
```



**shortcut(from django.shortcuts import *)**

view에서 사용하는 객체나 함수를 좀 더 편하게 사용할 수 있도록 지원하는 패키지

1. render

   ```python
   from django.shortcuts import render

   return render(request, template_name, context=None, content_type=None, status=None, using=None)

   # required arguments => request, template_name
   # context = 템플릿에 넘겨줄 값들의 dictionary로 ex) { 'post': post }
   # content_type = default 값은 'text/html', 문서 참고
   # status = 응답코드, default 값은 200
   # using = 템플릿을 로드하는데 사용되는 템플릿 엔진의 이름, 기본값은 엔진 클래스를 정의하는 모듈의 이름... 뭔 말인가...
   'BACKEND': 'django.template.backends.django.DjangoTemplates', ??
   ```

   ​


2. get_object_or_404

   ```python
   from django.shortcuts import get_object_or_404
   from .models import Post

   post = get_object_or_404(Post, pk=pk)
   """
   직독직해로도 알 수 있듯이 해당 모델에서 원하는 값을 반환 - call get()
   없으면 404 - raise Http404 ... not DoesNotExist 
   """
   ```



3. redirect

   ```python
   from django.shortcuts import redirect
   # view 함수 내에서 특정 url로 이동하고자 할 때 사용
   # url로 이동할 수 있는 HttpResponseRedirect 객체 반환
   def my_view(request):
       ...
       return redirect('https://example.com/')
   ```

   reverse() 나 get_absolute_url() 과 같이 쓰일 수 있다.

   - reverse()

     ```python
     # urls.py
     app_name = 'blog'

     urlpatterns = [
         path('/post_detail/<int:pk>', views.post_detail, name='post_detail'),
     ]
     ```

     ```python
     # views.py
     from django.shortcuts import redirect
     from django.urls import reverse

     def post_detail(request, pk):
         post = Post.objects.get(pk=pk)
         
         # name 값으로 url에 접근 가능, 스트링으로 반환
         # /post_detail/<int:pk> ... /post_detail/2
         url = reverse('blog:post_detail', args=[2])
         url = reverse('blog:post_detail', kwargs={'pk': post.pk})
         return redirect(url)
     ```

   ​

   - get_absolute_url()

     ```python
     # models.py

     class Post(models.Model):
         ...
         def get_absolute_url(self):
             return reverse('blog:post_detail', args=[self.id])
     ```

     ```python
     # views.py
     from django.shortcuts import redirect
     from .models import Post

     def post_detail(request, pk):
       post = Post.objects.get(pk=pk)
       return redirect(post)
     # get_absolute_url() method will be called to figure out the redirect URL
     ```

     ```html
     <!-- post_list.html -->
     <a href='{{ post.get_absolute_url }}'>{{ post.name }}</a>
     ```

   get_absolute_url() 메소드 이용시, view 로직이나 html에서 해당 url 작성이 간결해질 수 있음



**decorator**

[login_required](https://docs.djangoproject.com/en/2.1/topics/auth/default/#the-login-required-decorator)

[login_required Github](https://github.com/django/django/blob/master/django/contrib/auth/decorators.py)



[require_POST](https://docs.djangoproject.com/ko/2.1/topics/http/decorators/)

```python
# views.py
from django.contrib.auth.decorators import login_required
from django.views.decorators import require_POST
...

@login_required
def post_detail(request, pk):
    post = get_object_or_404(pk=pk)
    return render(request, 'post_detail.html', {'post': post})

@require_POST
def only_post(request, pk):
    post = get_object_or_404(Post, pk=pk)
    form = PostForm(request.POST)
    form.save()
```

[클로저 기본개념](https://bluese05.tistory.com/30)



**CBV(Class Based View)**


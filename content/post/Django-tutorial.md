---
title: Django tutorial
date: 2018-02-22 21:40:00
tags: [Python, Django]
---

通过 django 官网的 polls 项目教程，学习一下建立一个 django 网站所需要的基本的操作。

<!--more-->

# django 环境搭建
django 作为一个 Python 的包，可以通过包管理器 pip 来安装，当然也可以在官网下载，或者 clone GitHub 上的项目安装。


截至我写这篇操作记录的时候，官方最新的发行版是 django2.0 ，我个人强烈建议使用最新的 django2.0 。因为 2.0 之后添加了很多极其美滋滋的操作，比如 `<int:score>` 这种操作，比如 `path.include` 。


我在一年多之前写过一个 django 项目，当时使用的是 Python2 + djang1.x ，因此再拾起 django2.0 ，看到这些新的神奇的操作，发现 django 和我当时用的已经大不一样了，借鉴了 flask 的很多优秀的地方。直接上手 2.0 会让我们的生活更加美好～

---

# 创建一个项目
```shell
django-admin startproject <mysite>
```

---

# 启动内置的开发服务器
```shell
python manage.py runserver
```
注意：django2.0 之后仅支持 Python3，因此在同时有 Python2 和 Python3 环境的电脑下，需要用 python3 来替换 python  


默认情况下该命令在内部 ip 的 8000 端口启动开发服务器，如果想要更改其端口，可以修改命令行参数
```shell
python3 manage.py runserver 8080
```
该命令将在 8080 端口启动服务，而如果要监听所有的 ip（在其他电脑上访问）：
```shell
python manage.py runserver 0:8080
```
0 是 0.0.0.0 的快捷方式

---

# 创建一个应用
```shell
python manage.py startapp <myapp>
```
这将创建一个目录 <myapp>

---

# urls
django 通过一个名为 urlpatterns 的列表来匹配路由，列表中的每一项可以为一个路由对应一个函数（或类），也可以通过 include 的方式包含其他文件里面的路由规则。
```python
urlpatterns = [
    path('', views.index, name='index'),  # 对应 views 里面的 index 函数
    path('myapp/', include('myapp.urls')),  # 包含 myapp 的路由规则
]
```


## path 的参数

### route
route 是路由匹配的规则， path 不搜索请求的域名和 get 请求的参数，比如 /myapp/ 和 /myapp/?id=1 将搜索到同一个路由

### view
当 django 找到一个匹配的路由时，会把 HttpRequest 对象作为第一个参数传递给指定的视图函数

### kwargs
传递给视图函数的任意参数，在教程中没有应用

### name
该路由的命名，便于在其他文件中明确的引用它（尤其是模板中）。

---

# 数据库设置
django 的数据库设置在 mysite/settings.py 中，默认情况下 django 使用 sqlite3 数据库，这是一个基于文件的轻量级数据库，无需账号密码或端口等配置，可以直接使用。


如果想要使用其他 SQL数据库（如 MySQL 或者 PostgreSQL），那么需要修改 settings.py 中的 DATABASES 设置。当使用 sqlite3 数据库时，只需要填写 engine 与 name 即可，如果要使用其他数据库，则还需要端口、账号与密码等信息。
```python
# 使用 sqlite3
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': 'mydatabase',
    }
}


# 使用 postgresql
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydatabase',
        'USER': 'mydatabaseuser',
        'PASSWORD': 'mypassword',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}


# 一些其他的 engine
# 'django.db.backends.postgresql'
# 'django.db.backends.mysql'
# 'django.db.backends.sqlite3'
# 'django.db.backends.oracle'


```


在完成 settings.py 中的数据库设置后，可以通过 migrate 命令来创建必要的数据库表。要注意， mingrate 命令创建的数据库表同时基于 settings.py 的设置与数据库迁移脚本（之后会介绍如何创建数据库迁移脚本）。
```shell
python manage.py migrate
```

---

# 时区和语言设置
在启动项目之前，需要先修改项目对应的时区与语言。时区的作用不言而喻，而修改默认语言，可以使自带的 admin 后台管理变成中文的。
```python
LANGUAGE_CODE = 'zh-Hans'  # 修改语言为中文
TIME_ZONE = 'Asia/Shanghai'  # 修改时区
```

---

# 创建模型
django 推崇 Don’t repeat yourself (DRY) 原则，数据库迁移脚本中的模型将完全来源于你在代码中定义的模型。数据库迁移脚本将自动从你编写的模型中推断出迁移命令，不需要自己手动编写。此处以 django 的教程 polls 来介绍如何编写模型的定义。
```python
from django.db import models


class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```


关于这些字段的意思和使用方法，可以参考我的另一篇博客 [Field types](http://meik2333.com/index.php/archives/4/) 。


django 通过这些模型的定义代码来创建数据库表和创建访问 API ，但是首先，我们需要告诉我们的项目，我们创建了一个新的应用。


---


# 激活模型


为了将应用包含在我们的项目中，我们需要在 settings.py 中添加其配置。以 polls 应用为例： polls 的 PollsConfig 包含在 polls/apps.py 文件中，其对应的路径为 'polls.apps.PollsConfig' ，编辑 settings.py 文件，将其添加至 INSTALLED_APPS 中：


```python
INSTALLED_APPS  =  [ 
    'polls.apps.PollsConfig' ，
    'django.contrib.admin' ，
    'django.contrib.auth' ，
    'django.contrib.contenttypes' ，
    'django.contrib.sessions' ，
    'django.contrib.messages' ，
    'django.contrib.staticfiles' ，
]
```


现在 django 项目已经知道应该包含 polls 应用，此时运行另一个命令：


```shell
python manage.py makemigrations polls
```


通知 django 已经对模型进行了修改，重新生成数据库迁移脚本，可以通过以下命令进行查看：


```shell
python manage.py sqlmigrate polls 0001
```


之后，重新运行数据库迁移脚本以使刚刚对模型的修改生效。


```shell
python manage.py migrate
```


请记住进行模型修改需要做的三个步骤：


1. 修改你的模型（models.py）
2. 创建迁移脚本 python manage.py makemigrations
3. 运行迁移脚本 python manage.py migrate


---

# Playing with the API


请参阅[官方文档](https://docs.djangoproject.com/en/2.0/intro/tutorial02/#playing-with-the-api) 。


---


# 创建管理员用户


```shell
python manage.py createsuperuser
```


按照提示输入账号密码与邮箱即可。


---


# 在管理页面显示自己的模型


为了在 django 自带的 admin 中管理我们自己创建的模型，需要将模型注册到 admin ，修改 polls/admin.py


```python
from django.contrib import admin

from .models import Question

admin.site.register(Question)
```


---


# render()


一个视图的函数应该返回一个 Response 对象，然而在返回一个渲染后的模板时，我们却不需要费力去自己构建一个 Response 对象。


```python
from django.shortcuts import render

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)
```


---


# get_object_or_404()


我们可能会需要在没有找到一个对象的时候给出 404 的错误，这时候 get_object_or_404() 就很有用了。


```python
from django.shortcuts import get_object_or_404, render

from .models import Question
# ...
def detail(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/detail.html', {'question': question})
```


---


# 为 urls 模块命名


如果您开发的项目中包含多个应用，那么这些应用之中可能包含了相同 name 的 url ，可以通过给 urls 命名的方式防止在模板中引用的时候冲突。


```python
# polls/urls.py
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.index, name='index'),
    path('<int:question_id>/', views.detail, name='detail'),
    path('<int:question_id>/results/', views.results, name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```


```html
# polls/templates/polls/index.html
<li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>
```


---


# 一个简单的 form 示例


{% raw %}
```html
<h1>{{ question.question_text }}</h1>

{% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
{% for choice in question.choice_set.all %}
    <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}" />
    <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br />
{% endfor %}
<input type="submit" value="Vote" />
</form>
```

django 的表单提交默认有 csrf 防跨站检测，需要在 form 中添加 {% csrf_token %} 以自动添加一个隐藏的 csrf 字段。
{% endraw %}


---


# 测试


django 中的[测试](https://docs.djangoproject.com/en/2.0/intro/tutorial05/)


---


# 自定义管理表单


django 自带的 admin 功能及其强大，通过自定义，可以实现很多强大的功能。


这一章的篇幅较长且操作较为分散，直接给出官网的[文档](https://docs.djangoproject.com/en/2.0/intro/tutorial07/) 。


如果想要做深入的 admin 的自定义，那么可以看一下下面三个文档：


1. [The Django admin site](https://docs.djangoproject.com/en/2.0/ref/contrib/admin/)


2. [Admin actions](https://docs.djangoproject.com/en/2.0/ref/contrib/admin/actions/)


3. [The Django admin documentation generator](https://docs.djangoproject.com/en/2.0/ref/contrib/admin/admindocs/)
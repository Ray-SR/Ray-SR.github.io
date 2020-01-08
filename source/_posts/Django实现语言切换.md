---
title: Django实现语言切换
date: 2020-01-08 18:38:49
tags: Django
categories: Django
---

Django提供了实现多语言切换的中间件，可以实现模板（templates）、视图（views）以及JavaScript中语言的切换。基本的模式是在代码中对需要切换的内容加上标签，然后在翻译文件中填充翻译过后的字符串，最后编译即可。本文以中英文切换为例，介绍如果在Django中实现语言的切换。参考文档：[https://djangobook-cn.readthedocs.io/en/latest/chapter19.html](https://djangobook-cn.readthedocs.io/en/latest/chapter19.html)

<!--more-->

## 配置

- 首先在Django settings文件中加入中间件`django.middleware.locale.LocaleMiddleware`
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    # django语言国际化中间件
    'django.middleware.locale.LocaleMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

- 然后增加`LANGUAGES`和`LOCALE_PATHS`配置，`locale`文件夹需要手动创建：
```python
LANGUAGES = (
    ('zh-hans', '中文简体'),
    ('en', 'English'),
)

LOCALE_PATHS = (
    os.path.join(os.path.dirname(BASE_DIR), 'locale'),
)
```

## 修改html模板文件

- 首先需要在每个html第一行加入以下代码，可以将它放在公共页面
```python
{% load i18n %}
```
- 在需要翻译的字符串上加上trans标签,例如：
```python
<li><a href="/management/language_set"><i class="ti-settings"></i> {% trans '语言设置' %}</a></li>
```

- 多次重复翻译的内容可以设置成常量：
```python
{% trans "This is the title" as the_title %}
<title>{{ the_title }}</title>
<meta name="description" content="{{ the_title }}">
```

- 如果翻译的内容有django模板输出的变量，就需要用blocktrans和endblocktrans，例如：
```html
{% blocktrans %}This string will have {{ value }} inside.{% endblocktrans %}
```

## 后端视图

- 如果Django后端视图中有返回中文字符串也需要切换成英文，可以利用django的`gettext`模块，代码示例如下：
```python
from django.utils.translation import gettext as _

def test_views(request):
    response_str = _("中文字符串")
    return HttpResponse(response_str)
    
```

- 需要注意，Django渲染模板时需要使用`render`而不是`render_to_response`

## JavaScript中的语言转换

- 首先在根`urls.py`文件中的`urlpatterns`列表中添加如下代码：
```python
url(r'^jsi18n/$', JavaScriptCatalog.as_view(packages=['ProjectName']), name='javascript-catalog')
```

- 然后在模板中引入js(可放在公共页面)
```python
<script type="text/javascript" src="{% url 'javascript-catalog' %}"></script>
```

- 使用`gettext`在js中标记字符串，例如：
```javascript
function editpwd(){
        layer.open({
          type: 1,
          title: gettext('修改密码'),
          maxmin: true,
          shadeClose: true, //点击遮罩关闭层
          area : ['550px' , ''],
          content:$('#addsort_style'),
       })
    }
```

- 需要注意，必须将JavaScript代码单独放在.js文件中，不能写在html代码后面，否则后面生成翻译文件时检测不到

## 翻译文件生成与编译

- 添加完翻译的标记后，执行以下命令即可在`locale`文件夹下生成翻译文件：

`python manage.py makemessages -l en`
`python manage.py makemessages -d djangojs -l en`

- 执行成功后可以发现生成了`django.po`以及`djangojs.po`文件，然后就可以填充翻译后的字符串：
```python
#: DrBrain3/templates/Management/series_list.html:46
#: DrBrain3/templates/Users/new_psw.html:89
msgid "确认修改"
msgstr "Confirm the changes"
```
需要注意，有时由于识别的错乱，执行`python manage.py makemessages -l en`后.po文件中会出现fuzzy的字眼，此时需要将.po文件中这些出现错乱的都删除，正确的翻译文件如上图所示

- 填充完所有翻译后的字符串，就可以执行编译命令生成.mo文件

`python manage.py compilemessages`

## 如何切换语言
Django本身提供了语言切换的功能，可参考如下方法实现：

- 在项目根路由文件`urls.py`中添加切换语言的url
```python
url(r'^i18n/',include('django.conf.urls.i18n'))
```

- html页面中添加如下form表单
```python
<form action="{% url 'set_language' %}" method="post" id="change_language_form" enctype="multipart/form-data">
	{% csrf_token %}
    <input type="hidden" name="next" value=""/>
	<select class="change_language" name="language" id="language" onclick="changeLan()">
		{% for lang in LANGUAGES %}
        	<option value="{{ lang.0 }}"{% ifequal lang.0 LANGUAGE_CODE %} selected {% endifequal %}>{{ lang.1 }}</option>
		{% endfor %}
	</select>
</form>
```

- JavaScript代码
```javascript
$('.change_language').change(function (e) {
    e.preventDefault();
    $('#change_language_form').submit();
    return false

});
```

- 登录时记录用户选择的语言

在用户登录的后端视图中加入：
```python
request.session['_language']='zh-hans'
```


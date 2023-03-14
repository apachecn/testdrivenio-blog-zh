# 在 Django 中使用 AJAX

> 原文：<https://testdriven.io/blog/django-ajax-xhr/>

[AJAX](https://en.wikipedia.org/wiki/Ajax_(programming)) ，代表异步 JavaScript 和 XML，是一组在客户端使用的技术，用于异步发送和从服务器检索数据。

AJAX 允许我们对网页内容进行修改，而不需要用户重新加载整个页面。例如，这对于搜索栏中的自动完成或表单验证非常有用。如果使用得当，您可以提高网站的性能，减少服务器负载，并改善整体用户体验。

在本文中，我们将看看如何在 Django 中执行 GET、POST、PUT 和 DELETE AJAX 请求的例子。虽然重点将放在[获取 API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) 上，我们也将展示 [jQuery](https://jquery.com/) 的例子。

## 什么是 AJAX？

AJAX 是一种编程实践，它使用 [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) (XHR)对象与服务器异步通信并构建动态网页。尽管 AJAX 和 XMLHttpRequest 经常互换使用，但它们是[不同的](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API#differences_from_jquery)事物。

为了向 web 服务器发送数据和从 web 服务器接收数据，AJAX 使用以下步骤:

1.  创建一个 XMLHttpRequest 对象。
2.  使用 XMLHttpRequest 对象在客户端和服务器之间异步交换数据。
3.  使用 JavaScript 和 DOM 来处理数据。

通过使用 [ajax](https://api.jquery.com/jquery.ajax/) 方法，AJAX 可以与 jQuery 一起使用，但是本机 Fetch API 要好得多，因为它有一个干净的接口，不需要第三方库。

获取 API 的一般结构如下所示:

```
`fetch('http://some_url.com') .then(response  =>  response.json())  // converts the response to JSON .then(data  =>  { console.log(data); // do something (like update the DOM with the data) });` 
```

> 参考 [MDN 文档](https://developer.mozilla.org/)中的[使用 Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) 和[windoworworkerglobalscope . Fetch()](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch)获取更多示例以及`fetch`方法可用的完整选项。

## 什么时候应该使用 AJAX？

同样，AJAX 有助于提高站点的性能，同时降低服务器负载，改善整体用户体验。也就是说，它增加了应用程序的复杂性。正因为如此，除非你使用的是一个[单页面应用](https://en.wikipedia.org/wiki/Single-page_application)(SPA)——比如 React、Angular 或 Vue——你应该只在绝对必要的时候使用 AJAX。

您可能会考虑使用 AJAX 的一些例子:

1.  搜索自动完成
2.  表单验证
3.  表格排序和过滤
4.  验证码
5.  调查和民意测验

一般来说，如果内容需要根据用户交互进行大量更新，您可能希望使用 AJAX 来管理 web 页面的更新部分，而不是通过页面刷新来管理整个页面。

## CRUD 资源

本文中的例子可以应用于任何 CRUD 资源。示例 Django 项目使用 todos 作为它的资源:

| 方法 | 统一资源定位器 | 描述 |
| --- | --- | --- |
| 得到 | `/todos/` | 返回所有待办事项 |
| 邮政 | `/todos/` | 添加待办事项 |
| 放 | `/todos/<todo-id>/` | 更新待办事项 |
| 删除 | `/todos/<todo-id>/` | 删除待办事项 |

示例项目可以在 GitHub 上找到:

1.  获取版本:[https://github.com/testdrivenio/django-ajax-xhr](https://github.com/testdrivenio/django-ajax-xhr)
2.  jQuery 版本:[https://github.com/testdrivenio/django-ajax-xhr/tree/jquery](https://github.com/testdrivenio/django-ajax-xhr/tree/jquery)

## 获取请求

让我们从简单的获取数据的 GET 请求开始。

### 获取 API

示例:

```
`fetch(url,  { method:  "GET", headers:  { "X-Requested-With":  "XMLHttpRequest", } }) .then(response  =>  response.json()) .then(data  =>  { console.log(data); });` 
```

唯一必需的参数是您希望从中获取数据的资源的 URL。如果 URL 需要关键字参数或查询字符串，可以使用 Django 的`{% url %}`标签。

你注意到标题了吗？这是通知服务器您正在发送一个 AJAX 请求所必需的。

`fetch`返回包含 HTTP 响应的承诺。我们使用`.then`方法首先从响应中提取 JSON 格式的数据(通过`response.json()`)，然后访问返回的数据。在上面的例子中，我们只是在控制台中输出数据。

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/static/main . js # L19-L39](https://github.com/testdrivenio/django-ajax-xhr/blob/main/static/main.js#L19-L39)

### jQuery AJAX

等效的 jQuery 代码:

```
`$.ajax({ url:  url, type:  "GET", dataType:  "json", success:  (data)  =>  { console.log(data); }, error:  (error)  =>  { console.log(error); } });` 
```

[https://github . com/test rivieno/django-Ajax-xhr/blob/jquery/static/main . js # l19-l41](https://github.com/testdrivenio/django-ajax-xhr/blob/jquery/static/main.js#L19-L41)

### 姜戈观点

在 Django 方面，虽然有几种方法可以处理视图中的 AJAX 请求，但最简单的方法是使用基于函数的视图:

```
`from django.http import HttpResponseBadRequest, JsonResponse

from todos.models import Todo

def todos(request):
    # request.is_ajax() is deprecated since django 3.1
    is_ajax = request.headers.get('X-Requested-With') == 'XMLHttpRequest'

    if is_ajax:
        if request.method == 'GET':
            todos = list(Todo.objects.all().values())
            return JsonResponse({'context': todos})
        return JsonResponse({'status': 'Invalid request'}, status=400)
    else:
        return HttpResponseBadRequest('Invalid request')` 
```

在本例中，我们的资源是 todos。因此，在从数据库获取 todos 之前，我们验证了我们正在处理一个 AJAX 请求，并且请求方法是 GET。如果两者都为真，我们序列化数据并使用`JsonResponse`类发送响应。由于`QuerySet`对象不是 JSON 可序列化的(在本例中是 todos)，我们使用`values`方法将 QuerySet 作为一个字典返回，然后将其包装在`list`中。最终结果是一个字典列表。

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/todos/views . py # L13-L28](https://github.com/testdrivenio/django-ajax-xhr/blob/main/todos/views.py#L13-L28)

## 发布请求

接下来，让我们看看如何处理 POST 请求。

### 获取 API

示例:

```
`fetch(url,  { method:  "POST", credentials:  "same-origin", headers:  { "X-Requested-With":  "XMLHttpRequest", "X-CSRFToken":  getCookie("csrftoken"), }, body:  JSON.stringify({payload:  "data to send"}) }) .then(response  =>  response.json()) .then(data  =>  { console.log(data); });` 
```

我们需要指定如何在请求中发送凭证。

在上面的代码中，我们使用了`"same-origin"`(默认值)的值来指示浏览器，如果所请求的 URL 与 fetch 调用在同一个源上，就发送凭证。

在前端和后端托管在不同服务器上的情况下，您必须将`credentials`设置为`"include"`(它总是随每个请求发送凭证)，并在后端启用[跨源资源共享](http://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)。您可以使用 [django-cors-headers](https://pypi.org/project/django-cors-headers/) 包将 cors 报头添加到 django 应用程序的响应中。

> 想进一步了解如何处理同一域和跨域的 AJAX 请求吗？查看 [Django 基于会话的单页面应用认证](/blog/django-spa-auth/)文章。

这次我们在请求的`body`中向服务器发送数据。

注意`X-CSRFToken`标题。如果没有它，您将在终端中从服务器得到 403 禁止响应:

```
`Forbidden (CSRF token missing or incorrect.): /todos/` 
```

这是因为在发布请求时必须包含 CSRF 令牌，以防止[跨站请求伪造](https://en.wikipedia.org/wiki/Cross-site_request_forgery)攻击。

我们可以通过将每个`XMLHttpRequest`上的`X-CSRFToken`头设置为 CSRF 令牌的值来包含 CSRF 令牌。

Django 文档为我们提供了一个很好的[函数，允许我们获取令牌](https://docs.djangoproject.com/en/3.0/ref/csrf/#ajax)，从而简化了我们的生活:

```
`function  getCookie(name)  { let  cookieValue  =  null; if  (document.cookie  &&  document.cookie  !==  "")  { const  cookies  =  document.cookie.split(";"); for  (let  i  =  0;  i  <  cookies.length;  i++)  { const  cookie  =  cookies[i].trim(); // Does this cookie string begin with the name we want? if  (cookie.substring(0,  name.length  +  1)  ===  (name  +  "="))  { cookieValue  =  decodeURIComponent(cookie.substring(name.length  +  1)); break; } } } return  cookieValue; }` 
```

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/static/main . js # L42-L56](https://github.com/testdrivenio/django-ajax-xhr/blob/main/static/main.js#L42-L56)

### jQuery AJAX

带有 jQuery 的 AJAX POST 请求与 GET 请求非常相似:

```
`$.ajax({ url:  url, type:  "POST", dataType:  "json", data:  JSON.stringify({payload:  payload,}), headers:  { "X-Requested-With":  "XMLHttpRequest", "X-CSRFToken":  getCookie("csrftoken"),  // don't forget to include the 'getCookie' function }, success:  (data)  =>  { console.log(data); }, error:  (error)  =>  { console.log(error); } });` 
```

[https://github . com/test rivieno/django-Ajax-xhr/blob/jquery/static/main . js # l44-l61](https://github.com/testdrivenio/django-ajax-xhr/blob/jquery/static/main.js#L44-L61)

### 姜戈观点

在服务器端，视图需要从请求中获取 JSON 格式的数据，所以您需要使用`json`模块来加载它。

```
`import json

from django.http import HttpResponseBadRequest, JsonResponse

from todos.models import Todo

def todos(request):
    # request.is_ajax() is deprecated since django 3.1
    is_ajax = request.headers.get('X-Requested-With') == 'XMLHttpRequest'

    if is_ajax:
        if request.method == 'POST':
            data = json.load(request)
            todo = data.get('payload')
            Todo.objects.create(task=todo['task'], completed=todo['completed'])
            return JsonResponse({'status': 'Todo added!'})
        return JsonResponse({'status': 'Invalid request'}, status=400)
    else:
        return HttpResponseBadRequest('Invalid request')` 
```

在验证我们正在处理一个 AJAX 请求并且请求方法是 POST 之后，我们反序列化请求对象并提取有效负载对象。然后，我们创建了一个新的 todo 并发回了适当的响应。

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/todos/views . py # L13-L28](https://github.com/testdrivenio/django-ajax-xhr/blob/main/todos/views.py#L13-L28)

## 上传请求

### 获取 API

示例:

```
`fetch(url,  { method:  "PUT", credentials:  "same-origin", headers:  { "X-Requested-With":  "XMLHttpRequest", "X-CSRFToken":  getCookie("csrftoken"),  // don't forget to include the 'getCookie' function }, body:  JSON.stringify({payload:  "data to send"}) }) .then(response  =>  response.json()) .then(data  =>  { console.log(data); });` 
```

这应该类似于 POST 请求。唯一的区别是 URL 的形状:

1.  后- `/todos/`
2.  放- `/todos/<todo-id>/`

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/static/main . js # L59-L73](https://github.com/testdrivenio/django-ajax-xhr/blob/main/static/main.js#L59-L73)

### jQuery AJAX

等效 jQuery:

```
`$.ajax({ url:  url, type:  "PUT", dataType:  "json", data:  JSON.stringify({payload:  payload,}), headers:  { "X-Requested-With":  "XMLHttpRequest", "X-CSRFToken":  getCookie("csrftoken"),  // don't forget to include the 'getCookie' function }, success:  (data)  =>  { console.log(data); }, error:  (error)  =>  { console.log(error); } });` 
```

[https://github . com/test rivieno/django-Ajax-xhr/blob/jquery/static/main . js # l64-l81](https://github.com/testdrivenio/django-ajax-xhr/blob/jquery/static/main.js#L64-L81)

### 姜戈观点

示例:

```
`import json

from django.http import HttpResponseBadRequest, JsonResponse
from django.shortcuts import get_object_or_404

from todos.models import Todo

def todo(request, todoId):
    # request.is_ajax() is deprecated since django 3.1
    is_ajax = request.headers.get('X-Requested-With') == 'XMLHttpRequest'

    if is_ajax:
        todo = get_object_or_404(Todo, id=todoId)

        if request.method == 'PUT':
            data = json.load(request)
            updated_values = data.get('payload')

            todo.task = updated_values['task']
            todo.completed = updated_values['completed']
            todo.save()

            return JsonResponse({'status': 'Todo updated!'})
        return JsonResponse({'status': 'Invalid request'}, status=400)
    else:
        return HttpResponseBadRequest('Invalid request')` 
```

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/todos/views . py # L31-L53](https://github.com/testdrivenio/django-ajax-xhr/blob/main/todos/views.py#L31-L53)

## 删除请求

### 获取 API

示例:

```
`fetch(url,  { method:  "DELETE", credentials:  "same-origin", headers:  { "X-Requested-With":  "XMLHttpRequest", "X-CSRFToken":  getCookie("csrftoken"),  // don't forget to include the 'getCookie' function } }) .then(response  =>  response.json()) .then(data  =>  { console.log(data); });` 
```

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/static/main . js # L76-L89](https://github.com/testdrivenio/django-ajax-xhr/blob/main/static/main.js#L76-L89)

### jQuery AJAX

jQuery 代码:

```
`$.ajax({ url:  url, type:  "DELETE", dataType:  "json", headers:  { "X-Requested-With":  "XMLHttpRequest", "X-CSRFToken":  getCookie("csrftoken"), }, success:  (data)  =>  { console.log(data); }, error:  (error)  =>  { console.log(error); } });` 
```

[https://github . com/test rivieno/django-Ajax-xhr/blob/jquery/static/main . js # l84-l100](https://github.com/testdrivenio/django-ajax-xhr/blob/jquery/static/main.js#L84-L100)

### 姜戈观点

查看:

```
`from django.http import HttpResponseBadRequest, JsonResponse
from django.shortcuts import get_object_or_404

from todos.models import Todo

def todo(request, todoId):
    # request.is_ajax() is deprecated since django 3.1
    is_ajax = request.headers.get('X-Requested-With') == 'XMLHttpRequest'

    if is_ajax:
        todo = get_object_or_404(Todo, id=todoId)

        if request.method == 'DELETE':
            todo.delete()
            return JsonResponse({'status': 'Todo deleted!'})
        return JsonResponse({'status': 'Invalid request'}, status=400)
    else:
        return HttpResponseBadRequest('Invalid request')` 
```

[https://github . com/testdrivenio/django-Ajax-xhr/blob/main/todos/views . py # L31-L53](https://github.com/testdrivenio/django-ajax-xhr/blob/main/todos/views.py#L31-L53)

## 摘要

AJAX 允许我们执行异步请求来更改页面的某些部分，而不必重新加载整个页面。

在本文中，您看到了如何使用 Fetch API 和 jQuery 在 Django 中执行 GET、POST、PUT 和 DELETE AJAX 请求的详细示例。

示例项目可以在 GitHub 上找到:

1.  获取版本:[https://github.com/testdrivenio/django-ajax-xhr](https://github.com/testdrivenio/django-ajax-xhr)
2.  jQuery 版本:[https://github.com/testdrivenio/django-ajax-xhr/tree/jquery](https://github.com/testdrivenio/django-ajax-xhr/tree/jquery)
# 带有 Redis 的 Flask 中的服务器端会话

> 原文：<https://testdriven.io/blog/flask-server-side-sessions/>

本文通过 [Flask-Session](https://flask-session.readthedocs.io/en/latest/) 和 [Redis](https://redis.io) 来研究如何在 Flask 中利用服务器端会话。

本文是关于如何在 Flask 中使用会话的两部分系列文章的一部分:

1.  客户端:Flask 中的[个会话](/blog/flask-sessions/)
2.  服务器端:[带 Redis 的 Flask 中的服务器端会话](/blog/flask-server-side-sessions) ( *本文！*)

> 本文假设您之前有使用 Flask 的经验。如果您有兴趣了解有关 Flask 的更多信息，请查看我关于如何构建、测试和部署 Flask 应用程序的课程:
> 
> [用 Python 和 Flask 开发 Web 应用](/courses/learn-flask/)

## 会议

由于 HTTP 是一个无状态协议，每个请求都不知道以前执行过的任何请求:

![Flask Requests Responses Diagram](img/56f63b408165cdd0376f4c6b50024869.png)

虽然这极大地简化了客户端/服务器通信，但是当用户与应用程序本身交互时，web 应用程序通常需要一种方法来存储每个请求之间的数据。

例如，在电子商务网站上，您通常会将用户添加到购物车中的商品存储到数据库中，这样一旦他们完成购物，就可以查看他们的购物车来购买这些商品。不过，这个在数据库中存储项目的工作流只对经过身份验证的用户有效。因此，您需要一种方法在请求之间为未经身份验证的用户存储特定于用户的数据。

这就是会话发挥作用的地方。

一个[会话](https://en.wikipedia.org/wiki/Session_(computer_science))用于在用户与 web 应用程序交互时，跨不同请求存储与用户相关的信息。因此，在上面的例子中，购物车商品将被添加到用户的会话中。

为会话存储的数据应被视为临时数据，因为会话最终会过期。为了永久存储数据，你需要利用数据库。

> 计算机存储在这里是一个很好的类比:计算机上的临时项目存储在 RAM(随机存取存储器)中，很像会话，而永久项目存储在硬盘上，很像数据库。

存储在会话中的数据示例:

*   用户购物车中的商品
*   无论用户是否登录
*   首选项(语言、货币、明暗模式)

存储在数据库中的数据示例:

*   用户凭据(电子邮件、用户名、哈希密码、电子邮件确认布尔值)
*   用户输入的数据(股票数据、食谱、博客文章)

在 Flask 中，您可以在会话期间存储特定于用户的信息。保存数据以供整个会话使用允许 web 应用程序在多次请求中保持数据的持久性，例如，当用户访问 web 应用程序中的不同页面时。

## 烧瓶中的会话

web 开发中通常使用两种类型的会话:

1.  **客户端** -会话存储在客户端的浏览器 cookies 中
2.  **服务器端** -会话存储在服务器端(通常会创建一个会话标识符并存储在客户端的浏览器 cookies 中)

Flask 使用客户端方法作为会话的内置解决方案。

> 对客户端会话感到好奇？查看 Flask 文章中的[部分。](/blog/flask-sessions/)

### 服务器端会话

服务器端会话将与服务器上的会话相关联的数据存储在特定的数据存储解决方案中。来自 Flask 的每个响应中都包含一个加密签名的 cookie，用于指定会话标识符。这个 cookie 在对 Flask 应用程序的下一个请求中返回，然后用于从服务器端存储加载会话数据。

**优点**:

*   敏感数据存储在服务器上，而不是 web 浏览器中
*   您可以存储任意多的会话数据，而不必担心 cookie 的大小
*   Flask 应用程序可以很容易地终止会话

**缺点**:

*   难以设置和扩展
*   由于必须管理会话状态，增加了复杂性

## 烧瓶会议

Flask-Session 是 Flask 的扩展，支持服务器端会话。它支持在服务器端存储会话数据的各种解决方案:

1.  雷迪斯
2.  Memcached
3.  sqllcemy(SQL 语法)
4.  MongoDB

在本文中，我们将使用 Redis，这是一种内存中的数据结构存储，因为它的读/写速度很快，并且易于设置。

> 参考 Flask-Session 文档的[配置](https://flask-session.readthedocs.io/en/latest/#configuration)部分，了解如何配置其他数据存储解决方案。

Flask-Session 使用 Flask 的[会话接口](https://flask.palletsprojects.com/en/1.1.x/api/#session-interface)，它提供了一种简单的方法来替换 Flask 的内置会话实现，因此您可以像通常使用内置客户端会话实现一样继续使用`session`对象。

## 服务器端会话示例

下面的 *app.py* 文件说明了如何通过 Flask-Session 在 Flask 中使用服务器端会话:

```py
`from datetime import timedelta

import redis
from flask import Flask, render_template_string, request, session, redirect, url_for
from flask_session import Session

# Create the Flask application
app = Flask(__name__)

# Details on the Secret Key: https://flask.palletsprojects.com/en/1.1.x/config/#SECRET_KEY
# NOTE: The secret key is used to cryptographically-sign the cookies used for storing
#       the session identifier.
app.secret_key = 'BAD_SECRET_KEY'

# Configure Redis for storing the session data on the server-side
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_PERMANENT'] = False
app.config['SESSION_USE_SIGNER'] = True
app.config['SESSION_REDIS'] = redis.from_url('redis://localhost:6379')

# Create and initialize the Flask-Session object AFTER `app` has been configured
server_session = Session(app)

@app.route('/set_email', methods=['GET', 'POST'])
def set_email():
    if request.method == 'POST':
        # Save the form data to the session object
        session['email'] = request.form['email_address']
        return redirect(url_for('get_email'))

    return """
 <form method="post">
 <label for="email">Enter your email address:</label>
 <input type="email" id="email" name="email_address" required />
 <button type="submit">Submit</button
 </form>
 """

@app.route('/get_email')
def get_email():
    return render_template_string("""
 {% if session['email'] %}
 <h1>Welcome {{ session['email'] }}!</h1>
 {% else %}
 <h1>Welcome! Please enter your email <a href="{{ url_for('set_email') }}">here.</a></h1>
 {% endif %}
 """)

@app.route('/delete_email')
def delete_email():
    # Clear the email stored in the session object
    session.pop('email', default=None)
    return '<h1>Session deleted!</h1>'

if __name__ == '__main__':
    app.run()` 
```

要运行此示例，首先创建并激活一个新的虚拟环境:

```py
`$ mkdir flask-server-side-sessions
$ cd flask-server-side-sessions
$ python3 -m venv venv
$ source venv/bin/activate` 
```

[安装](https://redis.io/download)并运行 Redis。

> 启动和运行 Redis 的最快方法是使用 Docker:
> 
> ```py
> $ docker run --name some-redis -d -p 6379:6379 redis 
> ```
> 
> 如果您不是 Docker 用户，请查看以下资源:

安装烧瓶、烧瓶会话和 [redis-py](https://redis-py.readthedocs.io/en/stable/) :

```py
`(venv)$ pip install Flask Flask-Session redis` 
```

> 由于我们使用 Redis 作为会话数据存储，redis-py 是[必需的](https://flask-session.readthedocs.io/en/latest/#redissessioninterface)。

将上述代码保存到一个 *app.py* 文件中。然后，启动 Flask 开发服务器:

```py
`(venv)$ export FLASK_APP=app.py
(venv)$ export FLASK_ENV=development
(venv)$ python -m flask run` 
```

现在，使用您最喜欢的网络浏览器导航到[http://localhost:5000/get _ email](http://localhost:5000/get_email):

![Flask Session Example - Step 1](img/52f025444c6ab8053fa06d8002e22bc1.png)

### 配置

创建 Flask 应用程序(`app`)后，需要指定密钥:

```py
`# Details on the Secret Key: https://flask.palletsprojects.com/en/1.1.x/config/#SECRET_KEY
# NOTE: The secret key is used to cryptographically-sign the cookies used for storing
#       the session identifier.
app.secret_key = 'BAD_SECRET_KEY'` 
```

秘密密钥用于对存储会话标识符的 cookies 进行加密签名。

接下来，需要将 Redis 配置为服务器端会话数据的存储解决方案:

```py
`# Configure Redis for storing the session data on the server-side
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_PERMANENT'] = False
app.config['SESSION_USE_SIGNER'] = True
app.config['SESSION_REDIS'] = redis.from_url('redis://localhost:6379')

# Create and initialize the Flask-Session object AFTER `app` has been configured
server_session = Session(app)` 
```

配置变量:

*   `SESSION_TYPE` -指定使用哪种类型的会话接口
*   `SESSION_PERMANENT` -表示是否使用永久会话(默认为`True`)
*   `SESSION_USE_SIGNER` -指示是否签署会话 cookie 标识符(默认为`False`)
*   `SESSION_REDIS` -指定 Redis 实例(默认连接到`127.0.0.1:6379`)

> 有关所有可用配置变量的详细信息，请参考 Flask-Session 文档的[配置](https://flask-session.readthedocs.io/en/latest/#configuration)部分。

### 设置会话数据

在本例中，`set_email`视图函数在提交表单时处理电子邮件:

```py
`@app.route('/set_email', methods=['GET', 'POST'])
def set_email():
    if request.method == 'POST':
        # Save the form data to the session object
        session['email'] = request.form['email_address']
        return redirect(url_for('get_email'))

    return """
 <form method="post">
 <label for="email">Enter your email address:</label>
 <input type="email" id="email" name="email_address" required />
 <button type="submit">Submit</button
 </form>
 """` 
```

这个视图函数支持 GET 和 POST HTTP 方法。当使用 GET 方法时，会返回一个 HTML 表单供您输入电子邮件地址:

![Flask Session Example - Step 2](img/a624d482a106f15904ea5d7fcec62d92.png)

当您使用您的电子邮件地址提交表单时(通过 POST 方法)，电子邮件将保存在`session`对象中:

```py
`# Save the form data to the session object
session['email'] = request.form['email_address']` 
```

请在[http://localhost:5000/set _ email](http://localhost:5000/set_email)中输入您的电子邮件，然后提交表单。

### 访问会话数据

当电子邮件没有存储在会话中时，`get_email`视图功能利用 Jinja 模板引擎显示存储在`session`对象中的电子邮件地址或者到`set_email()`视图功能的链接:

```py
`@app.route('/get_email')
def get_email():
    return render_template_string("""
 {% if session['email'] %}
 <h1>Welcome {{ session['email'] }}!</h1>
 {% else %}
 <h1>Welcome! Please enter your email <a href="{{ url_for('set_email') }}">here.</a></h1>
 {% endif %}
 """)` 
```

`session`对象可以在模板文件中使用！

输入您的电子邮件地址后，当您导航到[http://localhost:5000/get _ email](http://localhost:5000/get_email)URL 时，您的电子邮件将显示:

![Flask Session Example - Step 3](img/a8924dd9ca51267c10e198f9790afa5f.png)

### 删除会话数据

存储在`session`对象中的电子邮件地址可以通过`delete_email`查看功能删除:

```py
`@app.route('/delete_email')
def delete_email():
    # Clear the email stored in the session object
    session.pop('email', default=None)
    return '<h1>Session deleted!</h1>'` 
```

这个视图函数从`session`对象中获取`email`元素。`pop`方法将返回弹出的值，因此在`session`对象中没有定义元素的情况下，提供默认值是一个好的做法。

当您导航到“[http://localhost:5000/delete _ email](http://localhost:5000/delete_email)URL”时，您将看到:

![Flask Session Example - Step 4](img/a14474f5690c7806fa20395a1b1f05f9.png)

由于电子邮件地址不再存储在`session`对象中，当您导航到[http://localhost:5000/get _ email](http://localhost:5000/get_email)URL 时，将再次要求您输入您的电子邮件地址:

![Flask Session Example - Step 5](img/d7f226bd7a33a5aab186cb5bf97f2c4d.png)

### 会话唯一性

为了演示每个用户的会话数据是如何不同的，请在[http://localhost:5000/set _ email](http://localhost:5000/set_email)再次输入您的电子邮件地址。然后，在不同的浏览器(或当前浏览器中的私人/匿名窗口)中，导航至[http://localhost:5000/set _ email](http://localhost:5000/set_email)并输入不同的电子邮件地址。在你被重定向到[http://localhost:5000/get _ email](http://localhost:5000/get_email)之后，你希望看到什么？

由于使用了不同的网络浏览器，这被认为是 Flask 应用程序的不同用户。因此，将有一个唯一的`session`用于该用户。

要更详细地了解这一点，您可以从计算机上的两个不同的 web 浏览器访问 Flask 应用程序后，检查 Redis 数据库中存储的内容:

```py
`$ redis-cli

127.0.0.1:6379> KEYS *
1) "session:8a77d85b-7ed9-4961-958a-510240bcbac4"
2) "session:5ce4b8e2-a2b5-43e4-a0f9-7fa465b7bb0c"

127.0.0.1:6379> exit
$` 
```

Redis 中存储了两个不同的会话，对应于用于访问 Flask 应用程序的两个不同的 web 浏览器:

1.  `8a77d85b-7ed9-4961-958a-510240bcbac4`来自火狐
2.  `session:5ce4b8e2-a2b5-43e4-a0f9-7fa465b7bb0c`来自 Chrome

## 结论

本文展示了如何使用 Flask-Session 和 Redis 在 Flask 中实现服务器端会话。

如果你想了解更多关于 Flask 中的会话，一定要看看我的课程- [用 Python 和 Flask 开发 Web 应用](/courses/learn-flask/)。
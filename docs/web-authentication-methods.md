# 比较的 Web 身份验证方法

> 原文：<https://testdriven.io/blog/web-authentication-methods/>

在本文中，我们将从 Python web 开发人员的角度来看最常用的处理 web 身份验证的方法。

> 虽然代码示例和资源是为 Python 开发人员准备的，但是每种身份验证方法的实际描述适用于所有 web 开发人员。

## 身份验证与授权

身份认证是对试图访问受限系统的用户或设备的凭据进行验证的过程。同时，授权是验证用户或设备是否被允许在给定系统上执行某些任务的过程。

简而言之:

1.  认证:*你是谁？*
2.  授权:*你能做什么？*

认证先于授权。也就是说，在根据授权级别授予用户访问资源的权限之前，用户必须是有效的。认证用户最常见的方式是通过`username`和`password`。一旦通过认证，不同的角色如`admin`、`moderator`等。分配给他们，这将授予他们系统的特权。

接下来，让我们来看看认证用户的不同方法。

## HTTP 基本身份验证

HTTP 协议中内置的基本身份验证是最基本的身份验证形式。有了它，登录凭据将在每个请求的请求头中发送:

```py
`"Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" your-website.com` 
```

用户名和密码不加密。相反，用户名和密码使用一个`:`符号连接在一起，形成一个字符串:`username:password`。然后，使用 base64 对该字符串进行编码。

```py
`>>> import base64
>>>
>>> auth = "username:password"
>>> auth_bytes = auth.encode('ascii') # convert to bytes
>>> auth_bytes
b'username:password'
>>>
>>> encoded = base64.b64encode(auth_bytes) # base64 encode
>>> encoded
b'dXNlcm5hbWU6cGFzc3dvcmQ='
>>> base64.b64decode(encoded) # base64 decode
b'username:password'` 
```

这个方法是无状态的，所以客户端必须为每个请求提供凭证。它适用于 API 调用以及不需要持久会话的简单身份验证工作流。

### 流动

1.  未经身份验证的客户端请求受限资源
2.  HTTP 401 Unauthorized 返回一个值为`Basic`的头`WWW-Authenticate`。
3.  `WWW-Authenticate: Basic`标题使浏览器显示用户名和密码提示
4.  输入您的凭证后，它们会在每个请求的标头中发送:`Authorization: Basic dcdvcmQ=`

![http basic auth workflow](img/e88c0ee734e80bfc58aab5d98d866717.png)

### 赞成的意见

*   因为没有太多的操作在进行，所以使用这种方法认证会更快。
*   容易实现。
*   所有主流浏览器都支持。

### 骗局

*   Base64 不同于加密。这只是表示数据的另一种方式。base64 编码的字符串很容易被解码，因为它是以纯文本形式发送的。这种糟糕的安全特性需要多种类型的攻击。因此，HTTPS/SSL 是绝对必要的。
*   每个请求都必须发送凭据。
*   用户只能通过用无效凭据重写凭据来注销。

### 包装

### 密码

使用 [Flask-HTTP](https://github.com/miguelgrinberg/Flask-HTTPAuth) 包可以在 Flask 中轻松完成基本的 HTTP 认证。

```py
`from flask import Flask
from flask_httpauth import HTTPBasicAuth
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
auth = HTTPBasicAuth()

users = {
    "username": generate_password_hash("password"),
}

@auth.verify_password
def verify_password(username, password):
    if username in users and check_password_hash(users.get("username"), password):
        return username

@app.route("/")
@auth.login_required
def index():
    return f"You have successfully logged in, {auth.current_user()}"

if __name__ == "__main__":
    app.run()` 
```

### 资源

## HTTP 摘要认证

HTTP 摘要式身份验证(或摘要式访问身份验证)是 HTTP 基本身份验证的一种更安全的形式。主要区别在于密码是以 MD5 散列形式发送的，而不是以明文形式，因此它比基本 Auth 更安全。

### 流动

1.  未经身份验证的客户端请求受限资源
2.  服务器生成一个名为 nonce 的随机值，并发回一个 HTTP 401 未授权状态，带有一个带有值为`Digest`的`WWW-Authenticate`报头以及 nonce: `WWW-Authenticate: Digest nonce="44f0437004157342f50f935906ad46fc"`
3.  `WWW-Authenticate: Basic`标题使浏览器显示用户名和密码提示
4.  在输入您的凭证之后，密码被散列，然后在每个请求的报头中与随机数一起发送:`Authorization: Digest username="username", nonce="16e30069e45a7f47b4e2606aeeb7ab62", response="89549b93e13d438cd0946c6d93321c52"`
5.  有了用户名，服务器就获得了密码，将密码和随机数一起散列，然后验证散列是否相同

![http basic auth workflow](img/9037d80356254d882ab2a1cfa1aff8c1.png)

### 赞成的意见

*   比基本身份验证更安全，因为密码不是以纯文本形式发送的。
*   容易实现。
*   所有主流浏览器都支持。

### 骗局

*   每个请求都必须发送凭据。
*   用户只能通过使用无效凭据重写凭据来注销。
*   与基本 auth 相比，密码在服务器上不太安全，因为 bcrypt 不能使用。
*   易受中间人攻击。

### 包装

### 密码

[Flask-HTTP](https://github.com/miguelgrinberg/Flask-HTTPAuth) 包也支持摘要 HTTP 认证。

```py
`from flask import Flask
from flask_httpauth import HTTPDigestAuth

app = Flask(__name__)
app.config["SECRET_KEY"] = "change me"
auth = HTTPDigestAuth()

users = {
    "username": "password"
}

@auth.get_password
def get_user(username):
    if username in users:
        return users.get(username)

@app.route("/")
@auth.login_required
def index():
    return f"You have successfully logged in, {auth.current_user()}"

if __name__ == "__main__":
    app.run()` 
```

### 资源

## 基于会话的身份验证

使用基于会话的身份验证(或会话 cookie 身份验证或基于 cookie 的身份验证)，用户的状态存储在服务器上。它不要求用户为每个请求提供用户名或密码。相反，在登录后，服务器会验证凭据。如果有效，它将生成一个会话，将其存储在会话存储中，然后将会话 id 发送回浏览器。浏览器将会话 ID 存储为 cookie，每当向服务器发出请求时，就会发送该 cookie。

基于会话的身份验证是有状态的。每当客户端请求服务器时，服务器必须在内存中定位会话，以便将会话 ID 绑定到相关用户。

### 流动

![http session auth workflow](img/6dd47a95130c6d8af0ecd7d86e3842e0.png)

### 赞成的意见

*   更快的后续登录，因为不需要凭据。
*   改善了用户体验。
*   相当容易实现。许多框架(比如 Django)都提供了开箱即用的特性。

### 骗局

*   它是有状态的。服务器在服务器端跟踪每个会话。用于存储用户会话信息的会话存储需要跨多个服务共享，以实现身份验证。正因为如此，它不适合 RESTful 服务，因为 REST 是一种无状态协议。
*   Cookies 会随每个请求一起发送，即使它不需要身份验证。
*   易受 CSRF 攻击。点击这里阅读更多关于 CSRF 以及如何预防它的信息。

### 包装

### 密码

[Flask-Login](https://flask-login.readthedocs.io/) 非常适合基于会话的认证。这个包负责登录、注销，并且可以在一段时间内记住用户。

```py
`from flask import Flask, request
from flask_login import (
    LoginManager,
    UserMixin,
    current_user,
    login_required,
    login_user,
)
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
app.config.update(
    SECRET_KEY="change_this_key",
)

login_manager = LoginManager()
login_manager.init_app(app)

users = {
    "username": generate_password_hash("password"),
}

class User(UserMixin):
    ...

@login_manager.user_loader
def user_loader(username: str):
    if username in users:
        user_model = User()
        user_model.id = username
        return user_model
    return None

@app.route("/login", methods=["POST"])
def login_page():
    data = request.get_json()
    username = data.get("username")
    password = data.get("password")

    if username in users:
        if check_password_hash(users.get(username), password):
            user_model = User()
            user_model.id = username
            login_user(user_model)
        else:
            return "Wrong credentials"
    return "logged in"

@app.route("/")
@login_required
def protected():
    return f"Current user: {current_user.id}"

if __name__ == "__main__":
    app.run()` 
```

### 资源

## 基于令牌的认证

这种方法使用令牌而不是 cookies 来验证用户。用户使用有效凭据进行身份验证，服务器返回一个签名令牌。该令牌可用于后续请求。

最常用的令牌是一个 [JSON Web 令牌](https://jwt.io/) (JWT)。JWT 由三部分组成:

*   标头(包括令牌类型和使用的哈希算法)
*   有效负载(包括声明，即关于主题的陈述)
*   签名(用于验证消息没有被更改)

所有这三个都是 base64 编码的，使用一个`.`连接起来并进行哈希处理。因为它们是编码的，任何人都可以解码和阅读信息。但是只有真实的用户才能产生有效的签名令牌。令牌使用签名进行身份验证，签名是用私钥签署的。

> JSON Web Token (JWT)是一种紧凑的、URL 安全的方式，用于表示在双方之间传输的声明。JWT 中的声明被编码为 JSON 对象，该对象被用作 JSON Web 签名(JWS)结构的有效载荷，或者被用作 JSON Web 加密(JWE)结构的明文，从而能够对声明进行数字签名或使用消息认证码(MAC)进行完整性保护和/或加密。- [IETF](https://tools.ietf.org/html/rfc7519)

令牌不需要保存在服务器端。他们可以通过他们的签名来验证。最近，由于 RESTful APIs 和单页面应用程序(spa)的兴起，令牌的采用有所增加。

### 流动

![token auth workflow](img/b5ad315e085457dc4fda914fddcc7452.png)

### 赞成的意见

*   它是无国籍的。服务器不需要存储令牌，因为它可以使用签名进行验证。这使得请求更快，因为不需要数据库查找。
*   适合微服务架构，其中多个服务需要身份验证。我们在每一端需要配置的只是如何处理令牌和令牌秘密。

### 骗局

*   根据令牌在客户端的保存方式，它可能会导致 XSS(通过本地存储)或 CSRF(通过 cookies)攻击。
*   无法删除令牌。它们只能过期。这意味着，如果令牌泄露，攻击者可以滥用它，直到到期。因此，将令牌到期时间设置得很短很重要，比如 15 分钟。
*   刷新令牌需要设置为在到期时自动颁发令牌。
*   删除令牌的一种方法是创建一个用于将令牌列入黑名单的数据库。这给微服务架构增加了额外的开销，并引入了状态。

### 包装

### 密码

[烧瓶 JWTs 扩展](https://github.com/vimalloc/flask-jwt-extended)包为处理 JWT 提供了许多可能性。

```py
`from flask import Flask, request, jsonify
from flask_jwt_extended import (
    JWTManager,
    jwt_required,
    create_access_token,
    get_jwt_identity,
)
from werkzeug.security import check_password_hash, generate_password_hash

app = Flask(__name__)
app.config.update(
    JWT_SECRET_KEY="please_change_this",
)

jwt = JWTManager(app)

users = {
    "username": generate_password_hash("password"),
}

@app.route("/login", methods=["POST"])
def login_page():
    username = request.json.get("username")
    password = request.json.get("password")

    if username in users:
        if check_password_hash(users.get(username), password):
            access_token = create_access_token(identity=username)
            return jsonify(access_token=access_token), 200

    return "Wrong credentials", 400

@app.route("/")
@jwt_required
def protected():
    return jsonify(logged_in_as=get_jwt_identity()), 200

if __name__ == "__main__":
    app.run()` 
```

### 资源

## 一次性密码

一次性密码(OTP)通常用作身份验证的确认。OTP 是随机生成的代码，可用于验证用户是否是他们声称的那个人。它通常在利用双因素身份验证的应用程序的用户凭据得到验证后使用。

要使用 OTP，必须存在可信系统。这个可信系统可以是经过验证的电子邮件或手机号码。

现代 OTP 是无状态的。可以使用多种方法来验证它们。虽然有几种不同类型的动态口令，但基于时间的动态口令(TOTPs)无疑是最常见的类型。一旦生成，它们会在一段时间后过期。

因为您获得了额外的安全层，所以建议将 OTP 用于涉及高度敏感数据的应用程序，如网上银行和其他金融服务。

### 流动

实现 OTP 的传统方式是:

*   客户端发送用户名和密码
*   在凭证验证之后，服务器生成随机代码，将其存储在服务器端，并将代码发送到可信系统
*   用户在可信系统上获取代码，并将其输入回 web 应用程序
*   服务器根据存储的代码验证该代码，并相应地授权访问

TOTPs 如何工作:

*   客户端发送用户名和密码
*   在凭证验证之后，服务器使用随机生成的种子生成随机代码，将种子存储在服务器端，并将代码发送到可信系统
*   用户在可信系统上获取代码，并将其输入回 web 应用程序
*   服务器根据存储的种子验证代码，确保它没有过期，并相应地授予访问权限

像 [Google Authenticator](https://en.wikipedia.org/wiki/Google_Authenticator) 、[微软 Authenticator](https://www.microsoft.com/en-us/account/authenticator) 、 [FreeOTP](https://en.wikipedia.org/wiki/FreeOTP) 这样的 OTP 代理是如何工作的:

*   注册双因素身份验证(2FA)后，服务器会生成一个随机种子值，并以唯一 QR 码的形式将该种子发送给用户
*   用户使用他们的 2FA 应用程序扫描 QR 码来验证可信设备
*   每当需要 OTP 时，用户就在他们的设备上检查代码，并在 web 应用程序上输入代码
*   服务器验证代码并相应地授权访问

### 赞成的意见

*   增加一层额外的保护。
*   不存在被盗密码可用于也实现 OTP 的多个站点或服务的危险。

### 骗局

*   您需要存储用于生成 OTP 的种子。
*   如果您丢失了恢复代码，像 Google Authenticator 这样的 OTP 代理很难再次设置。
*   当可信设备不可用时(电池没电、网络错误等)，问题就会出现。).因此，通常需要备份设备，这会增加额外的攻击媒介。

### 包装

### 密码

PyOTP 包提供基于时间和基于计数器的 OTP。

```py
`from time import sleep

import pyotp

if __name__ == "__main__":
    otp = pyotp.TOTP(pyotp.random_base32())
    code = otp.now()
    print(f"OTP generated: {code}")
    print(f"Verify OTP: {otp.verify(code)}")
    sleep(30)
    print(f"Verify after 30s: {otp.verify(code)}")` 
```

示例:

```py
`OTP generated: 474771
Verify OTP: True
Verify after 30s: False` 
```

### 资源

## OAuth 和 OpenID

OAuth/OAuth2 和 OpenID 分别是授权和认证的流行形式。它们用于实现[社交登录](https://en.wikipedia.org/wiki/Social_login)，这是一种形式的[单点登录](https://en.wikipedia.org/wiki/Single_sign-on) (SSO)，使用来自脸书、推特或谷歌等社交网络服务的现有信息登录第三方网站，而不是专门为该网站创建新的登录帐户。

当您需要高度安全的身份验证时，可以使用这种类型的身份验证和授权。其中一些提供商有足够的资源投资于身份验证本身。利用这种久经考验的身份验证系统最终可以使您的应用程序更加安全。

这种方法通常与基于会话的身份验证结合使用。

### 流动

您访问了一个要求您登录的网站。您导航到登录页面，看到一个名为“使用 Google 登录”的按钮。你点击按钮，它会把你带到谷歌登录页面。通过身份验证后，您会被重定向回自动让您登录的网站。这是一个使用 OpenID 进行身份验证的例子。它允许您使用现有帐户(通过 OpenID 提供者)进行身份验证，而无需创建新帐户。

最著名的 OpenID 提供商是谷歌、脸书、Twitter 和 GitHub。

登录后，您可以导航到网站内的下载服务，让您直接将大文件下载到 Google Drive。网站是如何访问你的谷歌硬盘的？这就是 OAuth 发挥作用的地方。您可以授予访问其他网站上的资源的权限。在这种情况下，对 Google Drive 进行写访问。

### 赞成的意见

*   提高安全性。
*   由于无需创建和记住用户名或密码，登录流程更加简单快捷。
*   在安全漏洞的情况下，不会发生第三方损害，因为认证是无密码的。

### 骗局

*   您的应用程序现在依赖于另一个应用程序，不受您的控制。如果 OpenID 系统关闭，用户将无法登录。
*   人们往往会忽略 OAuth 应用程序请求的权限。
*   在您配置的 OpenID 提供商上没有帐户的用户将无法访问您的应用程序。最好的方法是两者都实现——即用户名、密码和 OpenID——并让用户选择。

### 包装

希望实现社交登录？

想运行自己的 OAuth 或 OpenID 服务吗？

### 密码

你可以用 [Flask-Dance](https://flask-dance.readthedocs.io/) 实现 GitHub social auth。

```py
`from flask import Flask, url_for, redirect
from flask_dance.contrib.github import make_github_blueprint, github

app = Flask(__name__)
app.secret_key = "change me"
app.config["GITHUB_OAUTH_CLIENT_ID"] = "1aaf1bf583d5e425dc8b"
app.config["GITHUB_OAUTH_CLIENT_SECRET"] = "dee0c5bc7e0acfb71791b21ca459c008be992d7c"

github_blueprint = make_github_blueprint()
app.register_blueprint(github_blueprint, url_prefix="/login")

@app.route("/")
def index():
    if not github.authorized:
        return redirect(url_for("github.login"))
    resp = github.get("/user")
    assert resp.ok
    return f"You have successfully logged in, {resp.json()['login']}"

if __name__ == "__main__":
    app.run()` 
```

### 资源

## 结论

在本文中，我们研究了许多不同的 web 身份验证方法，它们都有各自的优缺点。

你应该在什么时候使用它们？看情况。基本经验法则:

1.  对于利用服务器端模板的 web 应用程序，通过用户名和密码进行基于会话的身份验证通常是最合适的。您也可以添加 OAuth 和 OpenID。
2.  对于 RESTful APIs，基于令牌的认证是推荐的方法，因为它是无状态的。
3.  如果您必须处理高度敏感的数据，您可能希望将 OTP 添加到您的授权流中。

最后，请记住，所示的示例只是触及了表面。生产使用需要进一步配置。
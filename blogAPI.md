## API设计

模仿 Github，设计一个博客网站的 API

#### 概括

[TOC]

### Current version

默认下所有请求 `https://api.blog.com` 都会响应 **v1** 版本REST API.

~~~
Accept: application/vnd.blog.v1+json
~~~

### Schema

所有API的访问时通过HTTPS访问`https://api.blog.com`实现的，而所有的数据都是通过JSON来传送和接受的。

~~~
curl -i https://api.blog.com/username
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 12 Oct 2012 23:33:14 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Status: 200 OK
ETag: "a00049ba79152d03380c34652f2cb612"
X-GitHub-Media-Type: github.v3
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4987
X-RateLimit-Reset: 1350085394
Content-Length: 5
Cache-Control: max-age=0, private, must-revalidate
X-Content-Type-Options: nosniff
~~~

- 空白字段用`null`表示。
- 所有的时间戳以ISO 8601格式返回：

~~~
YYYY-MM-DDTHH:MM:SSZ
~~~

### 简要显示

当获取一系列的资源时，响应包括该资源的属性子集，有些属性会排除在外。

例子：当获取一系列的用户博客的时候，会获得该用户每篇文章的摘要。如获取用户名为`username`用户的博客所有的文章：

~~~
GET /username/article
~~~

### 详细显示

当获取具体单个资源时，响应通常包括该资源的所有属性。(认证会影响能访问的资源，如个人文章等)

例子：当获取用户`username`的某个博客文章`title`时，将得到具体的文章表示：

~~~
GET /username/article/title
~~~

### 认证

有两种方法通过Blog API v1进行认证，请求认证在某些地方会返回`404 Not Found`而不是`403 Forbidden`，认证是为了防止私人博客被未经授权的用户查看。

#### 基本认证

~~~
curl -u "username" https://api.blog.com
~~~

#### 认证失败

密码或用户名错误将返回 401 Unauthorized，不准用户进行查看。

~~~
curl -i https://api.blog.com -u invalid_username：invalid_password
HTTP/1.1 401 Unauthorized
{
  "message": "Bad credentials",
  "documentation_url": "https://blog.com/v3"
}
~~~

监测到多时间内有多次错误认证后，API会暂时拒绝用户的认证请求（包括正确的认证），并返回`403 Forbidden`响应。

~~~
curl -i https://api.blog.com -u valid_username:valid_password
HTTP/1.1 403 Forbidden
{
  "message": "Maximum number of login attempts exceeded. Please try again later.",
  "documentation_url": "https://blog.com/v3"
}
~~~

### 参数

许多的API方法都接受多个可选择的参数，对于`GET`请求，任何不为路径段的参数都可以作为HTTP查询字符串参数传递：

~~~
curl -i "https://api.blog.com/username/title?state=closed"
~~~

在这个例子中，`username`、`title`作为`:username`、`:ariticle`的路径上的参数，`:state`在查询字符串时传递路径。

对于`POST`，`PATCH`，`PUT`，`DELETE`请求，不包括在URL中的参数应该编码为带有`application/json`Content-Type的`JSON`格式。

~~~
curl -i -u username -d '{"scopes":["public_blog"]}' https://api.blog.com/authorizations
~~~

### 根端点

可以通过向根端点发起`GET`请求来获得所有REST API支持的端点类别。

~~~
curl https://api.blog.com
~~~

### 客户端错误

在调用API时，有三种可能发生的客户端错误请求体：

1. 发送无效的JSON会得到`400 Bad Request`错误响应

   ~~~
   HTTP/1.1 400 Bad Request
   Content-Length: 35
   
   {"message":"Problems parsing JSON"}
   ~~~

2. 发送错误类型的JSON值会得到`400 Bad Request`错误响应

   ~~~
   HTTP/1.1 400 Bad Request
   Content-Length: 40
   
   {"message":"Body should be a JSON object"}
   ~~~

3. 发送无效的字段会得到`422 Unprocessable Entity`错误响应。

   ~~~
   HTTP/1.1 422 Unprocessable Entity
   Content-Length: 149
   
   {
     "message": "Validation Failed",
     "errors": [
       {
         "resource": "Issue",
         "field": "title",
         "code": "missing_field"
       }
     ]
   }
   ~~~

   所有错误对象都具有资源和字段属性，以便客户端可以知道问题所在。还有一个错误代码让你知道字段有什么问题，以下是可能的验证错误代码：

   |     错误名字     |                         Description                          |
   | :--------------: | :----------------------------------------------------------: |
   |    `missing`     |            This means a resource does not exist.             |
   | `missing_field`  | This means a required field on a resource has not been set.  |
   |    `invalid`     |      字段的格式无效，该资源的文档需要提供更具体的信息。      |
   | `already_exists` | 另一个资源与此字段具有相同的值。这可能发生在必须具有某些唯一键（如标签名）的资源中。 |

   资源也可能发送自定义验证错误，自定义错误将始终有一个“消息”字段来描述错误，大多数错误还将包括一个“document_url”字段，该字段指向可能有助于解决错误的某些内容。

### HTTP动作

使用API时的HTTP动作 

|   Verb   |               Description                |
| :------: | :--------------------------------------: |
|  `HEAD`  | 可以针对任何资源发出，仅获取HTTP头信息。 |
|  `GET`   |      Used for retrieving resources.      |
|  `POST`  |       Used for creating resources.       |
| `PATCH`  |          用于更新部分的JSON信息          |
|  `PUT`   |           用于替换资源或集合。           |
| `DELETE` |       Used for deleting resources.       |



#### 查看博客文章列表

- 使用GET方法查看某个用户的博客文章列表
- `GET /:username/article`
- `curl -i -u https://blog.com/username/article`
- username为用户名，article表示进入文章页面，并返回文章列表。

#### 查看具体某一篇文章

- 使用GET方法查看某个用户的某篇博客文章
- `GET /:username/article/title`
- `curl -i -u https://blog.com/username/article/title`
- username为用户名，article表示进入文章页面，title为具体文章题目。

####  搜索文章

- 使用get方法在文章列表中查找满足搜索关键字的文章
- `GET /:username/search/article`

- `curl -u -i https://blog.com/username/search/article?name=name`
- username为用户名，search表示进入搜索，后面时get方法的参数，也就是搜索条件。

#### 发表文章

* 通过post方法以及提供文章内容以及标题进行发表文章
* `POST /:username/publish/article`
* `curl -u -i -d '{"title":"","content":"","private":false,"other":""}' https://blog.com/username/publish/title`

* 参数:

  ```
  {
  "title":"文章标题",					//标题
  "content":"文章内容",				//内容
  "private":true/false,			//是否公开
  "other":"其他"						//其他信息
  }
  ```

#### 修改文章

* 使用put方法对文章资源进行修改

* `PUT /:username/update/article`

* `curl -u -i https://exampleBlog.com/user1/update/article -d {"id":"","content":"","title":""}`

* 参数

  ```
  {
  "id": 			"文章的id",
  "title": 		"修改标题",
  "content":	"修改内容",
  }
  ```

#### 删除文章

* 使用delete方法删除对应的文章资源
* `DELETE /:username/delete/article`
* `curl -i -u https://blog.com/username/delete/article&id = 1`
* 删除时在url给出删除文章的id。

### 用户代理

所有API请求都必须包含有效的用户代理头，没有用户代理头的请求将被拒绝。这里要求使用博客的用户名或应用程序名称作为用户代理头值。

例子

```
User-Agent: Awesome-Octocat-App
```

Curl 默认发送一个有效的 `User-Agent` ，如果通过一个无效的头文件使用curl进行访问将返回 `403 Forbidden` 错误信息：

~~~
curl -iH 'User-Agent: ' https://api.blog.com
HTTP/1.0 403 Forbidden
Connection: close
Content-Type: text/html
Request forbidden by administrative rules.
Please make sure your request has a User-Agent header.
Check https://developer.github.com for other possible causes.
~~~

### 条件请求

大多数响应都返回`ETag`标头。许多响应还返回`Last-Modified`标头。您可以使用这些标头的值分别使用`If-None-Match`和`If-Modified-Since头`向这些资源发出后续请求。如果资源没有更改，则服务器将返回`304 Not Modified`。

~~~
curl -i https://api.blog.com/user
HTTP/1.1 200 OK
Cache-Control: private, max-age=60
ETag: "644b5b0155e6404a9cc4bd9d8b1ae730"
Last-Modified: Thu, 05 Jul 2012 15:31:30 GMT
Status: 200 OK
Vary: Accept, Authorization, Cookie
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4996
X-RateLimit-Reset: 1372700873
curl -i https://api.blog.com/user -H 'If-None-Match: "644b5b0155e6404a9cc4bd9d8b1ae730"'
HTTP/1.1 304 Not Modified
Cache-Control: private, max-age=60
ETag: "644b5b0155e6404a9cc4bd9d8b1ae730"
Last-Modified: Thu, 05 Jul 2012 15:31:30 GMT
Status: 304 Not Modified
Vary: Accept, Authorization, Cookie
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4996
X-RateLimit-Reset: 1372700873
curl -i https://api.blog.com/user -H "If-Modified-Since: Thu, 05 Jul 2012 15:31:30 GMT"
HTTP/1.1 304 Not Modified
Cache-Control: private, max-age=60
Last-Modified: Thu, 05 Jul 2012 15:31:30 GMT
Status: 304 Not Modified
Vary: Accept, Authorization, Cookie
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4996
X-RateLimit-Reset: 1372700873
~~~


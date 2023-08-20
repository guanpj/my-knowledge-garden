---
title: HTTP 基础
date created: 2023-03-23
date modified: 2023-03-23
dg-publish: true
tags:
  - 计算机网络
  - 报文
  - HTTPS
---
# 什么是 HTTP ？

Hypertext Transfer Protocol，超文本传输协议，和 HTML (Hypertext Markup Language 超文本标记语言) 一起诞生，用于在网络上请求和传输 HTML 内容。

超文本，即「扩展型文本」，指的是 HTML 中可以有链向别的文本的链接 (hyperlink)。

# HTTP 报文格式

请求报文格式：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Http/clipboard_20230323_031410.png)

响应报文格式：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Http/clipboard_20230323_031414.png)

# Request Method <strong>请求方法 </strong>

## GET

- 用于获取资源
- 对服务器数据不进行修改
- 不发送 Body

```plain text
GET  /users/1  HTTP/1.1
Host: api.github.com
```

对应 Retrofit 的代码：

```java
@GET("/users/{id}")
Call<User> getUser(@Path("id") String id, 
                  @Query("gender") String gender);
```

## POST

- 用于增加或修改资源
- 发送给服务器的内容写在 Body 里面

```plain text
POST  /users  HTTP/1.1
Host: api.github.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
name=guanpj&gender=male
```

对应 Retrofit 的代码：

```java
@FormUrlEncoded
@POST("/users")
Call<User> addUser(@Field("name") String name, 
                  @Field("gender") String gender);
```

## PUT

- 用于修改资源
- 发送给服务器的内容写在 Body 里面

```plain text
PUT  /users/1  HTTP/1.1
Host: api.github.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
gender=female
```

对应 Retrofit 的代码：

```java
@FormUrlEncoded
@PUT("/users/{id}")
Call<User> updateGender(@Path("id") String id,
                        @Field("gender") String gender);
```

## DELETE

- 用于删除资源
- 不发送 Body

```plain text
DELETE  /users/1  HTTP/1.1
Host: api.github.com
```

对应 Retrofit 的代码：

```java
@DELETE("/users/{id}")
Call<User> getUser(@Path("id") String id,
                  @Query("gender") String gender);
```

## HEAD

- 和 GET 使用方法完全相同
- 和 GET 唯一区别在于，返回的响应中没有 Body

# Status Code <strong>状态码</strong>

三位数字，用于对响应结果做出类型化描述(如「获取成功」「内容未找到」)。

- 1xx:临时性消息。如:100 (继续发送)、101(正在切换协议)
- 2xx:成功。最典型的是 200(OK)、201(创建成功)。
- 3xx:重定向。如 301(永久移动)、302(暂时移动)、304(内容未改变)。
- 4xx:客户端错误。如 400(客户端请求错误)、401(认证失败)、403(被禁 止)、404(找不到内容)。
- 5xx:服务器错误。如 500(服务器内部错误)。

# Header <strong>首部</strong>

作用：HTTP 消息的 metadata（即原数据）。

## Host

目标主机。注意:不是在网络上用于寻址的，而是在目标服务器上用于定位子服务

器的。

## Content-Type

指定 Body 的类型。主要有四类:

### text/html

请求 Web ⻚面是返回响应的类型，Body 中返回 html 文本。格式如下:

```plain text
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 853
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
......
```

### x-www-form-urlencoded

Web ⻚面纯文本表单的提交方式。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Http/clipboard_20230323_031421.png)

格式如下:

```plain text
POST  /users  HTTP/1.1
Host: api.github.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 27
name=guanpj&gender=male
```

对应 Retrofit 的代码：

```java
@FormUrlEncoded
@POST("/users")
Call<User> addUser(@Field("name") String name,
@Field("gender") String gender);
```

### multipart/form-data

Web ⻚面含有二进制文件时的提交方式。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Http/clipboard_20230323_031424.png)

格式如下:

```plain text
POST  /users  HTTP/1.1
Host: guanpj.me.com
Content-Type: multipart/form-data; boundary=----
WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 2382
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="name"
guanpj
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="avatar";
filename="avatar.jpg"
Content-Type: image/jpeg
JFIFHHvOwX9jximQrWa......
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

对应 Retrofit 的代码:

```java
@Multipart
@POST("/users")
Call<User> addUser(@Part("name") RequestBody name,
@Part("avatar") RequestBody avatar);
...
RequestBody namePart =
RequestBody.create(MediaType.parse("text/plain"), nameStr);
RequestBody avatarPart =
RequestBody.create(MediaType.parse("image/jpeg"), avatarFile);
api.addUser(namePart, avatarPart);
```

### application/json , image/jpeg , application/zip ...

单项内容(文本或非文本都可以)，用于 Web Api 的响应或者 POST / PUT 的请求

<strong>请求中提交 JSON</strong>

```plain text
POST /users HTTP/1.1
Host: hencoder.com
Content-Type: application/json; charset=utf-8
Content-Length: 38
{"name":"guanpj","gender":"male"}
```

对应 Retrofit 的代码:

```java
@POST("/users")
Call<User> addUser(@Body("user") User user);
...
// 需要使用 JSON 相关的 Converter 
api.addUser(user);
```

<strong>响应中返回 JSON</strong>

```plain text
HTTP/1.1 200 OK
content-type: application/json; charset=utf-8
content-length: 234
[{"login":"mojombo","id":1,"node_id":"MDQ6VXNl
cjE=","avatar_url":"https://avatars0.githubuse
rcontent.com/u/1?v=4","gravat......
```

<strong>请求中提交二进制内容</strong>

```plain text
POST /user/1/avatar HTTP/1.1
Host: hencoder.com
Content-Type: image/jpeg
Content-Length: 1575
JFIFHH9......
```

对应 Retrofit 的代码:

```java
@POST("users/{id}/avatar")
Call<User> updateAvatar(@Path("id") String id, @Body
RequestBody avatar);
...
RequestBody avatarBody =
RequestBody.create(MediaType.parse("image/jpeg"),
avatarFile);
api.updateAvatar(id, avatarBody)
```

<strong>响应中返回二进制内容</strong>

```plain text
HTTP/1.1 200 OK
content-type: image/jpeg
content-length: 1575
JFIFHH9......
Content-Length
```

指定 Body 的⻓度(字节)。

## Transfer: chunked (<strong>分块传输编码</strong>)

用于当响应发起时，内容⻓度还没能确定的情况下。和 Content-Length 不同时使 用。用途是尽早给出响应，减少用户等待。

格式：

```plain text
HTTP/1.1 200 OK
Content-Type: text/html
Transfer-Encoding: chunked
4
Chun
9
ked Trans
12
fer Encoding
0
Location
```

指定重定向的目标 URL。

## User-Agent

用户代理，即是谁实际发送请求、接受响应的，例如手机浏览器、某款手机 App。

## Range / Accept-Range

按范围取数据。

- `Accept-Range: bytes`： 响应报文中出现，表示服务器支持按字节来取范围数据；
- `Range: bytes=<start>-<end>`： 请求报文中出现，表示要取哪段数据；
- `Content-Range:<start>-<end>/total`： 响应报文中出现，表示发送的是哪段 数据。

作用：断点续传、多线程下载。

## <strong>其他 </strong>Headers

- Accept：客户端能接受的数据类型。如 text/html
- Accept-Charset：客户端接受的字符集。如 utf-8。
- Accept-Encoding：客户端接受的压缩编码类型。如 gzip。
- Content-Encoding：压缩类型。如 gzip。

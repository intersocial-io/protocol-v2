# InterSocial2
InterSocial2是一个联邦式社交协议。
## 1. 用户识别
### 1.1. 用户名格式
用户标识符的格式是`<用户名>.<服务器地址>`。例如：`alice.example.com`。

用户名可以包含的字符：小写字母，阿拉伯数字，横杠，中日韩文字。

因为用户名不能包含点号，所以可以使用第一个点号来作为用户名和服务器地址的分隔符。
### 1.2. 账户端点
端点：`GET https://<服务器地址>/.well-known/intersocial2/user/<用户名>`

请求数据：无

返回数据：JSON格式，定义如下：
```json
{
  "code": 200,
  "nickname":"<用户昵称>",
  "header_url":"<用户头像URL>",
  "rss": "<用户RSS地址>",
  "sections":[
    {"key":"<信息条目1>", "value":"<条目1内容>"},
    {"key":"<信息条目2>", "value":"<条目2内容>"}
  ]
}
```
示例如下：
```json
{
  "code": 200,
  "nickname":"alice",
  "header_url":"https://oss.example.com/user-alice.png",
  "rss": "https://example.com/rss/person/alice",
  "sections":[
    {"key":"full name", "value":"alice black"},
    {"key":"gender", "value":"normal female"},
    {"key":"hobby", "value":"cryptography"}
  ]
}
```
其中，`$`符号开头的section被视为变量（Variable）。变量名应该是一个包名。

我们官方背书的变量定义列表：
| 变量 Variable | 含义 Description |
| --- | --- |
| $std.email | 电子邮件地址 |
| $com.bilibili.uid | B站UID |
| $com.qq.wxid | 微信号 |
| $com.qq.qqid | QQ号 |
| $std.phone | 手机号 |
| $org.ethereum.address | 以太坊地址 |
| $com.github.user | Github用户名 |
| $org.gnu.gpgkey | GPG公钥（完整） |
| $org.gnu.gpgfinger | GPG指纹 |
| $org.matrix.account | Matrix地址 |
| $social.mastodon.name | AP系Fedi用户名 |

## 2. 内容发布

InterSocial2使用RSS作为帖子流协议。RSS地址包含在用户端点中。

通过[PubSubHubBub协议](https://www.w3.org/TR/websub/)来实现消息推送。

### 2.1. 单帖子端点
单个帖子的检索端点：`GET https://<服务器域名>/.well-known/intersocial2/posts/<用户名>/<Permalink>`

请求数据：无。

返回数据：json格式，定义如下：
```json
{
  "code": 200,
  "rss": "<RSS Item对象>",
  "sender": "<发布者用户名>"
}
```
示例如下：
```json
{
  "code": 200,
  "rss": "<item intersocialpost=\"helloworld\">...</item>",
  "sender": "alice"
}
```

在RSS的item项中，会包含一个intersocialpost字段，代表帖子permalink。
```xml
<item intersocialpost="helloworld">...</item>
```

### 2.2. 评论发布

RSS的Item项中，如果有一个intersocialreplyto字段，代表这是一个评论。

这个字段内容的定义如下：`<用户名>.<服务器地址>/<Permalink>`（即“帖子标识符”，下文以此表示），例子：`alice.example.com/helloworld`

当被评论者所在的服务器感知到评论后，会在item中添加一个Reply对象，包含评论信息。定义如下：
```xml
<item intersocialpost="helloworld">
...
<reply postcode="<用户名>.<服务器地址>/<Permalink>"/>
...
</item>
```
形如：
```xml
<item intersocialpost="helloworld">
<title>你好，世界！</title>
<content>你好，世界！你好，InterSocial！</content>
<reply postcode="bob.example.org/reply-298471"/>
<reply postcode="charlie.example.cn/curation-194jaixpfu25usy3"/>
</item>
```

### 2.3. 点赞与关注

RSS的channel中，可以添加Action项。Action项类似于Item项，用于承载点赞、关注等信息。
```xml
<action type="like"> <!-- type项是操作类型 -->
  <id>9d9940f4-01c4-c257-6771-e721eb7a48cd</id> <!-- UUID -->
  <target>alice.example.com/helloworld</target> <!-- 目标，可以是用户标识符、帖子标识符、互动UUID等 -->
</action>
```
操作类型列表：
| 操作 Action | 含义 Description | 目标类型 Target type |
| --- | --- | --- |
| like | 点赞 | 帖子 |
| downvote | 点踩 | 帖子 |
| repost | 转发 | 帖子 |
| subscribe | 关注 | 用户 |
| cancel | 取消 | 互动 |

Action端点：`GET https://<服务器域名>/.well-known/intersocial2/action/<UUID>`

请求数据：无

返回数据：JSON格式，定义如下：

```json
{
  "code": 200,
  "action": {
    "sender": "<用户名>",
    "type": "<操作>",
    "target": "<目标>"
  }
}
```

## 3. 互动信息推送

### 3.1. 单评论推送

推送端点：`POST https://<目的服务器域名>/.well-known/intersocial2/commit`

请求数据：JSON格式，定义如下：
```json
{
  "data": [
    {
      "server": "<源服务器域名>",
      "type": "reply",
      "id": "<评论本身的帖子标识符>"
    }
  ]
}
```
返回数据：JSON格式，定义如下：
```json
{ "code": 200 }
```

### 3.2. 单互动推送

推送端点：`POST https://<目的服务器域名>/.well-known/intersocial2/commit`

请求数据：JSON格式，定义如下：
```json
{
  "data": [
    {
      "server": "<源服务器域名>",
      "type": "interactive",
      "id": "<互动的UUID>"
    }
  ]
}
```
返回数据：JSON格式，定义如下：
```json
{ "code": 200 }
```

### 3.3. 单私聊请求推送

InterSocial2的私聊协议是 Matrix。

注意：InterSocial2的Matrix**不是**端到端的，私钥必须存在服务器上，否则无法完成握手。

推送端点：`POST https://<目的服务器域名>/.well-known/intersocial2/commit`

请求数据：JSON格式，定义如下：
```json
{
  "data": [
    {
      "server": "<源服务器域名>",
      "type": "matrixreq",
      "id": "<随机生成的UUID>",
      "from": "<源用户名>",
      "matrix": "<源Matrix>",
      "to": "<目的用户名>"
    }
  ]
}
```
返回数据：JSON格式，定义如下：
```json
{
  "code": 200
}
```
请求后，目的服务器会向源服务器推送私聊响应消息。

推送端点：`POST https://<源服务器域名>/.well-known/intersocial2/commit`

请求数据：JSON格式，定义如下：
```json
{
  "data": [
    {
      "server": "<目的服务器域名>",
      "type": "matrixresp",
      "id": "<与私聊请求相同的UUID>",
      "matrix": "<目的用户Matrix地址>",
      "sign": "<签名私密码>"
    }
  ]
}
```
返回数据：JSON格式，定义如下：
```json
{
  "code": 200
}
```

随后，源服务器需要计算签名，签名的计算规则：
```php
$sign = md5(
  "intersocial_matrix_"
  . md5(
    md5($source_identifier) // 源用户标识符
    . md5($source_matrix) // 源Matrix地址
    . md5($target_martix) // 目标Matrix地址
  ) . $secret // 签名私密码
)
```

随后，源服务器通过Matrix向目标服务器发送一个聊天消息（即“握手消息”），内容是一个JSON文本，定义如下：
```json
{
  "type": "intersocial2.matrixchat.handshake",
  "version": "2a",
  "sign": "<签名码>",
  "user": "<源用户标识符>"
}
```
示例：
```json
{
  "type": "intersocial2.matrixchat.handshake",
  "version": 2,
  "sign": "4d12b08f63fee0657f884f87d5d42bad",
  "user": "bob.example.org"
}
```

然后，目标服务器发送一串hello码，即可完成最终握手。格式：
```
INTERSOCIAL ACCEPTED HELLO <与私聊请求相同的UUID> V=2
```
例如：
```
INTERSOCIAL ACCEPTED HELLO 65a0ba40-46d6-5802-ab26-f8063c51123b V=2
```

### 3.4. 多消息同时推送

只需要将多个请求的data合并为多项列表再发送即可。

## 4. 绑定验证

如需进行InterSocial2账号绑定验证，需要先生成Secret，然后获得Secret哈希：
```php
$hash = md5("intersocial_secret_blobcat" . $secret)
```
绑定跳转URL：`https://<服务器域名>/.well-known/isisauth/create?callback=<跳转URL>&secrethash=<Secret哈希>&hashver=1`

最终返回的URL：`<callback地址>?authcode=<auth码>`

取回用户信息的端点：`https://<服务器域名>/.well-known/isisauth/resolve?authcode=<auth码>`

返回的数据（JSON格式）：
```json
{
  "code": 200,
  "success": true,
  "intersocial2":{
    "username": "<用户名>"
  }
}
```

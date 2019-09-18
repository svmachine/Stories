# <center>第三方集成钉钉实现单点</center>

钉钉的免登接口相较于微信来说比较复杂，并且分为h5应用和e应用两个版本，目前钉钉的api文档逐渐以e应用为主，e应用需要借助与钉钉官方提供的ide开发，两者之间的区别详见[https://open-doc.dingtalk.com/microapp/isv](https://open-doc.dingtalk.com/microapp/isv).

该文档主要以h5微应用来介绍获取访问用户信息实现免登的方法.

**1.环境准备**

js：[http://g.alicdn.com/dingding/dingtalk-jsapi/2.3.0/dingtalk.open.js](http://g.alicdn.com/dingding/dingtalk-jsapi/2.3.0/dingtalk.open.js)

> 具体可参考：[https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.31c14a97RE1GvB&treeId=171&articleId=104910&docType=1](https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.31c14a97RE1GvB&treeId=171&articleId=104910&docType=1)

corpId：在[https://open-dev.dingtalk.com/\#/index](https://open-dev.dingtalk.com/#/index)中可看到

CorpSecret：在[https://open-dev.dingtalk.com/\#/devAuthorize](https://open-dev.dingtalk.com/#/devAuthorize)中可看到

AccessTokenUrl :[https://oapi.dingtalk.com/gettoken?corpid=id&corpsecret=secrect](https://oapi.dingtalk.com/gettoken?corpid=id&corpsecret=secrect)

jsticketUrl :[https://oapi.dingtalk.com/get\_jsapi\_ticket?access\_token=ACCESS\_TOKE](https://oapi.dingtalk.com/get_jsapi_ticket?access_token=ACCESS_TOKE)

agentId：新建应用时的agentId

userIdUrl：[https://oapi.dingtalk.com/user/getuserinfo?access\_token=access\_token&code=authCode](https://oapi.dingtalk.com/user/getuserinfo?access_token=access_token&code=authCode)

userUrl：[https://oapi.dingtalk.com/user/get?access\_token=access\_token&userid=userid](https://oapi.dingtalk.com/user/get?access_token=access_token&userid=userid)根据用户id获取用户详细信息

**2.新建应用**

在钉钉管理后台工作台--自建应用中新建应用，设置图标、名称、简介、访问链接、有权访问用户等，确定之后，有权限访问的用户可以在手机端钉钉应用中看到新建的应用，点击即可访问（此时需要登录）

**3.实现免登**

手机端钉钉，获取访问的钉钉用户有两种方式（鉴权/非鉴权）

**鉴权：**

**第一步：**在访问的html中引入dingtalk.open.js.

**第二步：**鉴权获取access\_token，代码如下

```java
public static String getAccessToken(String corpSecret, String corpId, String url) throws Exception {
  url = url + "?corpid=" + corpId + "&corpsecret=" + corpSecret;
  String result = HttpUtil.getResult(url);
  ObjectMapper mapper = new ObjectMapper();

  JsonNode node = mapper.readTree(result);
  int errcode = node.path("errcode").asInt();

  if (errcode != 0) {
     throw new Exception("获取access_token失败!");
  }

  String access_token = node.path("access_token").asText();
  return access_token;
}
```

**第三步：**获取jsticket，代码如下

```java
public static String getTicket(String access_token, String jsticketUrl) throws Exception {
 url += "?access_token=" + access_token;
 String result = HttpUtil.getResult(url);

 ObjectMapper mapper = new ObjectMapper();
 JsonNode jsonNode = mapper.readTree(result);

 int errcode = jsonNode.path("errcode").asInt();
 String errmsg = jsonNode.path("errmsg").asText();

 if (errcode != 0) {
    throw new Exception("获取jsticket失败!" + errmsg);
 }

 String ticket = jsonNode.path("ticket").asText();
 return ticket;
}
```

**第四步：**计算签名信息， 代码如下

```java
/**nonceStr：随机串，自己定义
  timeStamp：当前时间，但是前端和服务端进行校验时候的值要一致
  url：当前请求网页的url
**/
public static String getSign(String ticket, String nonceStr, long timeStamp, String url) throws Exception {
 return DingTalkJsApiSingnature.getJsApiSingnature(url, nonceStr, timeStamp, ticket);
}
```

**第五步：**参数传递到前端页面，进行校验

```java
protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
 String accessTokenUrl = "https://oapi.dingtalk.com/gettoken";
 String ticketUrl = "https://oapi.dingtalk.com/get_jsapi_ticket";

 /**
  ** 该地方之所以从参数中获取url，是和项目中的特殊情况有关，后续会说明
 **/
 String url = request.getParameter("url");
 String corpId = corpId;
 String corpSecret = CorpSecret;
 String nonceStr = “123456”;
 String agentId = agentId;

 long timeStamp = System.currentTimeMillis() / 1000;
 String signedUrl = url;
 String accessToken = null;
 String ticket = null;
 String signature = null;

 try {
  /** accessToken的有效期是2个小时，可以设置下缓存，从缓存中取**/
  Object token = Cache.get("accessToken");

  if (token == null) {
    accessToken = getAccessToken(corpSecret, corpId, accessTokenUrl);
    Cache.put("accessToken", accessToken);
  } else {
    accessToken = token.toString();
  }

  ticket = getTicket(accessToken, ticketUrl);
  signature = getSign(ticket, nonceStr, timeStamp, signedUrl);
 } catch (Exception e) {
  e.printStackTrace();
 }

 HashMap<Object, Object> map = new HashMap<Object, Object>();
 map.put("jsticket", ticket);
 map.put("signature", signature);
 map.put("nonceStr", nonceStr);
 map.put("timeStamp", timeStamp);
 map.put("corpId", corpId);
 map.put("agentid", agentId);

 try {
     response.getWriter().print(new ObjectMapper().writeValueAsString(map));
 } catch (JsonProcessingException e) {
     e.printStackTrace();
 }
}
```

**第六步：**前端js代码

```java
$(document).ready(function () {
 var url = window.location.href;
 var corpId = "dinge7c8f20eab0f079735c2f4657eb6378f";
 var signature = "";
 var nonce = "";
 var timeStamp = "";
 var agentId = "";
 $.post( "get_js_config", { "url": url, "corpId": corpId }, function (result) {
     var re = JSON.parse(result);
     dd.config({
        agentId : re.agentid,
        corpId : re.corpId,
        timeStamp : re.timeStamp,
        nonceStr : re.nonceStr,
        signature : re.signature,
        jsApiList : [ 'runtime.info', 'biz.contact.choose',
        'device.notification.confirm', 'device.notification.alert',
        'device.notification.prompt', 'biz.ding.post',
        'biz.util.openLink', 'biz.chat.pickConversation',
        'biz.chat.chooseConversationByCorpId']
     });

     dd.ready(function() {
          dd.runtime.permission.requestAuthCode({
             corpId : corpId,
             onSuccess : function(info) {
                $.get("userinfo?code=" + info.code + "&corpid=" + corpId, function(){
                      window.location.reload();
                })
             }
          })
      });
    dd.error(function(err) {
       alert('dd error: ' + JSON.stringify(err));
    });
 });
});
```

**第七步：**获取用户信息的方法, 代码如下

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
 String corpId = orpId;
 String corpSecret = CorpSecret;
 String code = request.getParameter("code");

 String accessTokenUrl = AccessTokenUrl;
 String useridUrl = userIdUrl;
 String userUrl = userUrl;

 ObjectMapper mapper = new ObjectMapper();
 String accessToken = null;

 try {
   response.setContentType("text/html; charset=utf-8");
   /** accessToken的有效期是2个小时，可以设置下缓存，从缓存中取  **/
   Object token = Cache.get("accessToken");

   if (token == null) {
    accessToken = getAccessToken(corpSecret, corpId, accessTokenUrl);
    Cache.put("accessToken", accessToken);
   } else {
    accessToken = token.toString();
   }

   String userid = getUserId(accessToken, code, useridUrl);
   String user = getUser(accessToken, userid, userUrl);
   Map map = mapper.readValue(user, Map.class);
 
 /** 此时已经拿到用户的信息，进行单点登录操作 **/
  } catch (Exception e) {
   e.printStackTrace();
   response.getWriter().append(e.getMessage());
  }
}

/**
** 获取用户id的方法
**/
public static String getUserId(String accessToken, String code, String url) throws Exception {
 url = "?access_token=" + accessToken + "&code=" + code;
 String result = HttpUtil.getResult(url);

 ObjectMapper mapper = new ObjectMapper();
 JsonNode jsonNode = null;

 try {
  jsonNode = mapper.readTree(result);
 } catch (IOException e) {
  e.printStackTrace();
 }

 int errcode = jsonNode.path("errcode").asInt();
 String errmsg = jsonNode.path("errmsg").asText();

 if (errcode != 0) {
  throw new Exception("获取userId失败!" + errmsg);
 }

 String userid = jsonNode.path("userid").asText();
 return userid;
}

/**
** 根据id获取用户方法
**/
public static String getUser(String accessToken, String userid, String url) throws Exception {
 url += "?access_token=" + accessToken + "&userid=" + userid;
 String result = HttpUtil.getResult(url);

 ObjectMapper mapper = new ObjectMapper();
 JsonNode jsonNode = null;

 try {
  jsonNode = mapper.readTree(result);
 } catch (IOException e) {
  e.printStackTrace();
 }

 int errcode = jsonNode.path("errcode").asInt();
 String errmsg = jsonNode.path("errmsg").asText();
 if (errcode != 0) {
  throw new Exception("获取user失败!" + errmsg);
 }

 return result;
}
```

**非鉴权：**非鉴权获取的用户信息相比较鉴权获取的用户信息要少，具体操作如下

**第一步：**在访问的html中引入dingtalk.open.js.

**第二步：**前端js，代码如下：

```js
$(document).ready(function () {
    dd.ready(function() {
      dd.runtime.permission.requestAuthCode({
          corpId : corpId,
          onSuccess : function(info) {
              $.get("userinfo?code=" + info.code + "&corpid=" + corpId, function(){
                  window.location.reload();
              })
          }
      })
    });

  dd.error(function(err) {
      alert('dd error: ' + JSON.stringify(err));
  });
});
```

**第三步：**获取用户信息，与鉴权中的第七步一样

**说明**

1.在项目中因无法先加载dingtalk.open.js以及获取用户信息的js代码，采用了一种没有登录跳转到指定页面，由该页面加载js获取用户信息并登陆，登录完成后使用window.location.reload\(\)刷新当前访问地址，通过这样来避免二次跳转的问题。

2.后续url是对应项目地址，项目中的loading.html即是用来跳转的指定页面

3.PC端的单点登录与app端相似，只需要将js换成[http://g.alicdn.com/dingding/dingtalk-pc-api/2.3.1/index.js](http://g.alicdn.com/dingding/dingtalk-pc-api/2.3.1/index.js), 然后将前端js中的dd换成DingTalkPC即可

​

## 特别说明

​

钉钉api文档最近做了更新，更新后获取access\_token的方式发生改变, 使用老方法获取access\_token会提示52013， 签名校验失败的错误

**更新前：**

通过corpId CorpSecret 获取access\_token, 具体见鉴权的第二步操作

**更新后：**

通过AppKey AppSecret （在钉钉开放平台---&gt;应用开发---&gt;微应用---&gt;微应用管理 中点开对应的应用即可查看）获取access\_token, 其他保持不变

> 以上更新说明仅针对在2018-12月之后创建的微应用而言，对于2018-12以前创建的微应用旧方法依然有效




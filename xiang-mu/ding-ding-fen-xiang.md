在钉钉实现免登操作后，可以直接访问第三方应用，基本上能满足查看的功能，但是在本人对应的项目中遇到这样的问题（主要以BI报表为主）

> 1.要求只能在钉钉中分享一张比较好的报表，但报表在手机端不支持分享功能
>
> 2.在使用钉钉自带容器，通过右上角自带的分享功能分享到钉钉中，因查看报告需要权限控制，分享后展示的内容为登录页面，效果不好

对于上述情况，钉钉提供了对应的api，可以修改钉钉容器导航栏上的按钮，具体可参考[https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.3b544a97sakwCp&treeId=171&articleId=104928&docType=1\#s8](https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.3b544a97sakwCp&treeId=171&articleId=104928&docType=1#s8), 对于只存在一个按钮操作，可以biz.navigation.setRight方法；如果是按钮菜单使用biz.navigation.setMenu（效果不太好）

本人的思路是重写导航栏，添加分享按钮，将当前访问的url通过后台以link的形式发送到钉钉中。

该文档主要以h5微应用来介绍分享功能的实现方法.

**1.环境准备**

js：[http://g.alicdn.com/dingding/dingtalk-pc-api/2.3.1/index.js](http://g.alicdn.com/dingding/dingtalk-pc-api/2.3.1/index.js)

> 具体可参考：[https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.31c14a97RE1GvB&treeId=171&articleId=104910&docType=1](https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.31c14a97RE1GvB&treeId=171&articleId=104910&docType=1)

corpId：在[https://open-dev.dingtalk.com/\#/index](https://open-dev.dingtalk.com/#/index)中可看到

CorpSecret：在[https://open-dev.dingtalk.com/\#/devAuthorize](https://open-dev.dingtalk.com/#/devAuthorize)中可看到

AccessTokenUrl : [https://oapi.dingtalk.com/gettoken?corpid=id&corpsecret=secrect](https://oapi.dingtalk.com/gettoken?corpid=id&corpsecret=secrect)

jsticketUrl : [https://oapi.dingtalk.com/get\_jsapi\_ticket?access\_token=ACCESS\_TOKE](https://oapi.dingtalk.com/get_jsapi_ticket?access_token=ACCESS_TOKE)

agentId：新建应用时的agentId

userIdUrl：[https://oapi.dingtalk.com/user/getuserinfo?access\_token=access\_token&code=authCode](https://oapi.dingtalk.com/user/getuserinfo?access_token=access_token&code=authCode)

userUrl：[https://oapi.dingtalk.com/user/get?access\_token=access\_token&userid=userid](https://oapi.dingtalk.com/user/get?access_token=access_token&userid=userid)根据用户id获取用户详细信息

sendMessageUrl：[https://oapi.dingtalk.com/message/send\_to\_conversation?access\_token=ACCESS\_TOKEN](https://oapi.dingtalk.com/message/send_to_conversation?access_token=ACCESS_TOKEN)

**2.新建应用**

在钉钉管理后台工作台--自建应用中新建应用，设置图标、名称、简介、访问链接、有权访问用户等，确定之后，有权限访问的用户可以在手机端钉钉应用中看到新建的应用，点击即可访问（此时需要登录）

**3.实现免登**

实现分享功能的前提是先要登录，具体免登的方法可以参考前一篇文章

**4.鉴权操作**

修改导航栏操作，是不需要鉴权操作的，但是后续选择聊天框是需要鉴权的，具体的鉴权操作js如下，后台代码与前一篇免登一样，可以自行参考

```js
$(document).ready(function () {
    var url = window.location.href;
    var corpId = corpId;
    var signature = "";
    var nonce = "";
    var timeStamp = "";
    var agentId = "";
    $.post(
        "get_js_config",
        {
            "url": url,
            "corpId": corpId
        },
        function (result) {
            var re = JSON.parse(result);

            <!-- 使用需要鉴权的jsapi应先在jsApiList中添加-->
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
                    'biz.chat.chooseConversationByCorpId','biz.navigation.setMenu']
            });

            dd.ready(function() {
                dd.runtime.permission.requestAuthCode({
                    corpId : corpId,
                    onSuccess : function(info) {
                       /** 修改导航栏的操作 **/
                    },
                    onFail : function(err) {
                        alert(JSON.parse(err));
                    }
                })
            })
            dd.error(function(err) {
                alert('dd error: ' + JSON.stringify(err));
            });
        });
});
```

**4.修改导航栏**

在前端js页面，步骤三中注释位置，添加分享按钮， 具体代码如下

```js
dd.biz.navigation.setMenu({
    backgroundColor : "#ADD8E6",
    items : [
        {
            id:"1",
            text:"分享"
        }
    ],
    onSuccess: function(data) {
        if(data.id == 1) {
            sendShare(window.location.href);
        }
    },
    onFail: function(err) {
        alert(JSON.stringify(err));
    }
});
```

**5.选择需要分享的群聊或者个人**

当点击分享按钮时，调用钉钉api去选择一个分享的群组或者个人，获取相应的cid（只能使用一次），具体代码如下

```js
function sendShare(url) {
    dd.biz.chat.pickConversation({
        corpId: corpId, //企业id
        isConfirm:'true', //是否弹出确认窗口，默认为true
        onSuccess : function(result) {
            /**此处可以获取到url，构造到result中**/
            $.post("sendShare", {"key":JSON.stringify(result)}, function () {
                alert("分享成功!");
            })
        },
        onFail : function(err) {
            alert('dd biz error: ' + JSON.stringify(err));
        }
    })
}
```

**6.后台实现分享功能**  
当选择分享的群组或者个人后，可以获取到对应的cid，通过cid参数，可以向对应的群聊中发送消息，为了避免开头提到的问题1，使用文本的形式向群聊中发送数据，具体代码如下

```java
protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String corpId = corpId;
    String corpSecret = CorpSecret;
    String accessTokenUrl = AccessTokenUrl;
    String parameter = req.getParameter("key");
    Map<Object, Object> map = new ObjectMapper().readValue(parameter, Map.class);

    resp.setContentType("text/html; charset=utf-8");

    Object token = Cache.get("accessToken");
    String accessToken = null;
    try {
       if (token == null) {
          accessToken = DingDingUtil.getAccessToken(corpSecret, corpId, accessTokenUrl);
          Cache.put("accessToken", accessToken);
       } else {
          accessToken = token.toString();
       }

       DingTalkClient client = new DefaultDingTalkClient(sendMessageUrl);

       OapiMessageSendToConversationRequest reqd = new OapiMessageSendToConversationRequest();
       reqd.setSender("钉钉中发送人id");
       reqd.setCid(map.get("cid").toString());
       OapiMessageSendToConversationRequest.Msg msg = new OapiMessageSendToConversationRequest.Msg();
       OapiMessageSendToConversationRequest.Link link = new OapiMessageSendToConversationRequest.Link();
       link.setMessageUrl(map.get("url").toString());
       link.setPicUrl("图片");
       link.setText("名称");
       link.setTitle("标题");
       msg.setLink(link);
       msg.setMsgtype("link");
       reqd.setMsg(msg);
       OapiMessageSendToConversationResponse res = client.execute(reqd, accessToken);
    }
    catch (Exception e) {

    }
 }
```

**7.点击分享内容**

前六步已经将url分享到钉钉群组或者个人，但对于一些要登录才能访问的系统来说，点击分享后还是需要登录，此时就需要自定义拦截器（在第6步中的url上加上一些特殊标记），实现免登陆功能

**说明**

1.在项目中因无法先加载dingtalk.open.js以及获取用户信息的js代码，采用了一种没有登录跳转到指定页面，由该页面加载js获取用户信息并登陆，登录完成后使用window.location.reload\(\)刷新当前访问地址，通过这样来避免二次跳转的问题。

2.钉钉api中没有获取所有群聊cid信息的接口，所以不能再后台实现数据发送，只能通过jsapi选择群聊或个人获取cid


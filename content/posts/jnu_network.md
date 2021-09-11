---
title: "暨南大学珠海校区有线网认证分析及Golang实现"
date: 2021-09-11T15:19:56+08:00
draft: false
---



## 前言

暨南大学珠海校区有线网使用H3C的**iMC Portal**网页认证，每次使用网络前需要登录认证，并且要在整个上网过程保持认证网页存在，否则就会被断开连接。并且在电脑睡眠、网络中断的情况下也会被断开连接，需要重新认证，突出一个麻烦。因此希望通过抓包分析其认证原理，并且编写脚本实现认证以及自动保活（这也给使用路由器带来了可能性）。

**懒人版总结：**

到[GitHub仓库](https://github.com/Steve0x2a/JNU_ZH_Network)处下载Releases对应版本，使用教程参考仓库的文档直接使用。

以下请求仅在教学区有线网络通过测试，其他地方尚未测试。

## 登录请求分析

通过抓包发现，校园网验证的登录接口为`portal/pws?t=li`，请求方式为`post`，发送的具体FormData如下所示：

```
userName: 学号
userPwd: base64编码的密码
userDynamicPwd: 
userDynamicPwdd: 
serviceType: 
isSavePwd: on
userurl: 
userip: 
basip: 
language: Chinese
usermac: null
wlannasid: 
wlanssid: 
entrance: null
loginVerifyCode: 
userDynamicPwddd: 
customPageId: 1
pwdMode: 0
portalProxyIP: 192.168.11.34
portalProxyPort: 50200
dcPwdNeedEncrypt: 1
assignIpType: 0
appRootUrl: https://xx:xx:xx:xx:xxxx/portal/
manualUrl: 
manualUrlEncryptKey: 
```

以上参数能改变的只有`userName`和`userPwd`两个参数， 其中`userName`是学号，`userPwd`是认证密码经过base64后的数值。

如果你的校园网当前是在线的，再次发送认证请求可能会返回以下内容：

```
JTdCJTIycG9ydFNlcnZFcnJvckNvZGUlMjIlM0ElMjIxJTIyJTJDJTIycG9ydFNlcnZFcnJvckNvZGVEZXNjJTIyJTNBJTIyJUU4JUFFJUJFJUU1JUE0JTg3JUU2JThCJTkyJUU3JUJCJTlEJUU4JUFGJUI3JUU2JUIxJTgyJTIyJTJDJTIyZV9jJTIyJTNBJTIycG9ydFNlcnZFcnJvckNvZGUlMjIlMkMlMjJlX2QlMjIlM0ElMjJwb3J0U2VydkVycm9yQ29kZURlc2MlMjIlMkMlMjJlcnJvck51bWJlciUyMiUzQSUyMjclMjIlN0Q
```

同样是base64编码后的内容，经过解码后可以得到如下内容：

```json
{
    "portServErrorCode": "1",
    "portServErrorCodeDesc": "设备拒绝请求",
    "e_c": "portServErrorCode",
    "e_d": "portServErrorCodeDesc",
    "errorNumber": "7"
}
```

经过不同的尝试，可以得到几种不同的返回：

1. 已经登录

   ```json
   {
       "portServErrorCode": "1",
       "portServErrorCodeDesc": "设备拒绝请求",
       "e_c": "portServErrorCode",
       "e_d": "portServErrorCodeDesc",
       "errorNumber": "7"
   }
   ```

2. 用户名错误或无权限

   ```json
    {
        "portServIncludeFailedCode": "63018",
        "portServIncludeFailedReason": "E63018:用户不存在或者用户没有申请该服务。",
        "e_c": "portServIncludeFailedCode",
        "e_d": "portServIncludeFailedReason",
        "errorNumber": "7"
    }
   ```

3. 密码错误

   ```json
    {
        "portServIncludeFailedCode": "63032",
        "portServIncludeFailedReason": "E63032:密码错误，您还可以重试x次。",
        "e_c": "portServIncludeFailedCode",
        "e_d": "portServIncludeFailedReason",
        "errorNumber": "7"
    }
   ```

4. 登录成功

   ```json
   {
       "errorNumber": "1",
       "heartBeatCyc": 60000,
       "heartBeatTimeoutMaxTime": 5,
       "userDevPort": "ZH-Campus-Core_SW-vlan-00-0409@vlan",
       "userStatus": 99,
       "serialNo": 11328,
       "ifNeedModifyPwd": false,
       "browserUrl": "",
       "clientPrivateIp": "",
       "userurl": "",
       "usermac": null,
       "nasIp": "",
       "clientLanguage": "Chinese",
       "ifTryUsePopupWindow": false,
       "triggerRedirectUrl": "",
       "portalLink": "xxx"
   }
   ```

   

如果能得到上面的返回， 基本上认证已经成功， 但是由于还需要保活，因此还需要继续抓包。

这里需要注意一下登陆返回成功的几个数值，后续会用得到：`userDevPort`, `userDevPort`,`userStatus`, `serialNo`,`portalLink`。其中`portalLink`内容我进行了删减， 其一样是一个json字符串的base64编码后的内容。具体内容是什么不重要，但后续保活需要用到这些内容。



## 心跳包分析

在成功登录之后，我们需要一直保持认证网页的原因是因为，网页会每隔一分钟向`/portal/page/doHeartBeat.jsp`使用`post`方法发送心跳包，因此我们的程序也要隔一定时间进行心跳包的模拟。根据登录成功返回的数据`heartBeatTimeoutMaxTime`我们大概可以得知，心跳包最大的间隔不能超过五分钟。

知道其原理后看一下心跳包的数据。

首先关注一下请求头，有两个数据需要重点关注一下：

第一个是`Cookie`，其需要带着两个值：

```
hello1=学号; hello2=false;
```

第二个是`Referer`，这里就需要用到我们第一步登陆成功返回`portalLink`了。其值为：

```
https://登陆网址/portal/page/online_heartBeat.jsp?pl=pl值&custompath=&uamInitCustom=0&uamInitLogo=H3C
```

然后是FormData：

```
userip: 
basip: 
userDevPort: ZH-Campus-Core_SW-vlan-00-0409@vlan
userStatus: 99
serialNo: 11226
language: Chinese
e_d: 
t: hb
```

这里的`userDevPort`, `userDevPort`,`userStatus`, `serialNo`都是第一步登陆成功返回的内容，但观察后发现其实只需要更改` serialNo`即可。



## 其他问题

至此，登录请求已经分析完毕，剩下的就是用自己喜欢的程序编写相关的程序了。

不过还有一点小问题，比如我使用Go语言的resty包进行http请求的时候，由于坑爹的认证网页倔强地使用https协议，但是证书又是非权威的，因此无法通过tls校验。因此需要大家在编写程序的时候手动跳过tls检查。比如我是用的resty包：

```go
client := resty.New()
client.SetTLSClientConfig(&tls.Config{InsecureSkipVerify: true})
```

目前我已经将我编写的Go版本开源在GitHub上，有需要的朋友可以自取：[传送门](https://github.com/Steve0x2a/JNU_ZH_Network)

有一些功能没有实现，但对于我来说没什么用处，就懒得写了，如果需要的话可以在GitHub提issue。

1. 退出功能（不需要）
2. 验证被顶出自动重连（可以实现，但是无法暂停导致实在需要登出的时候无法登出）




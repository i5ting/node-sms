# node-sms

sms是短消息的意思，这部分主要讲的是如何使用nodejs完成sms相关业务

## 场景

- 验证码
- 找回密码
- 支付密码
- 离线消息提醒
- 变更提醒
- ...

你总能找个理由，必须用sms，因为这样你就可以拿到手机号，各种运营手段，数据价值，此处就不八卦了。

## 技术点

- 发送短信息
- 6位不同的码（大部分都是）
- 多长时间发一次
- 如果同时并发非常高怎么办？
- 提供http接口，便于访问（express封装一下就可以，不必要讲）

## node发短信息

我其实找了很多，有的太烂，有的不敢信，有的界面太丑。。。最后选了luosimao，并写一个node模块

它的好处

- 收到短信时间比较快，按它的模板的特别快，如果是其他的，是要人工审核的
- 价格也还凑合，http://luosimao.com/service/sms#sms-price
- 提供的http接口，非常容易封装调用
- 界面是sms里最不土的


node-sms-luosimao是node发送短信模块，后台服务采用的是luosimao.com的服务

[![npm version](https://badge.fury.io/js/node-sms-luosimao.svg)](http://badge.fury.io/js/node-sms-luosimao)


## install
 
```
npm install --save node-sms-luosimao
```

## demo

 
```
var sms = require("./index");

sms.key = process.env.LSM_KEY

sms.send('18612189310', '测试1~~【node发送短信模块】', function(error, res, body){
  console.log(body);
});
``` 

说明

- sms.key是 https://sms-my.luosimao.com/api 里的API KEY
- 把LSM_KEY放到环境变量里，相对比较安全


## 6位不同的码

OTP全称叫One-time Password,也称动态口令，是根据专门的算法每隔60秒生成一个与时间相关的、不可预测的随机数字组合，每个口令只能使用一次，每天可以产生43200个密码。

它分2钟

- HOTP (counter based one time passwords) 基于一个加法计数器和一个静态的对称密钥
- TOTP (time based one time passwords).（基于时间的一次性密码算法）是支持时间作为动态因素基于HMAC一次性密码算法的扩展。


对于短信码这种需求，哪种都可以的，如果是60s内不允许重新生成，totp就足够了。

下面给出具体实现，参见https://github.com/guyht/notp

```
var notp = require('notp');

var opt = {
	window : 0,
};

var app = {
  encode: function(key) {
    // make sure we can not pass in opt
    return notp.totp.gen(key, opt);
  },
  decode: function(key, token) {
    var login = notp.totp.verify(token, key, opt);
    // invalid token if login is null
    if (!login) {
        console.log('Token invalid');
        return false;
    }

    // valid token
    // console.log('Token valid, sync value is %s', login.delta);
    return true;
  }
}


module.exports = app;
```

在这里我写了一个encode和decode方法，即采用totp加解密，核心参数是key。

那么key怎么能保证唯一呢？其实很简单，我们的业务一般是根据手机号或用户绑定的，所以就很简单了。

## 多长时间发一次

短信发送是按条收费的，不能乱发，那都是钱啊，于是有了各种限制，比如常见的60s内可以再发一次。每次验证码的有效期是10分钟或者其他时间。

那么这些规则怎么实现更好呢？

最简单也是最好的办法是利用redis的expire命令

http://redis.io/commands/expire


expire命令的原理是在redis里设置key的是value，从设置开始，xx秒之后，这个key就会被删除掉

这样做的好处：

- redis是内存数据库非常快
- key过期删除，非常节省


实现如下：

```
var redis = require('redis')
    , client = redis.createClient();
 
client.on('error', function (err) {
    console.log('Error ' + err);
});
 
client.on('connect', function(){
  
});

function cache_expire(k, v){
  console.log('============= cache_expire ==============');
  if(client){
    client.set(k, v, redis.print);
    // Expire in 1*60 seconds
    client.expire(k, 1*60);
  }else{
    console.log('redis client instance is not exist.');
  }
}

/* GET home page. */
router.get('/request_verify_token', function(req, res) {
  var tel = req.param('tel')
  
  var key = tel + '123456789ssdsfx01234567890';

  a = opt.encode(key);
  console.log(a);
  
  //首先检测缓存里是否有tel的key
  client.get(tel, function (err, reply) {
      if(reply) {
          console.log('已存在: 不做任何处理' + reply.toString());
          res.status(200).json({
            code: 1,
            msg:'fail，最近1分钟内会已有申请，不允许重复操作'
          })
      } else {
          console.log('不存在，发送短信');
          
          sms.send(tel,  a + '（xxx验证码），请尽快完成验证。'  + '~~【xxxxx】', function(error, response, body){
            console.log(body);
            if(error){
              res.status(200).json({
                code: 1,
                msg:'sms.send fail'
              })
            }else{
              cache_expire(tel, a);
      
              res.status(200).json({
                code: 0,
                msg:'sms.send sucess'
              })
            }
          });
      }
  });
});

```

说明

- cache_expire是设置redis的key
- request_verify_token是检测缓存里是否有tel的key，如果有就什么也不做，如果没有就发送短信
- 如果需要保存发送历史记录，可以日志，也可以库表保存

其他逻辑和这个类似，自己实现。

## 如果同时并发非常高怎么办？

业务系统和sms系统是分开的，如果并发非常高

1. sms部分做好集群，比如pm2
2. 使用mq（见 [Nodejs消息队列](http://mp.weixin.qq.com/s?__biz=MzAxMTU0NTc4Nw==&mid=222389072&idx=1&sn=c0baf99bda2c74aa8b4fd0e2a2b14096#rd)）

全文完

欢迎提问，欢迎分享

欢迎关注我的公众号【node全栈】  ![](node全栈-公众号.png)


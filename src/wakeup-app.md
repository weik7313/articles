## 需求背景
在推广活动广告中引导用户打开/下载app，终极目标自然是在任何app的webview中都可以正确引导。但是由于手机系统的差距以及微信的存在，使得这件事情并没有想象的那么简单。

## 唤醒实践

由于手机系统区别，本文按照IOS和Android进行分别的讲述

### IOS
##### url schema的方式
这种方式是最容易想到的方式，是两个系统都兼容的方式，也是最方便的，只需要app开发的时候注册scheme，好在我们的app是两端都有统一注册过的 `` tastyfood `` ，直接通过打开src的方式就可以触发app的响应  
    
```
window.location.href = 'tastyfood://wechatx.34580.com'
// or 
<a href="tastyfood://wechatx.34580.com">打开app</a>
```
值得注意的是，ios中``tastyfood://``后的域名从测试的结果来看是不做严格限制，即便是跟上``tastyfood://www.baidu.com`` 也是无碍的，而安卓必须是``tastyfood://wechatx.34580.com``，这应该是app的配置问题。

**兼容问题**：首先这个方案在微信及qq中是被屏蔽的，其次，当用户并未安装app的时候，大部分安卓的外部浏览器（安卓7.0 uc 、Chrome for android）中点击无反应，ios10.1中页面弹出url无效的提示框等。解决方案后面会给出

##### Universal link 方式
官方连接：[https://developer.apple.com/ios/universal-links/](https://developer.apple.com/ios/universal-links/)

``schema``方式的缺点是明显的，为了应对这些问题，苹果在ios9中推出了 ``Universal Links`` 功能

**使用条件**：
1. https的域名
2. ios9以上的系统
3. 名为``apple-app-site-association``的json放置在这个https域名下的根目录或者``.well-known``目录下，知乎就是用的这种方式，有兴趣的同学可以可以访问 [https://oia.zhihu.com/apple-app-site-association](https://oia.zhihu.com/apple-app-site-association) 查看下
4. 其他的部分需要ios开发的同学完成，这里先忽略

ios开发同学的那边配置差不多之后， 在``https://jrd.34580.com``的根目录下放置了一个文件名为 ``apple-app-site-association`` json文件


``` json
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "4NMV7F73W3.com.suiyi.app.dev",
                "paths": [ "/test" ]
            },
            {
                "appID": "ABCD1234.com.apple.wwdc",
                "paths": [ "*" ]
            }
        ]
    }
}
```
直接在页面中通过下面的方式即可成功打开，由于是系统级别的处理，微信并不能屏蔽掉这种方式，并且可以自动跳转app store 可以说除了系统要求的问题，没有缺点的方式

```
window.location.href = 'https://jrd.34580.com/test'
``` 

注意点：
1. 直接打开这个地址是不行的，一定要在页面中通过跳转此链接的方式，ios系统才会处理
2. 此https的域名必须和当前访问的页面url形成跨域才能响应，对，是跨域的情况
3. 需要校验``apple-app-site-association``的话，可以通过官方的 [校验地址](https://search.developer.apple.com/appsearch-validation-tool/) 进行
### Android

##### url schema的方式
和ios的使用方式一致

##### Android intent的方式

官方连接[https://developer.chrome.com/multidevice/android/intents](https://developer.chrome.com/multidevice/android/intents)

语法
```
intent:
   HOST/URI-path // Optional host 
   #Intent; 
      package=[string]; 
      action=[string]; 
      category=[string]; 
      component=[string]; 
      scheme=[string]; 
   end; 
```

使用方式

```
 <a href="intent://wehchatx.34580.com//#Intent;scheme=tastyfood;package=com.suiyi.client.android;end">打开</a>
```
这种方式用户安装了app，直接打开，如果没有，则会跳转商城，原生安卓是跳转Google play商城，国内的定制安卓会跳转系统自带的商城，体验也是值得肯定的。
另外，这种方式可以配置一个链接，可以在打开失败的时候跳转
```
<a href="intent://wehchatx.34580.com//#Intent;scheme=tastyfood;package=com.suiyi.client.android;S.browser_fallback_url=[https://download.34580.com；end">打不开回调</a>
```

##### App Links 方式
ios有了Universal link 安卓就不能有相同的方式吗？答案是有的，在Android M系统之后，可以使用，或者也可以称之为 ``Deep Links``
具体的配置大部分都需要app的开发同学来完成 [官方参考链接](https://developer.android.com/training/app-links/deep-linking.html#adding-filters)
但是这种方式要求的是安卓6.0，版本要求过高，暂时不考虑。

### 兼容处理
针对url schema的方式，用户未下载app的时候的无反应情况，或者打开失败，需要特别的做处理，提升体验。

通常我们都是通过打开一个链接去处理，这样势必会出现链接错误的情况下，跳转了错误的页面。那么能不能不跳转呢？ 下意识的我们就会想到用iframe来处理，通常会这么做：

```
const ifr = document.createElement('iframe')
ifr.src = 'tastyfood://wechatx.34580.com'
ifr.style.display = 'none'
document.body.appendChild(ifr)

const startTime = new Date()
window.setTimeout(function(){
  document.body.removeChild(ifr)
  if( ((new Date()) - startTime) > 2500 ){
    window.location = 'https://down.34580.com'
  }
}, 2000)
```

这种方案使用了iframe的方式，根据chrome的官方说明
> The functionality has changed slightly in Chrome for Android, versions 25 and later. It is no longer possible to launch an Android app by setting an iframe's src attribute.

Chrome for Android 25版本之后，可能并不试用iframe的方式，以及ios9中iframe也并不能使用。所以这个方案暂时不能用，或者至少需要加额外的兼容,所以我尝试了下面的方式

```
openApp () {
  // schema的方式打开
  window.location.href = 'tastyfood://wechatx.34580.com'
    
  // 默认往下载页跳转（知乎目前也是这样的跳转逻辑）
  window.setTimeout(() => {
    window.location.href = 'https://down.34580.com'
  }, 250)

  window.setTimeout(() => {
    window.location.reload()
  }, 1000)
}
```

### 最后
打开app的操作还是需要做一些兼容的，不然的话唤醒的成功率还是很低的。另外，微信里的话 除了Universal link ，基本上只能用应用宝了，没办法，毕竟腾讯。 另外本文未谈及的唤醒某个详情页，大致上使用url router匹配即可，这里就不讲了。







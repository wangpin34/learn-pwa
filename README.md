# Learn progressive web application

pwa 的核心是两个关键技术： cache 和 service worker（服务工作线程）。通过 service worker 管理 cache，缺一不可。

## cache


## service worker
service worker 是可以在后台独立于网页运行的 js 脚本。具备以下几个**关键**特性。
* 离线运行
* 网络代理，能够控制发送网络请求的处理方式
* 不能访问 dom
* 支持 promise 编程

利用 service worker，可以自如的管理缓存，从而获得媲美原生 app 的**加载**体验。并且，因为它可以离线运行，所以，原本离线状态下一片空白的 web app 也可以选择性的展示缓存内容。这是以往的 web app 或者网站绝对无法实现的功能。

service worker 必须在 https 环境下才可以使用。为了调试方便，本地开发时允许在 http 下使用。使用 service worker 前，必须先注册：
```javascript
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/sw.js').then(function(registration) {
      // Registration was successful
      console.log('ServiceWorker registration successful with scope: ', registration.scope);
    }).catch(function(err) {
      // registration failed :(
      console.log('ServiceWorker registration failed: ', err);
    });
  });
}
```


注册完成后，service worker 脚本（如上，/sw.js）开始安装。以下是几个关键的 event:
### install - 安装完成
在 install 回调的内部，要完成三项工作：
1.打开缓存。
2.缓存文件。
3.确认所有需要的资产是否缓存。

```javascript
var CACHE_NAME = 'my-site-cache-v1';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', function(event) {
  // Perform install steps
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
```
### fetch - 网络请求
在这个回调中代理网络请求
```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // Cache hit - return response
        if (response) {
          return response;
        }
        return fetch(event.request);
      }
    )
  );
});
```
上面的代码，代理了ui线程的网络请求，检查缓存中是否有对应内容，如果有，则直接返回内容；如果无，则转发到后台服务。

如果想要在请求后台服务成功以后，更新 app 对应缓存，则应该检测返回信息，如果 http code 是 200（ok），则替换对应缓存。

```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // Cache hit - return response
        if (response) {
          return response;
        }

        // IMPORTANT: Clone the request. A request is a stream and
        // can only be consumed once. Since we are consuming this
        // once by cache and once by the browser for fetch, we need
        // to clone the response.
        var fetchRequest = event.request.clone();

        return fetch(fetchRequest).then(
          function(response) {
            // Check if we received a valid response
            if(!response || response.status !== 200 || response.type !== 'basic') {
              return response;
            }

            // IMPORTANT: Clone the response. A response is a stream
            // and because we want the browser to consume the response
            // as well as the cache consuming the response, we need
            // to clone it so we have two streams.
            var responseToCache = response.clone();

            caches.open(CACHE_NAME)
              .then(function(cache) {
                cache.put(event.request, responseToCache);
              });

            return response;
          }
        );
      })
    );
});
```
### activate
如果后台更新了 app 的 service worker 脚本，则前台也需要以下流程来替换已有的 service worker 线程。
> 用户导航至您的站点时，浏览器会尝试在后台重新下载定义服务工作线程的脚本文件。如果服务工作线程文件与其当前所用文件存在字节差异，则将其视为“新服务工作线程”。
新服务工作线程将会启动，且将会触发 install 事件。
此时，旧服务工作线程仍控制着当前页面，因此新服务工作线程将进入 waiting 状态。
当网站上当前打开的页面关闭时，旧服务工作线程将会被终止，新服务工作线程将会取得控制权。
新服务工作线程取得控制权后，将会触发其 activate 事件。

```javascript
self.addEventListener('activate', function(event) {

  var cacheWhitelist = ['pages-cache-v1', 'blog-posts-cache-v1'];

  event.waitUntil(
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.map(function(cacheName) {
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```


## 参考文档
* [service workers](https://developers.google.com/web/fundamentals/primers/service-workers/)
* [codelabs:your first pwapp](https://codelabs.developers.google.com/codelabs/your-first-pwapp)

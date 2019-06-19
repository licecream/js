# ServiceWorker：BackgroundSync的基本指南


原文链接：https://ponyfoo.com/articles/backgroundsync 
翻译：刘晓倩



*想象一下场景。您正在使用移动设备浏览网站并预订一些重要的音乐会门票。几个月来，你一直在热切期待这场音乐会。您可以直接进入结账屏幕，输入您的信用卡详细信息并点击提交。然后你的片状3G连接下降。尖叫和痛苦随之而来！*

*我非常渴望分享将在Pony Foo上发布的许多客座帖子中的第一篇！* ![qingzhu.md](https://twemoji.maxcdn.com/2/72x72/1f389.png "qingzhu.md")

当Dean带着想在ServiceWorker上撰写文章的想法来找我时，我立即对他提出的关于BackgroundSync的建议表示激动。在之前的serviceworker文章中，我还没有完全涵盖这一领域，他将有关BackgroundSync是什么，它如何工作以及为什么它有用的文章。

我甚至无法开始告诉您我尝试提交表单或完成操作的次数，仅用于我连接到drop和浏览器以向我显示离线页面。到目前为止，我们一直无法应对这些情况，这对我们的用户来说是一次非常糟糕的体验。幸运的是，这是服务工作者可以拯救的地方。我之前曾写过关于使用[Service Workers](https://ponyfoo.com/articles/tagged/serviceworker )来提供[离线功能的文章](http://deanhume.com/create-a-really-really-simple-offline-page-using-service-workers/)，并使用缓存大大加快了加载时间。虽然这很有用，但是如果没有连接，页面就无法实际向服务器发送内容。后台同步是为了处理这种情况而构建的功能。

![背景同步徽标.md](https://dl.dropboxusercontent.com/u/909725/SW/background-sync-logo.jpg "背景同步徽标.md")
>背景同步徽标

那背景同步到底是什么？好吧，它是一个新的Web API，可让您推迟操作，直到用户具有稳定的连接。这使得它确保无论用户想要发送什么，实际上都是发送的。用户可以离线甚至关闭浏览器，安全地知道他们的请求将在重新获得连接时发送。

在本文中，我将通过一个简单的示例演示如何使用[后台同步](https://github.com/WICG/BackgroundSync/blob/master/explainer.md)来确保即使用户处于脱机状态时您的请求也会排队。我们将看一个发出HTTP请求的基本页面，并根据用户是否连接而做出不同的响应。

为了开始这个例子，你需要对服务工作者有一个基本的了解，但是如果你不熟悉它们也不要担心 - 网上有一些很好的资源。我建议您查看[slightlyoff/ServiceWorker](https://ponyfoo.com/articles/tagged/serviceworker )更多信息。

## 入门

想象一下，您有以下HTML页面。这是非常基本的，但它给你一般的想法。

```html
<!DOCTYPE html>
<html>
 <head>
  <meta charset="UTF-8">
  <title>Home Page</title>
 </head>
 <body>
  <div id="connectionStatus"></div>
  <p>Click on the button below to make an HTTP request.<p>
  <button id="requestButton">Make an HTTP request - do it!</button>
 </body>
</html>
```
上面的HTML显示了一个简单的按钮，可以在单击时为图像发出HTTP请求。如果网页有连接，HTTP请求应该没有任何问题。但是，如果没有连接，则只要用户再次建立连接，HTTP请求就会排队并同步。

为了使用后台同步，我们需要创建一个Service Worker来解锁此功能。让我们首先在我们刚刚创建的页面中注册Service Worker。将以下代码添加到HTML中。

```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker
    .register('./service-worker.js')
    .then(registration => navigator.serviceWorker.ready)
    .then(registration => { // register sync
      document.getElementById('requestButton').addEventListener('click', () => {
        registration.sync.register('image-fetch').then(() => {
            console.log('Sync registered');
        });
      });
    });
} else {
  document.getElementById('requestButton').addEventListener('click', () => {
    console.log('Fallback to fetch the image as usual');
  });
}
```

上面的代码起初可能看起来有点毛茸茸，但让我们分解它。首先，我正在做一个简单的检查，看看浏览器是否支持Service Workers。如果是，我会注册一个名为**service-worker.js**的文件。此文件将包含具有所有Background Sync魔法的Service Worker代码。我们将很快创建这个。您可能还注意到，如果浏览器不支持Service Workers，则代码会简单地退回并在不使用后台同步的情况下正常发出HTTP请求。

接下来，如果注册成功，我将向该按钮添加一个click事件并注册一个名为image-fetch的同步。这只是一个我命名的简单字符串，以帮助我识别此事件。您可以将这些同步名称视为不同操作的简单标记 - 您可以拥有任意数量的标记！




## 服务工作者

接下来，我们需要创建Service Worker文件并将其命名为“service-worker.js”。我们将使用此Service Worker为简单映像发出HTTP请求，并在成功时返回它。

一旦我创建了service-worker.js文件，我将添加以下代码：

```javascript
self.addEventListener('sync', function (event) {
  if (event.tag === 'image-fetch') {
    event.waitUntil(fetchDogImage());
  }
});
```
上面的代码创建了一个侦听sync事件的事件。如果他们的页面有连接，它将尝试获取图像，如果它满足，则同步完成。如果失败，将安排另一个同步重试。重试同步也等待连接，并采用指数退避。

接下来，我需要添加一个获取狗图像的功能。

```javascript
function fetchDogImage () {
    fetch('./doge.png')
      .then(function (response) {
        return response;
      })
      .then(function (text) {
        console.log('Request successful', text);
      })
      .catch(function (error) {
        console.log('Request failed', error);
      });
  }
```
上面的代码使用Fetch API从服务器检索图像。它将返回一个承诺，指示它是否成功或者是否因HTTP响应而失败。

就是这样！您现在可以使用BackgroundSync启用将根据连接排队的脱机操作。

##在野外测试

现在您已准备好页面以同步事件，您可以对其进行测试。如果你想看到这个例子的实际应用，我已经创建了一个可以用来试验的GitHub页面。转到[演示页面](https://deanhume.github.io/Service-Workers-BackgroundSync/)了解更多信息。

当您具有活动连接时，您应该会在开发人员工具的“ 网络”选项卡中看到网络请求。

![BackgroundSync在线服务工作者.md](https://dl.dropboxusercontent.com/u/909725/SW/background-sync-logo.jpg "BackgroundSync在线服务工作者.md")
>BackgroundSync在线服务工作者
>如果您未连接到互联网，则当用户重新获得连接时，您的请求将被同步。为了测试这个，我只是禁用了wifi，发出了HTTP请求，然后重新启用了wifi。

![BackgroundSync脱机服务工作者.md](https://dl.dropboxusercontent.com/u/909725/SW/background-sync-logo.jpg "BackgroundSync脱机服务工作者.md")
>BackgroundSync脱机服务工作者
>只要重新启用wifi连接，就会看到HTTP请求正常显示在网络选项卡中。

我在测试时注意到了同步行为的一些事情。首先，同步的标签名称对于给定的同步应该是唯一的。如果使用与待处理同步相同的标记注册同步，则它将与现有同步结合使用。这意味着，对于此示例，如果您在每次用户单击按钮时注册“图像获取”同步，则只有在再次连接时才会收到一个同步。如果您想要5个单独的同步事件，则需要使用唯一标记。

##通知用户

我们经历的例子不是很漂亮，并没有为用户提供很多反馈。如果用户在查看页面时收到关于其连接的当前状态的反馈，那将是更好的体验。

使用一些聪明的代码，我可以听取用户连接的任何变化。

```javascript
// Connection Status
function isOnline () {
  var connectionStatus = document.getElementById('connectionStatus');

  if (navigator.onLine){
    connectionStatus.innerHTML = 'You are currently online!';
  } else {
    connectionStatus.innerHTML = 'You are currently offline. Any requests made will be queued and synced as soon as you are connected again.';
  }
}

window.addEventListener('online', isOnline);
window.addEventListener('offline', isOnline);
isOnline();
```
在上面的代码中，我在window对象上添加了一个事件监听器来监听连接的变化。一旦此连接状态发生更改，我将根据结果更新包含通知消息的页面。通过将上述代码添加到我们的网页，我们可以在每次连接状态更改时更新用户。

上面的代码不是防弹的 -[navigator.onLine](https://developer.mozilla.org/en-US/docs/Web/API/NavigatorOnLine/onLine )只知道用户是否无法连接到LAN或路由器 - 它不知道用户是否可以实际连接到互联网。也就是说，我仍然认为，如果用户在没有连接的情况下尝试执行操作时向用户提供一些反馈会很有用！

##未来

后台同步还有更多功能！处理Background Sync的团队目前正致力于[定期背景同步](https://developers.google.com/web/updates/2015/12/background-sync?hl=en#the-future )。这将允许您请求将受时间，电池状态和网络状态限制的“周期性同步”事件。API仍在设计中，但它可能看起来像下面的代码：

```javascript
navigator.serviceWorker.ready.then(function(registration) {
  registration.periodicSync.register({
    tag: 'get-latest-news',         // default: ''
    minPeriod: 12 * 60 * 60 * 1000, // default: 0
    powerState: 'avoid-draining',   // default: 'auto'
    networkState: 'avoid-cellular'  // default: 'online'
  }).then(function(periodicSyncReg) {
    // success
  }, function() {
    // failure
  })
});
```
上面的代码非常类似于标准的Background Sync，只是它有一些参数。该**minPeriod**成功同步事件之间的最小时间，**电源状态**可以在电池充电时可以用来仅同步，并**networkState**让您是否希望通过WiFi与蜂窝连接同步控制。很酷！

关于定期同步的好处在于它们不需要任何服务器配置，并且允许浏览器在它们触发时进行优化，以便对用户最有帮助和最小的破坏性。所有这些都可以在客户端上完成，无需任何服务器配置！我已经可以想到这个功能的强大用途。

API仍在进行中，但如果您想了解最新的更改，我建议您密切关注[WICG后台同步规范](https://github.com/WICG/BackgroundSync/blob/master/explainer.md#periodic-synchronization-in-design)。

##摘要
作为用户，当您没有连接时无法执行操作可能会非常令人沮丧。幸运的是，Background Sync带来的功能是游戏规则改变者。我对他们带给网络的可怕可能性感到兴奋。

如果您希望看到本文的实际示例，请转到[演示页面](https://deanhume.github.io/Service-Workers-BackgroundSync/)。这背后的所有代码都可以在[deanhume/Service-Workers-BackgroundSync](https://github.com/deanhume/Service-Workers-BackgroundSync)。
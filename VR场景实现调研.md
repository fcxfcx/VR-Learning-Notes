# VR场景实现调研

针对网页端使用aframe框架的方案，进行VR场景实现相关的调研

## 1. 双人VR保龄球demo

项目教程地址：

https://www.pubnub.com/blog/build-multiplayer-browser-based-vr-game-aframe-webvr/?utm_source=Syndication&utm_medium=Medium&utm_campaign=SYN-CY18-Q3-Medium-Aug-17

项目演示网址：

https://youtu.be/uwmI_TXVeYE

该demo是利用aframe制作的双人保龄球游戏，其中物理引擎和画面采用aframe框架，游戏玩家之间的通信采取了pubnub工具作为实时通信框架。

![Click here to watch the game](https://github.com/namrathasubramanya/VR-Bowling-game/raw/master/IMG_1161.PNG)

一些可以学习的要点：

- **关于物理引擎：**A-frame提供了名为`aframe-physics-system`的中间件作为物理引擎支持，可以使用其中的`static-body`或者`dynamic-body`组件进行属性配置， `aframe-physics-system`会创建一个`Cannon.Body`实例并将其附加到我们的A-Frame实体上，因此在每个框架上，它都会调整实体的位置，旋转等。
- **关于固定动作：**该demo中固定了保龄球可移动的五条轨道（图片中的五个箭头），该过程是通过`<a-animation>`标签完成的，即在点击某个箭头后调用对应的函数，在函数中添加一个`<a-animation>`标签，并定义一个从指定坐标向指定坐标前进的动作（使用to, from, dur等属性），然后使用`appendChild()`方法将这个动作添加到保龄球实体上作为子标签，就实现了保龄球移动的效果。
- **关于计分：**demo中为了计算得分，会追踪每个保龄球瓶的x轴旋转值，如果大于零则认为是撞倒，并且遍历所有的瓶子以计算得分
- **动作侦测和反应：**所有点击的操作通过`addEventListener()`函数进行action的绑定，包括重新开始游戏、点击箭头放出保龄球
- **出球后锁定操作：**在一方发出保龄球或者说未到自己的回合的时候，会将所有箭头实体的位置置于原点（相当于隐藏了这些对象），以避免在一段动画未结束的时候重复操作
- **关于两个玩家的同步：**在一位玩家完成操作并执行完动画后，会将保龄球瓶的位置信息（包括rotation和position）作为参数使用pubnub框架publish到频道中，该参数作为message的一部分传输，message中还包括一个key作为传递消息的类别（例如是保龄球的动画还是保龄球瓶的位置信息）。在另一位玩家处会通过key首先判断需要更新哪部分实体（做一个if判断，不同的key对应不同的操作），然后再根据message中的内容进行更新
- **其他细节：**在未到自己操作的回合，本用户画面应该是复刻另一位用户的画面，因此需要将未操作的用户画面中的实体转化为static状态，避免其发生物理碰撞（尽管这可能会带来穿模等现象），具体的操作是通过维护一个`dynamics`变量，在需要物理碰撞的时候设置为`true`而在不需要的时候设置为`false`，并且在需要的时候，设置碰撞体的具体质量（mass属性）

此外， 虽然该项目使用了pubnub作为网络通信的框架，但该部分是完全可替代的，主要关注aframe在处理多人VR场景下的基本思路。

## 2. A-frame提供的多人框架

  A-frame本身包含了一个为多人VR应用的组件，名为networked-Aframe（来源：A-frame官方文档）

项目地址：[networked-aframe/networked-aframe: A web framework for building multi-user virtual reality experiences. (github.com)](https://github.com/networked-aframe/networked-aframe)

它的特性包括：

- 支持WEBRTC和/或WebSocket连接。
- 语音聊天。音频流以使您的用户在应用程序内谈论（仅WEBRTC）。
- 视频聊天。（请参阅视频流中应用程序）
- 带宽敏感。仅在情况发生变化时发送网络更新。
- 跨平台。在所有现代台式机和移动浏览器上都可以使用。Oculus Rift，Oculus Quest，HTC Vive和Google Cardboard。
- 可扩展。同步任何Aframe组件，包括您自己的组件，根本不更改组件代码。



对于该组件，从展示层面来说已经满足了基本的浏览器端VR场景的渲染，也提供了基本的音视频通话的demo，如果要在这方面去做进一步的优化，那就是需要对建模、3D场景等的美化，然而感觉这方面并不是目前调研和项目的重点。因此可以先保留最原始的vr场景，优先弄清楚这几个方面：

- NAF（networked-Aframe）在音视频通话上使用了esayRTC，具体实现上是怎么做的？
- easyRTC是否默认使用了p2p架构，如何更换成MCU或者SFU，并集成到NAF中使用？
- 对于entity的同步，在用法上已经包装成了直接添加一个`networked`component，在底层实现上是怎么做到的，能否拓展或者更改其中的方式？
- aframe对于硬件api的支持只介绍了手柄和头盔，对于力反馈设备的支持并不在列，如果需要对接力反馈设备，需要做哪些事情？（等需求明确和硬件到位）



**关于easyrtc**

首先，NAF在设置网络连接的相关配置的时候，需要在`<a-scene>`标签中添加networked-scene，并在其中配置相关的属性，示例如下：

```javascript
<a-scene networked-scene="
  serverURL: /;
  app: <appId>;
  room: <roomName>;
  connectOnLoad: true;
  onConnect: onConnect;
  adapter: wseasyrtc;
  audio: false;
  video: false;
  debug: false;
">
  ...
</a-scene>
```

其中有一个名为adapter的属性，就是用来配置使用哪种网络库作为底层框架，这里选用的是wseasyetc，其余的可选项如下所示：

| Adapter   | Description                                                  | Supports Audio/Video                      | WebSockets or WebRTC | How to start                                                 |
| --------- | ------------------------------------------------------------ | ----------------------------------------- | -------------------- | ------------------------------------------------------------ |
| wseasyrtc | DEFAULT - Uses the [open-easyrtc](https://github.com/open-easyrtc/open-easyrtc) library | No                                        | WebSockets           | `npm run dev`                                                |
| easyrtc   | Uses the [open-easyrtc](https://github.com/open-easyrtc/open-easyrtc) library | Audio and Video (camera and screen share) | WebRTC               | `npm run dev`                                                |
| janus     | Uses the [Janus WebRTC server](https://github.com/meetecho/janus-gateway) and [janus-plugin-sfu](https://github.com/networked-aframe/janus-plugin-sfu) | Audio and Video (camera OR screen share)  | WebRTC               | See [naf-janus-adapter](https://github.com/networked-aframe/naf-janus-adapter/tree/3.0.x) |
| socketio  | SocketIO implementation without external library (work in progress, currently no maintainer) | No                                        | WebSockets           | `npm run dev-socketio`                                       |
| webrtc    | Native WebRTC implementation without external library (work in progress, currently no maintainer) | Audio                                     | WebRTC               | `npm run dev-socketio`                                       |
| Firebase  | [Firebase](https://firebase.google.com/) for WebRTC signalling (currently no maintainer) | No                                        | WebRTC               | See [naf-firebase-adapter](https://github.com/networked-aframe/naf-firebase-adapter) |
| uWS       | Implementation of [uWebSockets](https://github.com/uNetworking/uWebSockets) (currently no maintainer) | No                                        | WebSockets           | See [naf-uws-adapter](https://github.com/networked-aframe/naf-uws-adapter) |

这里我们重点关注的是easyrtc的架构，因为它是支持音视频的，这里使用的是open-easyrtc的库，库的地址附在了表中

更多的adapter选择在：[NAF adapters comparison · networked-aframe/networked-aframe Wiki (github.com)](https://github.com/networked-aframe/networked-aframe/wiki/NAF-adapters-comparison)

另外，上面链接中的文章提到，目前naf并不支持MCU架构，这是因为MCU是不支持空间音频的。

## 3.NAF是如何同步场景的

在这里记录一下读NAF源码的学习路径：

### 3.1 从example说起

为了深入弄清楚NAF其中的工作逻辑，我们从官方给出的demo入手，去分析NAF内部的调用逻辑。首先如果我们跟着start up教程去学习的话，会发现它的构造方式是从头开始的（从头构建node.js项目并自定义package.json文件），之后将整个networked aframe中的用于例子展示的相关文件夹提取了出来，包括：

- example文件夹：装有承载所有demo所需的前端文件
- server文件夹：装有启动easyrtc或者socketio服务器的代码
- dist文件夹：内部装有NAF的整个库（networked-aframe.min.js）

实际上原本的example文件夹内是不含dist文件夹的，教程中将其复制到了example中，就跳过了build的过程，使得demo页面可以即插即用，因为dist中已经含有了打包好的NAF库。在`package.json`文件中定义了启动demo展示页面的script，如下：

```javascript
  "scripts": {
    "start": "node ./server/easyrtc-server.js"
  },
```

这里明显可以看出项目的入口是从easyetc-server.js开始的，而在该文件中，可以得知主要是一个建立本地服务器并监听端口的过程，这里包括了几个服务（实际上这个文件和open-easyrtc官方库里提供的server example中的文件是一致的）：

- express服务：维护并提供静态资源（demo的各种html文件），根目录定为example目录
- socket服务：利用socket.io建立socket服务
- easyrtc服务：最终的目的，建立easyrtc服务器

这里的express服务很好理解，因为要承载应用本身的网页内容，而socket服务是建立easyrtc服务器的必要条件，作为easyrtc服务器启动（listen）方法的参数传入，作为建立链接的时候的传输通道。但是整个这部分只是构建了音视频传输的工具架构，NAF内部的组件同步并不在这部分展现。

### 3.2 对webpack的使用

在easyrtc-server.js中我们可以看到有使用webpack相关的步骤，而在完全版本的NAF项目中我们也可以在package.json中找到和打包项目相关的script：

```javascript
  "scripts": {
    ...
    "dist:max": "webpack --config webpack.config.js",
    "dist:min": "webpack --config webpack.prod.config.js",
    ...
  },
```

而在webpack.config.js中我们也可以看到，打包好的文件被放置于dist目录下：

```javascript
const path = require("path");

module.exports = {
    entry  : './src/index.js',
    output : {
        path     : path.resolve(__dirname, 'dist'),
        publicPath: '/dist/',
        filename : 'networked-aframe.js'
    },
    mode: 'development',
    module : {
        rules: [
            {
                test: /\.js$/,
                use: {
                    loader: 'babel-loader',
                    options: {
                      presets: ['@babel/preset-env']
                    }
                }
            }
        ]
    }
};
```

webpack的作用是从一个或多个入口构建一个依赖图，然后将你项目中所需的每一个模块组合成一个或多个 *bundles*，它们均为静态资源，用于展示你的内容（webpack文档地址：[概念 | webpack 中文文档 (docschina.org)](https://webpack.docschina.org/concepts/)），这里很明显入口是src路径下的index.js，进入到该文件查看，会发现它定义了整个NAF所需要的依赖结构：

```javascript
// Global vars and functions
require('./NafIndex.js');

// Network components
require('./components/networked-scene');
require('./components/networked');
require('./components/networked-audio-source');
require('./components/networked-video-source');
```

其中第一个依赖是整个NAF中需要的变量和方法的入口，我们之后再细看，而后面的四个是对应着Aframe中的一个重要概念，**Component（组件）**，而这也是NAF对外暴露API的直接入口，因此我们先讨论component在NAF中起到的作用。（参考：[Component – A-Frame (aframe.io)](https://aframe.io/docs/1.2.0/core/component.html#component-html-form)）

### 3.3 认识AFrame中的Component

在探究NAF中组件的结构之前，我们需要认识Component在AFrame中起到了什么样的作用。作为一个Entity-Component系统，AFrame使用entity作为空间中的实体，组件则是我们插入实体以添加外观、行为和/或功能的可重用模块化数据块。举一个例子，如果我们将智能手机定义为一个实体，我们可能会使用组件赋予其外观（颜色、形状），定义其行为（通话时振动，电池电量低时关机），或添加功能（摄像头、屏幕）

一个组件会用一个或多个组件属性来表示一批数据，在实际使用的时候，单个的组件属性会类似一个HTML attribute出现在html文件中，而多个的组件属性则更像是CSS style表达，如下所示：

```html
<!-- `position` is the name of the position component. -->
<!-- `1 2 3` is the data of the position component. -->
<a-entity position="1 2 3"></a-entity>

<!-- `light` is the name of the light component. -->
<!-- The `type` property of the light is set to `point`. -->
<!-- The `color` property of the light is set to `crimson`. -->
<a-entity light="type: point; color: crimson"></a-entity>
```

组件提供给开发者一个非常方便的定义空间中实体属性的接口，而如果我们要自定义一个新的组件，则需要通过`registerComponent(name, definition)`方法来注册组件，这个步骤必须在建立`<a-scene>`并使用它们之前，在注册组件时，除了可以定义组件内的数据结构，例如包含哪些属性，每个属性的数据类型是怎样的，每个属性的默认值是怎样的。这些数据统一在schema对象中被声明，以键值对的方式去储存，示例如下：

```javascript
AFRAME.registerComponent('bar', {
  schema: {
    color: {default: '#FFF'},
    size: {type: 'int', default: 5}
  }
}

<!-- when use this component>
<a-scene>
  <a-entity bar="color: red; size: 20"></a-entity>
</a-scene>                     
```

除了属性，还有一个重要的步骤是注册组件时可以定义其生命周期内进行的操作，AFrame提供了多个生命周期的触发点，重要的包括：

- init：在组件初始化时调用一次。用于设置初始状态和实例化变量。
- update：在组件初始化和组件属性更新时调用(例如，通过setAttribute)。用于修改实体。
- remove：当组件从实体中移除(例如，通过removeAttribute)或实体从场景中分离时调用。用于撤消以前对实体的所有修改。
- tick：在场景的每次渲染循环或tick时调用。用于连续的变化或检查。
- tock：在场景渲染完成后，在每次渲染循环或tick时调用。用于后期处理效果或其他需要在场景绘制后发生的逻辑。

这个时候我们就可以回到代码去看NAF具体注册了哪些组件，以networked-scene组件为例，从使用上来说，我们知道这个组件是装载于a-scene标签上的，是对整个AFrame场景的设置，而我们熟悉的对于整个NAF的同步设置就在此处（例如最关心的adapter），而在注册时的定义如下所示：

```javascript
  schema: {
    serverURL: {default: '/'},
    app: {default: 'default'},
    room: {default: 'default'},
    connectOnLoad: {default: true},
    onConnect: {default: 'onConnect'},
    adapter: {default: 'wseasyrtc'}, // See https://github.com/networked-aframe/networked-aframe#adapters for list of adapters
    audio: {default: false}, // Only if adapter supports audio
    video: {default: false}, // Only if adapter supports video
    debug: {default: false},
  },
```

而后面该组件主要定义了在初始化阶段，init函数中的操作，具体如下：

```javascript
  init: function() {
    var el = this.el;
    this.connect = this.connect.bind(this);
    el.addEventListener('connect', this.connect);
    if (this.data.connectOnLoad) {
      el.emit('connect', null, false);
    }
  },
```

可以看到这里主要做了两个操作，第一个是将connect事件绑定到connect方法上，第二个是如果在networked-scene组件中设置connectOnLoad（这样只要页面开始加载就自动链接）为true，那么就触发connect事件，也就是开始链接。那么connect方法中具体做了哪些事呢？

```javascript
  /**
   * Connect to signalling server and begin connecting to other clients
   */
  connect: function () {
    NAF.log.setDebug(this.data.debug);
    NAF.log.write('Networked-Aframe Connecting...');

    this.checkDeprecatedProperties();
    this.setupNetworkAdapter();

    if (this.hasOnConnectFunction()) {
      this.callOnConnect();
    }
    return NAF.connection.connect(this.data.serverURL, this.data.app, this.data.room, this.data.audio, this.data.video);
  },
```

首先，方法将整个NAF应用的log设置好，并输出一个log提示正在链接。注意，这里直接用了`NAF.log`是因为它是一个全局变量，这个类是在`NAFIndex.js`中导出的，其中包含了NAF所需要的很多工具类，以方便在其他的js中直接调用方法。之后的`checkDeprecatedProperties()`目前是空方法，可以先不管，然后`setupNetworkAdapter()`方法是将选中的adapter设置到NAF中，代码如下：

```javascript
  setupNetworkAdapter: function() {
    var adapterName = this.data.adapter;
    var adapter = NAF.adapters.make(adapterName);
    NAF.connection.setNetworkAdapter(adapter);
    this.el.emit('adapter-ready', adapter, false);
  },

```

这里具体涉及到两步：

- 从AdapterFactory中取出对应的adapter类（属性的字符串→实际使用的类），具体的代码在`src/adapters/AdapterFactory.js`下
- 将得到的`adapter`类作为参数设置到网络连接类中去，这里的设置方法是放在`NAF.connection`中的，查看`NAFIndex.js`中的require语句可以得知对应的js文件是`src/NetworkConnection.js`，设置完毕后触发`adapter-ready`事件

设置好adapter后，`connect`方法会判断是否组件内是否含有onConnect方法（同样是在之前传入的，而且有默认值），如果有的话就触发这个方法，在`NAF.connection`类中去处理连接事件，最后调用`NAF.connection.connect()`方法进行实际的链接。种种迹象表明，我们需要进一步深入`NetworkConnection.js`去看一看具体发生了什么事情。

这里只是举了一个例子去说明component在NAF中起到的作用，可以想到，当我们将`networked-scene`组件添加到`<a-scene>`中后，整个场景在初始化的时候就会进行init方法中的操作，从而调动到NAF内部的连接逻辑，而对于其他的组件来说，从原理上讲它们是一致的，只是各自负责的entity类型和功能不同，分别是：

- networked：附加在一般的entity上，作为需要同步的标识
- networked-audio-source：附加在entity上表示音频流会从该entity的拥有者处上传
- networked-video-source：同上，附加在entity上表示视频流会从该entity的拥有者处上传

这些组件在之后我们再详细的去查看。

### 3.4 连接的时候发生了什么

在进一步细看所有组件之前，以我们举例的`networked-scene`组件入手，我们先进一步的进入`NetworkConnection.js`去看看刚才没有细看的几个方法。

按照调用的顺序，首先关于设置adapter的方法，此方法很简单，只是将该类中的adapter变量赋值：

```javascript
  setNetworkAdapter(adapter) {
    this.adapter = adapter;
  }
```

然后是在networked-scene组件中会判断设置的"onConnect"属性是否有值，这里其实默认是有值的（即”onConnect“），但是由于默认值的函数并不存在，所以如果要自定义的话，则需要自己定义好对应的函数，在networked-scene组件中会调用`hasOnConnectFunction()`方法来判断是否存在这个函数，如果有就将其传入NetworkConnection类中的`onConnect()`方法中，该方法接收一个callback function作为输入，如果已经连接（根据是否含有clientID来判断），则直接触发这个函数，否则的话会将这个callback函数绑定到名为"connected"的事件上去，这一步的操作是在实际调用NetworkConnection中的`connect()`方法前执行的，以便在连接成功后能正确的触发这个callback，代码如下所示：

```javascript
  onConnect(callback) {
    this.onConnectCallback = callback;

    if (this.isConnected()) {
      callback();
    } else {
      document.body.addEventListener('connected', callback, false);
    }
  }

// 通过判断clientID是否存在来判断是否连接成功
  isConnected() {
    return !!NAF.clientId;
  }
```

了解完这个前置工作之后，我们可以进一步去看NetworkConnection类中的`connect()`方法具体干了什么：

```javascript
  connect(serverUrl, appName, roomName, enableAudio = false, enableVideo = false) {
    NAF.app = appName;
    NAF.room = roomName;

    this.adapter.setServerUrl(serverUrl);
    this.adapter.setApp(appName);
    this.adapter.setRoom(roomName);

    var webrtcOptions = {
      audio: enableAudio,
      video: enableVideo,
      datachannel: true
    };
    this.adapter.setWebRtcOptions(webrtcOptions);

    this.adapter.setServerConnectListeners(
      this.connectSuccess.bind(this),
      this.connectFailure.bind(this)
    );
    this.adapter.setDataChannelListeners(
      this.dataChannelOpen.bind(this),
      this.dataChannelClosed.bind(this),
      this.receivedData.bind(this)
    );
    this.adapter.setRoomOccupantListener(this.occupantsReceived.bind(this));

    return this.adapter.connect();
  }
```

首先该方法将输入的app名称和房间名称赋值给了全局变量app和room，这里的两个值都是在networked-scene组件中由开发者自定义的data，之后调用了adapter中的三个方法，将server的URL信息和刚才提到的两个信息传给了adapter（注意这里的adapter已经在之前赋值过了），之后根据networked-scene组件中的另两个配置，即是否需要音频/视频，将值传递了adapter，之后的操作则是为adapter赋予了一批监听器，并调用adapter的`connect()`方法。看到这里我们不难发现，进一步的连接工作是在adapter对象中进行的，NetworkConnection类就好像一个中间层，连接了AFrame组件和adapter对象，我们进一步去adapter中查看对应方法的具体实现。



**下面先进入到adapters文件夹中，看一看adapter类是怎么实现的**



在3.3中我们提到过，全局变量中的`adapters`类具体的代码对应的是`src/adapters/AdapterFactory.js`（见NAFIndex.js的7和19行），在AdapterFactory.js中，首先索引了不同adapter具体的js文件，将adapter名称和其对应的js文件的键值对关系进行了定义，同时提供了一个`register()`方法作为注册新adapter的方式（留心，可能之后更换网络架构的时候会用到），具体的代码展示如下：

```javascript
class AdapterFactory {
  constructor() {
    this.adapters = {
      "wseasyrtc": WsEasyRtcAdapter,
      "easyrtc": EasyRtcAdapter,
      "socketio": SocketioAdapter,
      "webrtc": WebrtcAdapter,
    };

    this.IS_CONNECTED = AdapterFactory.IS_CONNECTED;
    this.CONNECTING = AdapterFactory.CONNECTING;
    this.NOT_CONNECTED = AdapterFactory.NOT_CONNECTED;
  }

  register(adapterName, AdapterClass) {
    this.adapters[adapterName] = AdapterClass;
  }

  make(adapterName) {
    var name = adapterName.toLowerCase();
    if (this.adapters[name]) {
      var AdapterClass = this.adapters[name];
      return new AdapterClass();
    } else {
      throw new Error(
        "Adapter: " +
          adapterName +
          " not registered. Please use NAF.adapters.register() to register this adapter."
      );
    }
  }
}
```

同时注意到了下面的`make()`方法，这个方法在networked-scene组件被调用过，负责返回指定的adapter的对象，并设置了错误处理方法，如果试图设置未注册过的adapter则会throw并报错。而在AdapterFactory.js文件中也提供了每个adapter所对应的js文件，它们也是实际的adapter对象，如下所示：

```javascript
const WsEasyRtcAdapter = require("./WsEasyRtcAdapter");
const EasyRtcAdapter = require("./EasyRtcAdapter");
const WebrtcAdapter = require("./naf-webrtc-adapter");
const SocketioAdapter = require('./naf-socketio-adapter');
```

所以我们现在应该选一个adapter作为例子，去它的代码中寻找NetworkConnection类所调用的那些方法，这里就以easyrtc的adapter为例进行分析。这里顺便提一嘴，adapters路径下的那个`NoOpAdapter.js`是所有adapter的父类，它定义了一个adapter应该实现的方法，同时继承了`NafInterface`类，而在这个接口类中仅仅提供了这样一个方法：

```javascript
class NafInterface {
  notImplemented(name) {
    NAF.log.error('Interface method not implemented:', name);
  }
}
```

这个方法的作用就是在`NoOpAdapter`类中负责对那些未实现的方法进行提示，如果没有实现指定的方法，就会log出该信息，包括未实现的方法名称。这一块在我们自己开发新的adapter的时候值得注意。

回到NetworkConnection类和easyrtc的adapter，一开始的三个set方法分别赋值了app name，room name和server url，它们对应在adapter中的方法如下：

- app name：直接赋值到了adapter类中的app变量上
- room name：easyrtc.joinRoom
- server url: easyrtc.setSocketUrl

之后是和`WebRtcOptions`有关的设置，在adapter中对应的是调用各种enable方法，其实就是简单的参数传递，如下所示：

```javascript
setWebRtcOptions(options) {
    // this.easyrtc.enableDebug(true);
    this.easyrtc.enableDataChannels(options.datachannel);

    this.easyrtc.enableVideo(options.video);
    this.easyrtc.enableAudio(options.audio);

    // TODO receive(audio|video) options ?
    this.easyrtc.enableVideoReceive(true);
    this.easyrtc.enableAudioReceive(true);
  }
```

再之后是关于连接结果的两个监听器，分别是success的和failure的，在`adapter.setServerConnectListeners()`中实际上也只是简单的把两个函数赋值给了adapter中的两个监听器的对象，为啥要做这么一道工序呢？因为NAF的全局变量是在NetworkConnection的方法里取赋值的，其中就包括了这里`connectSuccess()`的方法会给`NAF.clientID`赋值，而这个值又是从adapter里来的（准确的说比如这里是从easyrtc的api方法里得来的），具体的操作会在后面提到。在触发了`connectSuccess()`方法时，方法内部还会触发自定义的"connected"事件，这个事件就是我们在本节一开始提到的，在在networked-scene组件中定义的一个连接成功事件，会触发我们在schema中设置的"onConnect"属性（或者说函数）。这里一系列的触发逻辑可以理一下，这一块adapter的代码如下所示：

```javascript
\\ 关于这里两个监听器方法被触发的时机实际上在adapter类的connect方法中，之后会提到
setServerConnectListeners(successListener, failureListener) {
  this.connectSuccess = successListener;
  this.connectFailure = failureListener;
}
```

之后就是设置DataChannelListeners相关的方法，这里涉及到adapter进一步的方法我们暂时不做探究，先只讨论发生在NAF内部的事情，具体的例如easyrtc的方法可以去参考open-easyrtc的官方api。这一块在adapter里对应的部分如下所示（后面的`setRoomOccupantListener()`逻辑上差不多，就不列举了）：

```javascript
  setDataChannelListeners(openListener, closedListener, messageListener) {
    this.easyrtc.setDataChannelOpenListener(openListener);
    this.easyrtc.setDataChannelCloseListener(closedListener);
    this.easyrtc.setPeerListener(messageListener);
  }
```

在设置好一系列的监听器之后，在NetworkConnection类中就调用了adapter类的connect方法去进行正式的连接：

```javascript
connect() {
  Promise.all([
    this.updateTimeOffset(),
    new Promise((resolve, reject) => {
      this._connect(resolve, reject);
    })
  ]).then(([_, clientId]) => {
    this._myRoomJoinTime = this._getRoomJoinTime(clientId);
    this.connectSuccess(clientId);
  }).catch(this.connectFailure);
}
```

这里涉及到了promise的用法，它是JavaScript中异步编程的实现方式，可以参考：[JavaScript Promise | 菜鸟教程 (runoob.com)](https://www.runoob.com/js/js-promise.html)，这里的`promise.all()`函数可以认为是传入一串任务，必须执行完才会resolve，否则任何一个reject都会造成整个promise的reject。这里首先一共传入了两个任务，第一个是`updateTimeOffset()`，这个任务的作用是更新时间的偏移（可以暂时先不深究，具体实现就在connect方法前面，可以理解为时间校准），第二个任务是另一个异步执行的方法，即私有的`_connect()`方法，这个方法的内容如下所示：

```javascript
_connect(connectSuccess, connectFailure) {
  var that = this;

  this.easyrtc.setStreamAcceptor(this.setMediaStream.bind(this));

  this.easyrtc.setOnStreamClosed(function(clientId, stream, streamName) {
    if (streamName === "default") {
      delete that.mediaStreams[clientId].audio;
      delete that.mediaStreams[clientId].video;
    } else {
      delete that.mediaStreams[clientId][streamName];
    }

    if (Object.keys(that.mediaStreams[clientId]).length === 0) {
      delete that.mediaStreams[clientId];
    }
  });

  if (that.easyrtc.audioEnabled || that.easyrtc.videoEnabled) {
    navigator.mediaDevices.getUserMedia({
      video: that.easyrtc.videoEnabled,
      audio: that.easyrtc.audioEnabled
    }).then(
      function(stream) {
        that.addLocalMediaStream(stream, "default");
        that.easyrtc.connect(that.app, connectSuccess, connectFailure);
      },
      function(errorCode, errmesg) {
        NAF.log.error(errorCode, errmesg);
      }
    );
  } else {
    that.easyrtc.connect(that.app, connectSuccess, connectFailure);
  }
}
```

这里调用了大量的easyrtc提供的方法，抽象来理解可以认为是配置接受对方（peer connection中的）的媒体流和自己本地的媒体流，并且最终调用easyrtc库的connect方法作为实际连接操作。从这里就可以看出，音视频的传输完全是依赖adapter实现的，如果要深入探究原理，必须也去分析easyrtc（其他adapter同理）的源码，这里为了避免拓宽的太广，可以暂时不展开，不过需要有个概念，即从adapter到NAF的接口路径大概是怎样的。

在执行完私有的connect方法之后就获得了clientID和加入房间的时间，并在then语法的部分去执行`connectSuccess()`方法，如果失败了就是`connectFailure()`方法，这里也回收了之前提到的这些监听器方法是何时被触发的，通过触发这些方法，easyrtc产生的一些重要数据就回到了NAF层面并展示在了console窗口中。

### 3.5 networked组件

在介绍networked组件之前需要了解AFrame中的另一个概念，即System（参考：[System – A-Frame (aframe.io)](https://aframe.io/docs/1.2.0/core/systems.html#methods)），它可以统筹管理一类entity（附加了同一组件的）的属性和行为，也可以作为将复杂逻辑从component中分离的工具。它的声明方法和component很像，通过一个`registerSystem()`方法进行注册，在系统中同样的有`schema`作为属性，也同样可以通过data进行访问。同时system内部也可以使用以下方法对生命周期的行为进行定义：

| Name  | Description                                                  |
| ----- | ------------------------------------------------------------ |
| init  | Called once when the system is initialized. Used to initialize. |
| pause | Called when the scene pauses. Used to stop dynamic behavior. |
| play  | Called when the scene starts or resumes. Used to start dynamic behavior. |
| tick  | If defined, will be called on every tick of the scene’s render loop. |

同时System可以直接在同名的component中用`this.system`进行调用，这样一些复杂的逻辑就可以在system中进行实现而在component中进行调用，实现数据和复杂逻辑的分离。此外，对于同名的component，system可以将它们统一管理，使用system内部的`registerMe()`方法，可以将装有指定component的entity添加到system的一个数组中进行管理。观察networked组件可以发现，其中就进行了同名system的注册，并且也在其中实现了添加同名component的方法：

```javascript
AFRAME.registerSystem("networked", {
  init() {
    // An array of "networked" component instances.
    this.components = [];
    this.nextSyncTime = 0;
  },

  register(component) {
    this.components.push(component);
  },

  deregister(component) {
    const idx = this.components.indexOf(component);

    if (idx > -1) {
      this.components.splice(idx, 1);
    }
  },
  
  ...
 }
```

在注册完这些同名的component之后，system会对它们进行统一的逻辑处理，即更新它们的必要信息。这一步的操作是在生命周期中的`tick`函数中运行的，tick函数会在每帧被调用，因此这部分的代码不应该生成过多的新变量，否则会造成garbage collection方面的问题。这里的代码如下所示：

```javascript
tick: (function() {

  return function() {
    if (!NAF.connection.adapter) return;
    if (this.el.clock.elapsedTime < this.nextSyncTime) return;

    // "d" is an array of entity datas per entity in this.components.
    const data = { d: [] };

    for (let i = 0, l = this.components.length; i < l; i++) {
      const c = this.components[i];
      if (!c.isMine()) continue;
      if (!c.el.parentElement) {
        NAF.log.error("entity registered with system despite being removed");
        //TODO: Find out why tick is still being called
        return;
      }

      const syncData = this.components[i].syncDirty();
      if (!syncData) continue;

      data.d.push(syncData);
    }

    if (data.d.length > 0) {
      NAF.connection.broadcastData('um', data);
    }

    this.updateNextSyncTime();
  };
})(),

updateNextSyncTime() {
  this.nextSyncTime = this.el.clock.elapsedTime + 1 / NAF.options.updateRate;
}
```

这里可以看到，首先在tick中会进行两个判断，如果未配置adapter则不进行任何操作直接返回，而第二个判断则是表明如果未到我们设置的更新时间（避免一个帧更新一次，消耗过多的算力资源），则也直接返回，这里维护了一个`nextSyncTime`，这个变量是在每一次更新的最后使用`updateNextSyncTime()`方法进行更新和维护的，即当前已运行的时间加上我们设置的更新周期，这个周期是使用1除以更新率计算的。如果当前时间还未到下一次更新时间点，则直接返回。

之后的操作是遍历之前已经注册好的compnent数组（实际上存储的是一批entity），并且判断entity是否为当前用户所拥有（根据clientID来判断），之后判断该entity是否有父element（对于aframe的实体来说至少会有`<a-scene>`），满足条件后，则会调用component中的`syncDirty()`方法去读取某个entity中的数据，并且将数据存入事先声明的data对象中，这个方法是在之后的同名组件中声明的，可以在之后详细去看。最后如果data中含有了数据，则通过`NetworkConnection`类中的`broadcastData()`方法进行发送，可以想象，这一步是在connection那一层于adapter进行对接，如果去看具体的代码会发现果不其然，在`NetworkConnection`类中调用了adapter中的同名方法，而在adapter中则进行了如下操作：

```javascript
broadcastData(dataType, data) {
  var roomOccupants = this.easyrtc.getRoomOccupantsAsMap(this.room);

  // Iterate over the keys of the easyrtc room occupants map.
  // getRoomOccupantsAsArray uses Object.keys which allocates memory.
  for (var roomOccupant in roomOccupants) {
    if (
      roomOccupants[roomOccupant] &&
      roomOccupant !== this.easyrtc.myEasyrtcid
    ) {
      // send via webrtc otherwise fallback to websockets
      this.easyrtc.sendData(roomOccupant, dataType, data);
    }
  }
}
```

这里同样是以easyrtc的adapter为例，通过`getRoomOccupantsAsMap()`可以获得在房间内所有的peer连接，以map的形式存储，之后遍历map并通过adapter提供的`sendData()`方法（可以理解为非音视频信息的通道）为每一个连接发送。至此就看完了`networked`组件所对应的同名system的结构，它的主要目的就是统一管理所有附有`networked`组件的entity，并且在tick中进行统一更新的操作（但是并不是每个tick都更新）。之后我们就可以进入到`networked`组件本身的代码中去看它的设计，并且这里还有一个从组件中获取需要同步的信息的方法，即`syncDirty()`方法，需要在组件代码中寻找具体实现。首先我们从`networked`组件的schema开始看起：

```javascript
schema: {
  template: {default: ''},
  attachTemplateToLocal: { default: true },
  persistent: { default: false },

  networkId: {default: ''},
  owner: {default: ''},
  creator: {default: ''}
},
```

根据官方文档我们知道，`template`的作用是作为css选择器去选择需要同步的entity所对应的`<a-asset>`，而`attachTemplateToLocal`属性在文档中讲的较为模糊，之后可以根据具体的方法进行分析，最后`persistent`属性设置的是当远程用户断开时是否尝试去获取对方拥有的entity并归为自己拥有，这样就不会删除它们了。之后去看init函数中的操作，这部分的代码比较多，因为涉及到很多所需变量的初始化（例如在更新时所需要的一些坐标数据和欧拉角等）和事件的初始化，之后会根据该组件的`template`属性利用`NAF.schema.getComponents()`获取同一template下的所有entity（这个方法的源码见Schema.js），并且构建一个等长度的空数组作为缓存用的数据结构。

之后通过`initNetworkParent()`方法构建父元素的结构树，即判断该component加持下的entity是否有也含有`networked`组件的父元素，如果有的话则作为变量传入这个entity，以便之后的更新逻辑使用。再后，会通过随机数生成该entity的network id，这个id同时也是该entity的name的值。

如果当前组件所附加的entity是由网络产生的（即并非一开始就存在于场景中，而是通过`NetworkEntity`中的addNetworkComponent方法添加的），则需要调用`firstUpdate()`方法进行第一次的更新，而`attachTemplateToLocal()`方法也是在此处调用的，观察它的源码可以发现其功能是将目前处理的entity赋予其template（在创造的时候通过schema传入）的名称和值，这个template的具体信息是从`Schema.js`中的缓存取出的，这对应着我们通过`NAF.schemas.add()`操作添加template的过程（实际上就是在创造缓存）。因此如果我们将`attachTemplateToLocal`属性置为true，则该entity所对应的template会始终更新为缓存中的同名template。

在配置好一系列的准备工作后，init函数最终会触发connected事件（如果还没有clientID就视为未连接好，但是还是会先注册触发器函数），entityCreated事件和instantiated事件，这些事件的主要目的是输出一些信息在console中以供开发者清楚的知晓运行进度。最后，将处理的entity注册到system中，供system统一管理。在这里我们顺便看一下`syncDirty()`方法是如何从entity中提取需要同步的数据的：

```javascript
syncDirty: function() {
  if (!this.canSync()) {
    return;
  }

  var components = this.gatherComponentsData(false);

  if (components === null) {
    return;
  }

  return this.createSyncData(components);
},
```

首先会判定是否可以同步，这个判定条件有两个满足的情况：

- 当前用户是entity的拥有者
- 当前用户是entity的创造者，但是其拥有者已经退出了房间

之后通过`gatherComponentsData()`方法，首先根据当前处理的entity的template名称，去寻找template中定义的所需要同步的component有哪些（注意，这里的component值得往往是类似position和rotation这种属性类的组件），然后根据这些组件的名称去获取当前entity中对应的attribute，这样就可以进一步的通过attribute去获得我们想要的value了。当然在发送前，还需要经过`createSyncData()`的包装，使这一组数据不仅仅有需要更新的数据信息，还有更多的关于网络的设置等信息，具体可以参考其代码：

```javascript
createSyncData: function(components, isFirstSync) {
  var { syncData, data } = this;
  syncData.networkId = data.networkId;
  syncData.owner = data.owner;
  syncData.creator = data.creator;
  syncData.lastOwnerTime = this.lastOwnerTime;
  syncData.template = data.template;
  syncData.persistent = data.persistent;
  syncData.parent = this.getParentId();
  syncData.components = components;
  syncData.isFirstSync = !!isFirstSync;
  return syncData;
},
```

实际上在networked组件中也有tick方法，其中定义的是检查网络更新的数据是否valid，例如空值就会被认为是invalid的操作，如果产生了这种操作则会以log予以警示。此外，在本地执行更新的时候，使用了[buffered-interpolation](https://github.com/InfiniteLee/buffered-interpolation)库使得更新过程更加平滑。

尽管networked组件的代码较为复杂，但是我们可以总结一下其中的一些关键逻辑，例如本地的变化被采集后通过包装并贴上标签（例如u,um和r，详见NetworkConnection.js的第2行），通过adapter的数据通道发送到每个peer中，而在接受过程中，首先是在NetworkConnection中注册了每种标签对应的方法，例如um(upadate multi)标签就被对应到了entity的`updateMulti()`方法中，而这个方法是在NetworkEntities.js中实现的，其调用了我们现在正在谈论的networked组件中的`networkUpdate()`方法。更抽象一些来说，对接最外层网络通信的始终是adapter，而反馈在NAF场景中的始终是entity或者说其组件相关的代码。

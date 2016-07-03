title: 使用 WebRTC 构建简单的前端视频通讯
date: 2016-07-03 21:27:19
tags:
- 技术日记
---

在传统的 Web 应用中，浏览器与浏览器之间是无法直接相互通信的，必须借助服务器的帮助，但是随着 WebRTC 在各大浏览器中的普及，这一现状得到了改变。

WebRTC（Web Real-Time Communication，Web实时通信），是一个支持网页浏览器之间进行实时数据传输（包括音频、视频、数据流）的技术，谷歌于2011年5月开放了工程的源代码，目前在各大浏览器的最新版本中都得到了不同程度的支持。

这篇文章里我们采用 WebRTC 来构建一个简单的视频传输应用。

<!-- more -->

#一、关于 WebRTC 的一些基本概念
传统的视频推流的技术实现一般是这样的：客户端采集视频数据，推流到服务器上，服务器再根据具体情况将视频数据推送到其他客户端上。

但是 WebRTC 却截然不同，它可以在客户端之间直接搭建基于 UDP 的数据通道，经过简单的握手流程之后，可以在不同设备的两个浏览器内直接传输任意数据。

这其中的流程包括：

1. 采集视频流数据，创建一个 RTCPeerConnection
2. 创建一个 SDP offer 和相应的回应
3. 为双方找到 ICE 候选路径
4. 成功创建一个 WebRTC 连接

下面我们介绍这其中涉及到的一些关键词：

###1、RTCPeerConnection 对象
`RTCPeerConnection` 对象是 WebRTC API 的入口，它负责创建、维护一个 WebRTC 连接，以及在这个连接中的数据传输。目前新版本的浏览器大都支持了这一对象，但是由于目前 API 还不稳定，所以需要加入各个浏览器内核的前缀，例如 Chrome 中我们使用 `webkitRTCPeerConnection` 来访问它。

###2、会话描述协议（SDP）
为了连接到其他用户，我们必须要对其他用户的设备情况有所了解，比如音频视频的编码解码器、使用何种编码格式、使用何种网络、设备的数据处理能力，所以我们需要一张“名片”来获得用户的所有信息，而 SDP 为我们提供了这些功能。

一个 SDP 的握手由一个 offer 和一个 answer 组成。

###3、网络通信引擎（ICE）
通信的两侧可能会处于不同的网络环境中，有时会存在好几层的访问控制、防火墙、路由跳转，所以我们需要一种方法在复杂的网络环境中找到对方，并且连接到相应的目标。WebRTC 使用了集成了 STUN、TURN 的 ICE 来进行双方的数据通信。


#二、创建一个 RTCPeerConnection

首先我们的目标是在同一个页面中创建两个实时视频，一个的数据直接来自你的摄像头，另一个的数据来自本地创建的 WebRTC 连接。看起来是这样的：

图图图。。。。。。。

首先我们创建一个简单的 HTML 页面，含有两个 `video` 标签：

```html
<!DOCTYPE html>
<html>
<head>
    <title></title>
    <style type="text/css">
        #theirs{
            position: absolute;
            top: 100px;
            left: 100px;
            width: 500px;
        }
        #yours{
            position: absolute;
            top: 120px;
            left: 480px;
            width: 100px;
            z-index: 9999;
            border:1px solid #ddd;
        }
    </style>
</head>
<body>
<video id="yours" autoplay></video>
<video id="theirs" autoplay></video>
</body>
<script type="text/javascript" src="./main.js"></script>
</html>
```

下面我们创建一个 `main.js` 文件，先封装一下各浏览器的 `userMedia` 和 `RTCPeerConnection` 对象：

```js
function hasUserMedia() {
    navigator.getUserMedia = navigator.getUserMedia || navigator.msGetUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia;
    return !!navigator.getUserMedia;
}

function hasRTCPeerConnection() {
    window.RTCPeerConnection = window.RTCPeerConnection || window.webkitRTCPeerConnection || window.mozRTCPeerConnection || window.msRTCPeerConnection;
    return !!window.RTCPeerConnection;
}
```

然后我们需要浏览器调用系统的摄像头 API `getUserMedia` 获得媒体流，注意要打开浏览器的摄像头限制。Chrome由于安全的问题，只能在 https 下或者 localhost 下打开摄像头。

```js
var yourVideo = document.getElementById("yours");
var theirVideo = document.getElementById("theirs");
var yourConnection, theirConnection;

if (hasUserMedia()) {
    navigator.getUserMedia({ video: true, audio: false },
        stream => {
            yourVideo.src = window.URL.createObjectURL(stream);
            if (hasRTCPeerConnection()) {
                // 稍后我们实现 startPeerConnection
                startPeerConnection(stream);
            } else {
                alert("没有RTCPeerConnection API");
            }
        },
        err => {
            console.log(err);
        }
    )
}else{
	alert("没有userMedia API")
}
```

没有意外的话，现在应该能在页面中看到一个视频了。

下一步是实现 `startPeerConnection` 方法，建立传输视频数据所需要的 ICE 通信路径，这里我们以 Chrome 为例：

```js
function startPeerConnection(stream) {
	//这里使用了几个公共的stun协议服务器
    var config = {
        'iceServers': [{ 'url': 'stun:stun.services.mozilla.com' }, { 'url': 'stun:stunserver.org' }, { 'url': 'stun:stun.l.google.com:19302' }]
    };
    yourConnection = new RTCPeerConnection(config);
    theirConnection = new RTCPeerConnection(config);
    yourConnection.onicecandidate = function(e) {
        if (e.candidate) {
            theirConnection.addIceCandidate(new RTCIceCandidate(e.candidate));
        }
    }
    theirConnection.onicecandidate = function(e) {
        if (e.candidate) {
            yourConnection.addIceCandidate(new RTCIceCandidate(e.candidate));
        }
    }
}
```
我们使用这个函数创建了两个连接对象，在 `config` 里，你可以任意指定 ICE 服务器，虽然有些浏览器内置了默认的 ICE 服务器，可以不用配置，但还是建议加上这些配置。下面，我们进行 SDP 的握手。

由于是在同一页面中进行的通信，所以我们可以直接交换双方的 `candidate` 对象，但在不同页面中，可能需要一个额外的服务器协助这个交换流程。


#三、建立 SDP Offer 和 SDP Answer
浏览器为我们封装好了相应的 Offer 和 Answer 方法，我们可以直接使用。

```js
function startPeerConnection(stream) {
    var config = {
        'iceServers': [{ 'url': 'stun:stun.services.mozilla.com' }, { 'url': 'stun:stunserver.org' }, { 'url': 'stun:stun.l.google.com:19302' }]
    };
    yourConnection = new RTCPeerConnection(config);
    theirConnection = new RTCPeerConnection(config);
    yourConnection.onicecandidate = function(e) {
        if (e.candidate) {
            theirConnection.addIceCandidate(new RTCIceCandidate(e.candidate));
        }
    }
    theirConnection.onicecandidate = function(e) {
        if (e.candidate) {
            yourConnection.addIceCandidate(new RTCIceCandidate(e.candidate));
        }
    }

	//本方产生了一个offer
    yourConnection.createOffer().then(offer => {
        yourConnection.setLocalDescription(offer);
        //对方接收到这个offer
        theirConnection.setRemoteDescription(offer);
        //对方产生一个answer
        theirConnection.createAnswer().then(answer => {
            theirConnection.setLocalDescription(answer);
            //本方接收到一个answer
            yourConnection.setRemoteDescription(answer);
        })
    });

}
```
和 ICE 的连接一样，由于我们是在同一个页面中进行 SDP 的握手，所以不需要借助任何其他的通信手段来交换 offer 和 answer，直接赋值即可。如果需要在两个不同的页面中进行交换，则需要借助一个额外的服务器来协助，可以采用 websocket 或者其它手段进行这个交换过程。


#三、加入视频流
现在我们已经有了一个可靠的视频数据传输通道了，下一步只需要向这个通道加入数据流即可。WebRTC 直接为我们封装好了加入视频流的接口，当视频流添加时，另一方的浏览器会通过 `onaddstream` 来告知用户，通道中有视频流加入。

```js
yourConnection.addStream(stream);
theirConnection.onaddstream = function(e) {
    theirVideo.src = window.URL.createObjectURL(e.stream);
}
```

以下是完整的 `main.js` 代码：

```js
function hasUserMedia() {
    navigator.getUserMedia = navigator.getUserMedia || navigator.msGetUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia;
    return !!navigator.getUserMedia;
}
function hasRTCPeerConnection() {
    window.RTCPeerConnection = window.RTCPeerConnection || window.webkitRTCPeerConnection || window.mozRTCPeerConnection || window.msRTCPeerConnection;
    return !!window.RTCPeerConnection;
}

var yourVideo = document.getElementById("yours");
var theirVideo = document.getElementById("theirs");
var yourConnection, theirConnection;

if (hasUserMedia()) {
    navigator.getUserMedia({ video: true, audio: false },
        stream => {
            yourVideo.src = window.URL.createObjectURL(stream);
            if (hasRTCPeerConnection()) {
                startPeerConnection(stream);
            } else {
                alert("没有RTCPeerConnection API");
            }
        },
        err => {
            console.log(err);
        })
} else {
    alert("没有userMedia API")
}

function startPeerConnection(stream) {
    var config = {
        'iceServers': [{ 'url': 'stun:stun.services.mozilla.com' }, { 'url': 'stun:stunserver.org' }, { 'url': 'stun:stun.l.google.com:19302' }]
    };
    yourConnection = new RTCPeerConnection(config);
    theirConnection = new RTCPeerConnection(config);

    yourConnection.onicecandidate = function(e) {
        if (e.candidate) {
            theirConnection.addIceCandidate(new RTCIceCandidate(e.candidate));
        }
    }
    theirConnection.onicecandidate = function(e) {
        if (e.candidate) {
            yourConnection.addIceCandidate(new RTCIceCandidate(e.candidate));
        }
    }
    
    theirConnection.onaddstream = function(e) {
        theirVideo.src = window.URL.createObjectURL(e.stream);
    }
    yourConnection.addStream(stream);
    
    yourConnection.createOffer().then(offer => {
        yourConnection.setLocalDescription(offer);
        theirConnection.setRemoteDescription(offer);
        theirConnection.createAnswer().then(answer => {
            theirConnection.setLocalDescription(answer);
            yourConnection.setRemoteDescription(answer);
        })
    });
}

```
	
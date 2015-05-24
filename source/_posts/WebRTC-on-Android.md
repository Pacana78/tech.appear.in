---
title: "WebRTC on Android"
date: 2015-05-14 22:38:13
comments: true
toc: true
authorId: dagingaa
tags: 
	- webrtc
	- android
---

WebRTC has been heralded as a new front in a long war for an open and unencumbered web, and is seen as one of the most important innovations in web standards in recent years. WebRTC has allowed web developers to add video chat or peer to peer data transfer inside their web apps, all without complex code or expensive infrastructure. Supported today in Chrome, Firefox and Opera, with more browsers on the way, it has the capability to reach billions of devices already. 

However, WebRTC suffers from an urban myth: WebRTC is just for the browser. In fact, one of the most important things about WebRTC is the fact that it allows for full interoperability between native and web apps alike. Few take advantage of that fact.

In this blog post we will investigate how you can get started with building WebRTC into your Android apps, using the native libraries provided by the [WebRTC Initiative](http://www.webrtc.org/). We will not be going through how you set up a call using a signalling mechanism, but instead highlight what similarities and differences Android has over the implementation in browsers. As you will see, the APIs for Android parallel those for the web. If you are looking for a more basic introduction to WebRTC, I can highly recommend [Sam Dutton’s Getting started with WebRTC](http://www.html5rocks.com/en/tutorials/webrtc/basics/).

#Adding the WebRTC library to your project
_The following guide has been written with the Android WebRTC library version 9127 in mind._

The very first thing you need to do is add the WebRTC library to your application. The WebRTC Initiative has [a pretty barebones guide to compiling it yourself](http://www.webrtc.org/native-code/android), but trust me, you want to avoid that if you can. Instead, at appear.in we use pristine.io’s compiled version, [available from the maven central repository](https://oss.sonatype.org/content/groups/public/io/pristine/).

To add the WebRTC library to your project, you need to add the following line to your dependencies:

```
compile 'io.pristine:libjingle:9127@aar'
```

Sync your project, and you will now have the WebRTC libraries ready to use!

##Permissions
As with all Android applications, you need to have certain permissions to use certain APIs. Building stuff with WebRTC is no exception. Depending on what application you are making, or what features you need, such as audio and video, you will need different sets of permissions. Make sure you only request the ones you need! A good permission-set for a video chat application could be:

```
<uses-feature android:name="android.hardware.camera" />
<uses-feature android:name="android.hardware.camera.autofocus" />
<uses-feature android:glEsVersion="0x00020000" android:required="true" />

<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
```

#Lights, camera… factory...
When using WebRTC in the browser, you have some pretty well functioning and well documented APIs to work with. [navigator.getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/getUserMedia) and [RTCPeerConnection](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection) contain mostly everything you need to get up and running. Combine that with the `<video>` tag, and you have everything you need to show both your local webcam stream, as well as all the remote videos streams you’d like.

Luckily, the APIs are not that different on Android, though they have slightly different names. On Android, we talk about [VideoCapturerAndroid](#VideoCapturerAndroid), [VideoRenderer](#VideoRenderer), [MediaStream](#MediaStream), [PeerConnection](#PeerConnection), and [PeerConnectionFactory](#PeerConnectionFactory). Let’s take a deep dive into each one.

However, before you can begin doing anything, you need to create your PeerConnectionFactory, the very core of all things WebRTC on Android.

##PeerConnectionFactory
The core of all things WebRTC on Android. Understanding this class and how it works to help you create everything else is key to grokking Android-style WebRTC. It also behaves a bit differently from what you’d expect, so let’s dive in.

First of all, you need to initialize the PeerConnectionFactory.

```java 
// First, we initiate the PeerConnectionFactory with
// our application context and some options.
PeerConnectionFactory.initializeAndroidGlobals(
    context,
    initializeAudio,
    initializeVideo,
    videoCodecHwAcceleration,
    renderEGLContext);
```

To understand what’s going on here, let’s look at each of the parameters.

{% raw %}
<dl>
  <dt>context</dt>
  <dd>Simply the ApplicationContext, or any other Context relevant, as you are used to passing around.</dd>
  <dt>initializeAudio</dt>
  <dd>A boolean for initializing the audio portions.</dd> 
   <dt>initializeVideo</dt>
  <dd>A boolean for initializing the  video portions . Skipping either one of these two allows you to skip asking for permissions for this API as well, for example for DataChannel applications.</dd>
  <dt>videoCodecHwAcceleration</dt>
  <dd>A boolean for enabling hardware acceleration.</dd>
  <dt>renderEGLContext</dt>
  <dd>Can be provided to support HW video decoding to texture and will be used to create a shared EGL context on video decoding thread. This can be null - in this case HW video decoder will generate yuv420 frames instead of texture frames.</dd>
</dl>
{% endraw %}

`initializeAndroidGlobals` will also return a boolean, which is true if everything went OK, and false if something failed. It is best practice to fail gracefully if a false value is returned. See also [the source](https://code.google.com/p/webrtc/source/browse/trunk/talk/app/webrtc/java/src/org/webrtc/PeerConnectionFactory.java?r=8344&spec=svn8423#64) for more information.

Assuming everything went ok, you can now create your factory using the PeerConnectionFactory constructor, like any other class.

```java
PeerConnectionFactory peerConnectionFactory = new PeerConnectionFactory();
```
#Aaaand action, fetching media and rendering
Once you have your `peerConnectionFactory` instance, it's time to get vidoe and audio from the user's device, and finally rendering that to the screen. On the web, you have `getUserMedia` and `<video>`. Things aren't so simple on Android, but you get a lot more options! On Android, we talk about VideoCapturerAndroid, VideoSource, VideoTrack and VideoRenderer. It all starts with VideoCapturerAndroid.

##VideoCapturerAndroid
The VideoCapturerAndroid class is really just a neat wrapper around the Camera API, providing convenience functions to access the camera streams of the device. It allows you to fetch the number of camera devices, get the front facing camera, or the back facing camera.

```java
// Returns the number of camera devices
VideoCapturerAndroid.getDeviceCount();

// Returns the front face device name
VideoCapturerAndroid.getNameOfFrontFacingDevice();
// Returns the back facing device name
VideoCapturerAndroid.getNameOfBackFacingDevice();

// Creates a VideoCapturerAndroid instance for the device name
VideoCapturerAndroid.create(name);
```

With an instance of the VideoCapturerAndroid class containing a camera stream you have the possibility to create a MediaStream containing the video stream from your camera, which you can send to the other peer. But before we do that, let’s look at how we can display your own video in your application first.

##VideoSource/VideoTrack
To get anything useful from your VideoCapturer instance, or rather, to reach the end goal of getting a proper MediaStream for your PeerConnection, or even just to render it to the user, you need to go through the VideoSource and VideoTrack classes.

[VideoSource](https://chromium.googlesource.com/external/webrtc/+/master/talk/app/webrtc/java/src/org/webrtc/VideoSource.java) enables functions to start/stop capturing your device. This is advantageous in situations where it is better to disable the capture device to increase battery life.

[VideoTrack](https://chromium.googlesource.com/external/webrtc/+/master/talk/app/webrtc/java/src/org/webrtc/VideoTrack.java) is a wrapper to simplify adding the VideoSource to your MediaStream object.

Let’s look at some code to see how they work together. `capturer` is our VideoCapturer instance, and `videoConstraints` is an instance of MediaConstraints.

```java
// First we create a VideoSource
VideoSource videoSource = 
	peerConnectionFactory.createVideoSource(capturer, videoConstraints);

// Once we have that, we can create our VideoTrack
// Note that VIDEO_TRACK_ID can be any string that uniquely
// identifies that video track in your application
VideoTrack localVideoTrack = 
	peerConnectionFactory.createVideoTrack(VIDEO_TRACK_ID, videoSource);
```

##AudioSource/AudioTrack
[AudioSource](https://chromium.googlesource.com/external/webrtc/+/master/talk/app/webrtc/java/src/org/webrtc/AudioSource.java) and [AudioTrack](https://chromium.googlesource.com/external/webrtc/+/master/talk/app/webrtc/java/src/org/webrtc/AudioSource.java) are very similar to VideoSource and VideoTrack, except you don’t need an AudioCapturer to capture the microphone. `audioConstraints` is an instance of MediaConstraints.

```java
// First we create an AudioSource
AudioSource audioSource =
	peerConnectionFactory.createAudioSource(audioConstraints);

// Once we have that, we can create our AudioTrack
// Note that AUDIO_TRACK_ID can be any string that uniquely
// identifies that audio track in your application
AudioTrack localAudioTrack =
	peerConnectionFactory.createAudioTrack(AUDIO_TRACK_ID, audioSource);
```

##VideoRenderer
From working with WebRTC in browsers, you are probably familiar with using the `<video>` tag to display your MediaStream from getUserMedia. However, on native Android, there is no such thing as the `<video>` tag. Enter the VideoRenderer. The WebRTC library allows you to implement your own rendering implementation using `VideoRenderer.Callbacks` should you wish to do that. However, it also provides a nice default, [VideoRendererGui](https://chromium.googlesource.com/external/webrtc/+/master/talk/app/webrtc/java/android/org/webrtc/VideoRendererGui.java), which is included. In short, a VideoRendererGui is a [GLSurfaceView](https://developer.android.com/reference/android/opengl/GLSurfaceView.html) upon which you can draw your video stream.  Let’s look at how we set this up, including adding our renderer to our VideoTrack.

```java
// To create our VideoRenderer, we can use the 
// included VideoRendererGui for simplicity
// First we need to set the GLSurfaceView that it should render to
GLSurfaceView videoView = (GLSurfaceView) findViewById(R.id.glview_call);

// Then we set that view, and pass a Runnable
// to run once the surface is ready
VideoRendererGui.setView(videoView, runnable);

// Now that VideoRendererGui is ready, we can get our VideoRenderer
VideoRenderer renderer = VideoRendererGui.createGui(x, y, width, height);

// And finally, with our VideoRenderer ready, we
// can add our renderer to the VideoTrack.
localVideoTrack.addRenderer(renderer);
```

One thing to note here is that `createGui` needs four parameters. This is done to make it possible to use a single GLSurfaceView for rendering all your videos. At appear.in however, we use multiple GLSurfaceViews, which means that x,y will always be 0 to render properly. This is however up to what makes sense in your implementation.

##MediaConstraints
The MediaConstraint class is the WebRTC library’s way of supporting different constraints you can have on your audio and video tracks inside the MediaStream. See the [specification](https://w3c.github.io/mediacapture-main/#idl-def-MediaTrackSupportedConstraints) for which are supported. For most methods requiring MediaConstraints a simple instance of MediaConstraints will do.

```java
MediaConstraints audioConstraints = new MediaConstraints();
```

To add actual constraints, you can define these as `KeyValuePairs` and push to the constraint’s `mandatory` or `optional` list.

##MediaStream
Now that you can see yourself, it’s time to make sure the other party does too. From web, you are perhaps familiar with the concept of a [MediaStream](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_API). `getUserMedia` returns a MediaStream directly which you can add to your RTCPeerConnection to send to the other peer. This is also true for Android, except we have to build our own MediaStream. Let’s see how we can add our VideoTrack and AudioTrack to create a proper MediaStream.

```java
// We start out with an empty MediaStream object, 
// created with help from our PeerConnectionFactory
// Note that LOCAL_MEDIA_STREAM_ID can be any string
MediaStream mediaStream = peerConnectionFactory.createLocalMediaStream(LOCAL_MEDIA_STREAM_ID);

// Now we can add our tracks.
mediaStream.addTrack(localVideoTrack);
mediaStream.addTrack(localAudioTrack);
```

#Hello? Is there anybody out there?
We finally have our audio and video stream inside a MediaStream instance, in addition to showing the stream of your pretty face on screen. It's time to send all that to the other peer. While this guide will not include a way to set up your signalling channel, we will go through each API method and explain how it relates to the web. [AppRTC](https://chromium.googlesource.com/external/webrtc/+/master/talk/examples/android/) uses [autobahn](http://autobahn.ws/android/) to enable a WebSocket connection to a signalling server. I recommend checking out that project for details on how to set up your signalling channel inside Android.

##PeerConnection
Now that we finally have our MediaStream, we can start connecting to the other peer. Luckily, this part is much closer to how it is on the web, so if you’re familiar with WebRTC in browsers, this part should be fairly straight forward. Creating a PeerConnection is easy, with the help of none other than PeerConnectionFactory.

```java
PeerConnection peerConnection = peerConnectionFactory.createPeerConnection(
	iceServers,
	constraints, 
	observer);
```
The parameters are as follows.

{% raw %}
<dl>
  <dt>iceServers</dt>
  <dd>This is needed should you want to connect outside your local device or network. Adding STUN and TURN servers here will enable you to connect, even in hard network scenarios.</dd>
  <dt>constraints</dt>
  <dd>An instance of MediaConstraints. Should contain `offerToRecieveAudio` and `offerToRecieveVideo`.</dd>
   <dt>observer</dt>
  <dd>An instance of your implementation of PeerConnectionObserver.</dd>
</dl>
{% endraw %}

The PeerConnection API is pretty similar to that on the web, containing functions such as addStream, addIceCandidate, createOffer, createAnswer, getLocalDescription, setRemoteDescription and more. Check out [Getting started with WebRTC](http://www.html5rocks.com/en/tutorials/webrtc/basics/#toc-rtcpeerconnection) to see how all these work together to form a communication channel between two peers, or look at [AppRTC](https://chromium.googlesource.com/external/webrtc/+/master/talk) to see a live, fully functioning Android WebRTC application work. Let’s take a quick look into each of the important functions, and how they work.

###addStream
This is used to add your MediaStream to the PeerConnection. Does what it says on the label, really. If you want the other party to see your video and hear your audio, this is the function you’re looking for.

###addIceCandidate
[IceCandidates](http://stackoverflow.com/a/21071464) are created once the internal IceFramework has found candidates that allow the other peer to connect to you. While you send yours gathered through the PeerConnectionObserver.onIceCandidate handler to the other peer, you will, through whatever signalling channel you choose, receive the remote peer’s IceCandidates as they become available. Use addIceCandidate to add them to the PeerConnection, so the PeerConnection can attempt to connect to the other peer using the information within.

###createOffer/createAnswer
These two are used in the initial call setup. As you may know, in WebRTC, you have the notion of a caller and a callee, one who calls, and one who answers. createOffer is done by the caller, and it takes an sdpObserver, that allows you to fetch and transmit the Session Description Protocol (SDP) to the other party, and a MediaConstraint. Once the other party gets the offer, it will create an answer and transmit back to the caller. An SDP is metadata that describes to the other peer the format to expect (video, formats, codecs, encryption, resolution, size, etc). Once the answer is received, the peers can agree on a mutual set of requirements for the connection, video and audio codec, etc.

###setLocalDescription/setRemoteDescription
This is used to set the SDP generated from createOffer and createAnswer, including the one you get from the remote peer. This allows the internal PeerConnection engine to configure the connection so that it actually works once you start transmitting video and audio.

##PeerConnectionObserver
This interface provides a way of observing the PeerConnection for events, such as when the MediaStream is received, when iceCandidates are found, or when renegotiation is needed. These match their web counterparts in functionality, and shouldn’t be too hard understand if you come from the web, or follow [Getting started with WebRTC](http://www.html5rocks.com/en/tutorials/webrtc/basics/#toc-rtcpeerconnection). This interface must be implemented by you, so that you can handle the incoming events properly, such as signal iceCandidates to the other peer as they become available.

#Finishing up
As you can see, the APIs for Android are pretty simple and straightforward once you know how they relate to their web counterparts. With the tools above, you can develop a production ready WebRTC app, instantly deployable to billions of capable devices.

WebRTC opens up communications to all of us, free for developers, free for end users. And it enables a lot more than just video chat. We've seen applications such as health services, low-latency file transfer, torrents, and even gaming. 

To see a real-life example of a WebRTC app, check out appear.in on [Android]( https://play.google.com/store/apps/details?id=appear.in.app&referrer=utm_source%3Dtech.appear.in%26utm_medium%3Dblog%26utm_campaign%3Dandroid-launch-may15) or [iOS](https://itunes.apple.com/app/apple-store/id878583078?pt=1259761&ct=tech.appear.in&mt=8). It works perfectly between browsers and native apps, and is free for up to 8 people in the same room. No installation or login required.

Now go out there, and build something new and different!

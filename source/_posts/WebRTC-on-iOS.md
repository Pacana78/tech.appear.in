---
title: "WebRTC on iOS"
date: 2015-05-25 22:38:13
comments: true
toc: true
authorId: bruun
tags:
	- webrtc
	- iOS
---
Hangouts does it. Facebook Messenger does it. We at appear.in do it. We all use WebRTC for client to client video conversations in our iOS apps, and so can you! In addition to being an [open web standard](http://w3c.github.io/webrtc-pc/), WebRTC is an open-source project with high-level API implementations for both iOS and Android.

In this blog post we will investigate how to get started building WebRTC into your iOS app. We will use the native libraries provided by the [WebRTC Initiative](http://www.webrtc.org/), but you can check out the [OpenWebRTC project](http://www.openwebrtc.io/) as well. We will not go through how you set up a call using a signaling mechanism, but instead highlight what similarities and differences iOS has over the implementation in browsers. As you will see, the APIs for iOS parallel those for the web. If you are looking for a more basic introduction to WebRTC, I can highly recommend [Sam Dutton’s Getting started with WebRTC](http://www.html5rocks.com/en/tutorials/webrtc/basics/). The code in this article does not represent a complete working application, so I also recommend taking a look at the WebRTC team's reference implementation, [AppRTCDemo](https://chromium.googlesource.com/external/webrtc/+/master/talk/examples/objc/).

<!-- more -->

#Getting Started
The first thing you’ll need is the WebRTC library itself. The library we’re using for this tutorial is maintained by Google and is completely open source. You can download the code and build the library yourself, or you can download a prebuilt binary. The engineers at pristine.io have taken it upon themselves to build iOS binaries for every revison of WebRTC, and publish them using CocoaPods. **Unless you need to make changes to the underlying WebRTC code, we recommend that you use the pod `libjingle_peerconnection`.**

##Understanding the API
Once you have built or downloaded the library I recommend that you take some time to familiarize yourself with the header files. They are not thoroughly documented, but all methods have some clarifying comment giving you an overview of the different interfaces the library provides. All code examples below are based on revision 9103 of WebRTC, so there is a chance things don't add up if you are reading this post while working with the latest revision. With that in mind, let's write some code!

#Prepare for RTCPeerConnection
##Creating the list of ICE servers
For quick proof-of-concept assembling of an WebRTC app you do not need to provide a STUN or TURN server to the peer connection. If you omit it, you will only generate internal network host candidates, which is fine as long as you are testing on the same network. If you do want to specify STUN and TURN, this is how you do it:

```plaintext
RTCIceServer *iceServer = [[RTCICEServer alloc] initWithURI:[NSURL URLWithString:YOUR_SERVER]
    username:USERNAME_OR_EMPTY_STRING
    password:PASSWORD_OR_EMPTY_STRING]];
```

Add these to an `NSArray` and keep them until it’s time to create the peer connection.

##Callbacks and delegates
The iOS library uses the delegate pattern where the JavaScript API uses callbacks. The library provides several delegates you can implement in order to have a fully functional WebRTC application, but in order to get a demo up and running there are just two: `RTCPeerConnectionDelegate` and `RTCSessionDescriptionDelegate`.

The `RTCPeerConnectionDelegate` delegate is the Objective-C implementation of the web’s `RTCPeerConnection.onNN` callback pattern. Add the `RTCPeerConnectionDelegate` to your class’ protocol list, and add the required delegate methods. Many of them we don’t strictly need to use, but the compiler will complain if you don’t implement them. Those familiar with the web API will recognize most of these methods, but I will go through those we need to use.

The `RTCSessionDescriptionDelegate` includes the following two methods

```plaintext
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didCreateSessionDescription:(RTCSessionDescription *)sdp
                          error:(NSError *)error;
```

```plaintext
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didSetSessionDescriptionWithError:(NSError *)error;
```

and can best be explained by using their JavaScript equivalents:

```javascript
peerConnection.createOffer(function didCreateSessionDescription(sdp) {
    peerConnection.setLocalDescription(sdp, function didSetSessionDescription() {

    });
});
```

This differs slightly from the familiar JavaScript API, and I will walk you through the additional logic needed below.

##Creating the RTCPeerConnection
As you know from the web API, the `RTCPeerConnection` is what controls a WebRTC media session. The iOS library tries to mirror the web API as far as possible, meaning you will also have an `RTCPeerConnection` object for managing your call on iOS.

The `RTCPeerConnection` is created using the `RTCPeerConnectionFactory`:

```objc
// Enable SSL globally for WebRTC in our app
[RTCPeerConnectionFactory initializeSSL];
RTCPeerConnectionFactory *pcFactory = [[RTCPeerConnectionFactory alloc] init];

// Create the peer connection using the ICE server list and the current class as the delegate
RTCPeerConnection *peerConnection = [pcFactory peerConnectionWithICEServers:iceServers
                                                constraints:nil delegate:self];
```

As you can see, creating a `RTCPeerConnection` is pretty similar to the web, if you disregard the syntax differences! You should also disable SSL when you are done with your WebRTC call, or when the app terminates:

```objc
[RTCPeerConnectionFactory deinitializeSSL];
```

#Gaining camera and microphone permission

Unless we want a one-way audio/video session with a different client (like a browser or [Android application](http://tech.appear.in/2015/05/14/WebRTC-on-Android/), we need to have some media to send to the other peer. In this example we will send both the video and the audio, which we need to wrap in a `RTCMediaStream`. Unlike the web, where we have `getUserMedia`, we need to create the stream object ourself. While slightly different, this isn’t much harder than what you’re used to.

Note that the code below assumes that the user grants access to the camera and microphone, which might not be the case in the real world.

##Media stream
An `RTCMediaStream` consists of audio and video tracks, and we can choose to add several of each type, or none at all. Let’s create the stream, and then add one audio track (microphone) and one video track (camera).

```objc
RTCMediaStream *localStream = [factory mediaStreamWithLabel:@”someUniqueStreamLabel”];
```

##Audio
Getting the audio track is easy, since there is only one source, the microphone, which will be automatically selected by the WebRTC library. When you do this, iOS will trigger the microphone permission prompt automatically.

```objc
RTCAudioTrack *audioTrack = [factory audioTrackWithID:@”audio0”];
[localStream addAudioTrack:audioTrack];
```

##Video
For the video track we need to specify what `AVCaptureDevice` we want to use. You can choose between the front and back facing camera on most modern iOS devices. Let’s use the front-facing camera for that wonderful selfie experience.

Keep in mind that you don’t have access to the camera in the simulator, so the code below won’t find a `AVCaptureDevice` unless you run it on a device.

```objc
// Find the device that is the front facing camera
AVCaptureDevice *device;
for (AVCaptureDevice *captureDevice in [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo] ) {
    if (captureDevice.position == AVCaptureDevicePositionFront) {
        device = captureDevice;
        break;
    }
}

// Create a video track and add it to the media stream
if (device) {
    RTCVideoSource *videoSource;
    RTCVideoCapturer *capturer = [RTCVideoCapturer capturerWithDeviceName:device.localizedName];
    videoSource = [factory videoSourceWithCapturer:capturer constraints:nil];
    RTCVideoTrack *videoTrack = [factory videoTrackWithID:videoId source:videoSource];
    [localStream addVideoTrack:videoTrack];
}
```

That’s it, we now have a `RTCMediaStream` with a video and audio track. Now we need to add that to the peer connection.

```objc
[peerConnection addStream:localStream];
```

You are now ready to either send or receive an offer. That logic will be up to your application’s signaling mechanism, so let’s just now assume that you have been told to generate and send an offer.

#Doing the offer / answer dance
##Create offer
Let’s look at how the offer is created, which initiates the call.

```objc
RTCMediaConstraints *constraints = [RTCMediaConstraints alloc] initWithMandatoryConstraints:
    @[
        [[RTCPair alloc] initWithKey:@"OfferToReceiveAudio" value:@"true"],
        [[RTCPair alloc] initWithKey:@"OfferToReceiveVideo" value:@"true"]
    ]
    optionalConstraints: nil];

[peerConnection createOfferWithConstraints:constraints];
```

##Handle that the offer was set
Once the offer has been generated, we need to set our own local description with the session description passed into `didCreateSessionDescription`.

```objc
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didCreateSessionDescription:(RTCSessionDescription *)sdp error:(NSError *)error
{
	[peerConnection setLocalDescription:sdp]
}
```

##Handle that the local description was set
Once the local description has been set, we can transmit our session description offer generated from our `createOfferWithConstriants` call through our signaling channel to the other peer.
```objc
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didSetSessionDescriptionWithError:(NSError *)error
{
    if (peerConnection.signalingState == RTCSignalingHaveLocalOffer) {
        // Send offer through the signaling channel of our application
    }
}
```

##Handle incoming offer
At some point you should receive an answer back from the signaling server. Add that to the peer connection:

```objc
RTCSessionDescription *remoteDesc = [[RTCSessionDescription alloc] initWithType:@"answer" sdp:sdp];
[peerConnection setRemoteDescription:remoteDesc];
```

This will trigger the `didSetSessionDescriptionWithError` delegate method to fire, but currently that only handles setting the local description. Let’s extend it to handle setting the remote description:

```objc
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didSetSessionDescriptionWithError:(NSError *)error
{
  // If we have a local offer OR answer we should signal it
  if (peerConnection.signalingState == RTCSignalingHaveLocalOffer | RTCSignalingHaveLocalAnswer ) {
    // Send offer/answer through the signaling channel of our application
  } else if (peerConnection.signalingState == RTCSignalingHaveRemoteOffer) {
    // If we have a remote offer we should add it to the peer connection
    [peerConnection createAnswerWithConstraints:constraints];
  }
}
```

Great, now we are almost able to both initiate and receive a call!
We are missing one very important part: The ICE candidates.

#RTCIceCandidate
As soon as you call `setLocalDescription`, the ICE engine will start firing off the `RTCPeerConnectionDelegate` method `gotICECandidate`. These are your local `iceCandidates`, and must be sent to the other peer.

```objc
- (void)peerConnection:(RTCPeerConnection *)peerConnection
     gotICECandidate:(RTCICECandidate *)candidate
{
	// Send candidate through the signaling channel
}
```

When receiving an ICE candidate on the wire, you simple pass it into the peer connection:

```objc
RTCICECandidate *candidate = [[RTCICECandidate alloc] initWithMid:SDP_MID
                                                      index:SDP_M_LINE_INDEX
                                                        sdp:SDP_CANDIDATE];
[self.rtcPeerConnection addICECandidate:candidate];
```

#Receiving a video stream

If everything was called in the right order and your network is playing along nicely, you should now receive a call to the delegate method `addedStream`. This method is called when a remote stream is added.

```objc
- (void)peerConnection:(RTCPeerConnection *)peerConnection addedStream:(RTCMediaStream *)stream
{
    // Create a new render view with a size of your choice
    RTCEAGLVideoView *renderView = [[RTCEAGLVideoView alloc] initWithFrame:CGRectMake(100, 100)];
    [stream.videoTracks.lastObject addRenderer:self.renderView];

    // RTCEAGLVideoView is a subclass of UIView, so renderView
    // can be inserted into your view hierarchy where it suits your application.
}
```

**That’s it!** You should now have a fully working peer-to-peer video chat working.

#Finishing up
As you can see, the APIs for iOS are pretty simple and straightforward once you know how they relate to their web counterparts. With the tools above, you can develop a production ready WebRTC app, instantly deployable to billions of capable devices.

WebRTC opens up communications to all of us, free for developers, free for end users. And it enables a lot more than just video chat. We've seen applications such as health services, low-latency file transfer, torrents, and even gaming.

To see a real-life example of a WebRTC app, check out appear.in on [Android](https://play.google.com/store/apps/details?id=appear.in.app&referrer=utm_source%3Dtech.appear.in%26utm_medium%3Dblog%26utm_campaign%3Dandroid-launch-may15) or [iOS](https://itunes.apple.com/app/apple-store/id878583078?pt=1259761&ct=tech.appear.in&mt=8). It works perfectly between browsers and native apps, and is free for up to 8 people in the same room. No installation or login required.

Now go out there, and build something new and different!
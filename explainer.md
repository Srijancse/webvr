# WebVR Explained

## What is WebVR?
[WebVR](https://w3c.github.io/webvr/) is an API that provides access to input and output capabilities commonly associated with Virtual Reality hardware like [Google’s Daydream](https://vr.google.com/daydream/), the [Oculus Rift](https://www3.oculus.com/rift/), the [Samsung Gear VR](http://www.samsung.com/global/galaxy/gear-vr/), and the [HTC Vive](https://www.htcvive.com/). More simply put, it lets you create Virtual Reality web sites that you can view in a VR headset.

### Ooh, so like _Johnny Mnemonic_ where the Internet is all ’90s CGI?
Nope, not even slightly. And why do you even want that? That’s a terrible UX.

WebVR, at least initially, is aimed at letting you create VR experiences that are embedded in the web that we know and love today. It’s explicitly not about creating a browser that you use completely in VR (although it could work well in an environment like that).

### Goals
Enable Virtual Reality applications on the web by allowing pages to do the following:

* Detect available Virtual Reality devices.
* Query the devices capabilities.
* Poll the device’s position and orientation.
* Display imagery on the device at the appropriate frame rate.

### Non-goals

* Define how a Virtual Reality browser would work.
* Take full advantage of Augmented Reality devices.
* Build “[The Metaverse](https://en.wikipedia.org/wiki/Metaverse).”

## Use cases
Given the marketing of early VR hardware to gamers, one may naturally assume that this API will primarily be used for development of games. While that’s certainly something we expect to see given the history of the WebGL API, which is tightly related, we’ll probably see far more “long tail”-style content than large-scale games. Broadly, VR content on the web will likely cover areas that do not cleanly fit into the app-store models being used as the primary distribution methods by all the major VR hardware providers, or where the content itself is not permitted by the store guidelines. Some high level examples are:

### Video
360° and 3D video are areas of immense interest (for example, see [ABC’s 360° video coverage of the upcoming US election](http://abcnews.go.com/US/fullpage/abc-news-vr-virtual-reality-news-stories-33768357)), and the web has proven massively effective at distributing video in the past. A VR-enabled video player would, upon detecting the presence of VR hardware, show a “View in VR” button, similar to the “Fullscreen” buttons present in today’s video players. When the user clicks that button, a video would render in the headset and respond to natural head movement. Traditional 2D video could also be presented in the headset as though the user is sitting in front of a theater-sized screen, providing a more immersive experience.

### Object/data visualization
Sites can provide easy 3D visualizations through WebVR, often as a progressive improvement to their more traditional rendering. Viewing 3D models (e.g., [SketchFab](https://sketchfab.com/)), architectural previsualizations, medical imaging, mapping, and [basic data visualization](http://graphics.wsj.com/3d-nasdaq/) can all be more impactful, easier to understand, and convey an accurate sense of scale in VR. For those use cases, few users would justify installing a native app, especially when web content is simply a link or a click away.

Home shopping applications (e.g., [Matterport](https://matterport.com/try/)) serve as particularly effective demonstrations of this. Depending on device capabilities, sites can scale all the way from a simple photo carousel to an interactive 3D model on screen to viewing the walkthrough in VR, giving users the impression of actually being present in the house. The ability for this to be a low-friction experience for users is a huge asset for both users and developers, since they don’t need to convince users to install a heavy (and possibly malicious) executable before hand.

### Artistic experiences
VR provides an interesting canvas for artists looking to explore the possibilities of a new medium. Shorter, abstract, and highly experimental experiences are often poor fits for an app-store model, where the perceived overhead of downloading and installing a native executable may be disproportionate to the content delivered. The web’s transient nature makes these types of applications more appealing, since they provide a frictionless way of viewing the experience. Artists can also more easily attract people to the content and target the widest range of devices and platforms with a single code base.

## Example API usage

### Basic VR flow
The first thing that any VR-enabled page will want to do is enumerate the available VR hardware and, if present, determine which one to interact with.

[`navigator.vr.getDisplays`](https://w3c.github.io/webvr/#navigator-getvrdisplays-attribute) returns a [`Promise`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that resolves to a list of available displays. Each [`VRDisplay`](https://w3c.github.io/webvr/#interface-vrdisplay) represents a single piece of physical VR hardware. On desktops this will usually be a headset peripheral; on mobile devices it may represent the device itself in conjunction with a viewer harness (e.g., Google Cardboard or Samsung Gear VR). It may also represent devices with no stereo presentation capabilities but more advanced tracking, such as Tango devices.

```js
let vrDisplay = null;

navigator.vr.getDisplays().then((displays) => {
  if (displays.length > 0) {
    // Use the first display in the array if one is available. If multiple
    // displays are present, you may want to present the user with a way of
    // choosing which display to use.
    vrDisplay = displays[0];
    OnVRAvailable();
  } else {
    // Could not find any VR hardware connected.
  }
}, (err) => {
  // An error occurred querying VR hardware. May be the result of blocked
  // permissions by a parent frame.
});
```

If a VRDisplay is available and has the appropriate capabilities the page will usually want to add some UI to trigger activation of "VR Presentation Mode", where the page can begin sending imagery to the display.

```js
// Initially only canvases with WebGL contexts will be supported.
let glCanvas = document.createElement("canvas");
let gl = glCanvas.getContext("webgl");

function OnVRAvailable() {
  // Most (but not all) VRDisplays are capable of presenting stereo imagery to
  // the user. If the display has that capability the page will want to add an
  // "Enter VR" button (similar to "Enter Fullscreen") that triggers the page
  // to begin showing imagery on the headset.
  if (vrDisplay.capabilities.canPresent) {
    var enterVrBtn = document.createElement("button");
    enterVrBtn.innerHTML = "Enter VR";
    enterVrBtn.addEventListener("click", () => {
      // VRDisplay.requestPresent must be called within a user gesture event
      // like click or touch.
      vrDisplay.requestPresent().then(() => {
        // If the promise resolves the page is now presenting and should begin
        // an animation loop to draw stereo imagery.
        // VRDisplay.requestAnimationFrame should be used in this case to ensure
        // that animations run at the refresh rate of the VR display rather than
        // the default monitor.
        vrDisplay.requestAnimationFrame(OnDrawFrame);
      }, (err) => {
        // May fail for a variety of reasons, including another page already
        // presenting to the display.
      });
    });
    document.body.appendChild(enterVrBtn);

    // The content that will be shown on the display when presenting is defined
    // by the layers list. Throws if the page attempts to add more than
    // vrDisplay.layers.maxLayers
    vrDisplay.layers.appendLayer(new VRCanvasLayer(glCanvas));
  }
}
```

WebVR provides tracking information via the [`VRDisplay.getFrameData` method](https://w3c.github.io/webvr/#dom-vrdisplay-getframedata) method, which developers can poll each frame to get the view and projection matrices for each eye, along with the position, orientation, velocity, and acceleration data for the display. The ideal pattern is to query this information within a [`VRDisplay.requestAnimationFrame`](https://w3c.github.io/webvr/#dom-vrdisplay-requestanimationframe) callback loop, which runs at the refresh rate of the headset (which is frequently higher than that of an average monitor). The matrices provided by the [frameData](https://w3c.github.io/webvr/#vrframedata) can then be used to render the appropriate viewpoint of the scene for both eyes.

```js
// The Frame of Reference indicates what the matricies and coordinates the
// VRDisplay returns are relative to. A VRRelativeFrameOfReference reports
// values relative to the location where the display first began tracking.
let frameOfRef = new VRRelativeFrameOfReference(vrDisplay);

function OnDrawFrame(time) {
  // Queue up another frame callback, just like window.requestAnimationFrame.
  vrDisplay.requestAnimationFrame(OnDrawFrame);

  let frameData = vrDisplay.getFrameData(frameOfRef);

  // Draw the left eye's view of the scene to the left half of the canvas.
  gl.viewport(0, 0, glCanvas.width * 0.5, glCanvas.height);
  drawScene(frameData.leftProjectionMatrix, frameData.leftViewMatrix);

  // Draw the right eye's view of the scene to the right half of the canvas.
  gl.viewport(glCanvas.width * 0.5, 0, glCanvas.width * 0.5, glCanvas.height);
  drawScene(frameData.rightProjectionMatrix, frameData.rightViewMatrix);

  // Notify the headset that the new frame is ready.
  vrDisplay.submitFrame();
}
```

To stop presenting to the `VRDisplay`, the page must call the [`VRDisplay.exitPresent`](https://w3c.github.io/webvr/#dom-vrdisplay-exitpresent) method.

### More sample code
This overview attempts to touch on all the important parts of using WebVR but, for the sake of clarity, avoids discussing advanced uses. For developers who want to dig a bit deeper, there are several working samples of the API in action at https://webvr.info/samples/. These samples each outline a specific part of the API with plenty of code comments to help guide developers through what everything is doing.

## Appendix A: I don’t understand why this is a new API. Why can’t we use…

### `DeviceOrientation` Events
The data provided by a `VRPose` instance is similar to the data provided by `DeviceOrientationEvent`, with some key differences:

* It’s an explicit polling interface, which ensures that new input is available for each frame. The event-driven `DeviceOrientation` data may skip a frame, or may deliver two updates in a single frame, which can lead to disruptive, jittery motion in a VR application.
* `DeviceOrientation` events do not provide positional data, which is a key feature of high-end VR hardware.
* More can be assumed about the intended use case of `VRDisplay` data, so optimizations such as motion prediction can be applied.
* `DeviceOrientation` events are typically not available on desktops.

That being said, however, for some simple VR devices (e.g., Cardboard) `DeviceOrientation` events provide enough data to create a basic [polyfill](https://en.wikipedia.org/wiki/Polyfill) of the WebVR API, as demonstrated by [Boris Smus](https://twitter.com/borismus)’ wonderful [`webvr-polyfill` project](https://github.com/borismus/webvr-polyfill). This provides an approximation of a native implementation, allowing developers to experiment with the API even when unsupported by the user’s browser. While useful for testing and compatibility, such pure-JavaScript implementations miss out on the ability to take advantage of VR-specific optimizations available on some mobile devices (e.g., Google Daydream-ready phones or Samsung Gear VR’s compatible device lineup). A native implementation on mobile can provide a much better experience with lower latency, less jitter, and higher graphics performance than can a `DeviceOrientation`-based one.

### WebSockets
A local [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) service could be set up to relay headset poses to the browser. Some early VR experiments with the browser tried this route, and some non-VR tracking devices (most notably [Leap Motion](https://www.leapmotion.com/)) have built their JavaScript SDKs around this concept. Unfortunately, this has proven to be a high-latency route. A key element of a good VR experience is low latency. Ideally, the movement of your head should result in an update on the display (referred to as “motion-to-photons time”) in 20ms or fewer. The browser’s rendering pipeline already makes hitting this goal difficult, and adding additional overhead for communication over WebSockets only exaggerates the problem. Additionally, using such a method requires users to install a separate service, likely as a native app, on their machine, eroding away much of the benefit of having access to the hardware via the browser. It also falls down on mobile where there’s no clear way for users to install such a service.

### The Gamepad API
Some people have suggested that we try to expose VR headset data through the [Gamepad API](https://w3c.github.io/gamepad/), which seems like it should provide enough flexibility through an unbounded number of potential axes. While it would be technically possible, there are a few properties of the API that currently make it poorly suited for this use.

* Axes are normalized to always report data in a `[-1, 1]` range. That may work sufficiently for orientation reporting, but when reporting position or acceleration, you would have to choose an arbitrary mapping of the normalized range to a physical one (i.e., `1.0` is equal to 2 meters or similar). But that forces developers to make assumptions about the capabilities of future VR hardware, and the mapping makes for error-prone and unintuitive interpretation of the data.
* Axes are not explicitly associated with any given input, making it difficult for users to remember if axis `0` is a component of devices’ position, orientation, acceleration, etc.
* VR device capabilities can differ significantly, and the Gamepad API currently doesn’t provide a way to communicate a device’s features and its optical properties.
* Gamepad features such as buttons have no clear meaning when describing a VR headset and its periphery.

There is a related effort to expose motion-sensing controllers through the Gamepad API by adding a `pose` attribute and some other related properties. Although these additions would make the API more accommodating for headsets, we feel that it’s best for developers to have a separation of concerns such that devices exposed by the Gamepad API can be reasonably assumed to be gamepad-like and devices exposed by the WebVR API can be reasonably assumed to be headset-like.

### These alternatives don’t account for display
It’s important to realize that all of the alternative solutions offer no method of displaying imagery on the headset itself, with the exception of Cardboard-like devices where you can simply render a fullscreen split view. Even so, that doesn’t take into account how to communicate the projection or distortion necessary for an accurate image. Without a reliable display method the ability to query inputs from a headset becomes far less valuable.

# TruexAdRenderer Web Integration: Quick Guide

This document describes the how to use Infillion's TrueX web ad renderer, a library that allows the presentation of
interactive ads, intended for use in "Full Experience Playback" (FEP) scenarios such as video web sites or applications.

The library provides a class called the `TruexAdRenderer` (TAR). TAR supports running within any web content, 
specifically for desktop and mobile web content, as well as for the "Connected TV" (CTV) platforms, such as Smart TVs
and game consoles, i.e. for the so-called "10 foot" experience.

## Setup
The TrueX ad renderer is available as an `npm` module. For the typical web based development around a `package.json` 
project file, you add the TrueX dependency as follows:
```sh
npm add @truex/ad-renderer
```
this will add an entry in the `"dependencies"` section in the `package.json` file, something like:
```json
    "dependencies": {
        "@truex/ad-renderer": "1.11.0",
```
You then build and run your web app like usual, e.g. invoking `npm start` for webpack-based projects.

Alternatively, if you prefer you can refer to the TAR library directly in a script tag, e.g.
```html
<script src="https://cdn.jsdelivr.net/npm/@truex/ad-renderer@1.11.0"></script>
```
or if you want to always refer to the latest version:
```html
<script src="https://cdn.jsdelivr.net/npm/@truex/ad-renderer@latest"></script>
```

### Code Sample

Once a TrueX ad is detected, the renderer needs to be created and displayed. The following code provides an example of the typical approach of integrating to TAR. For example, it shows how you can call the `init` and `start` methods to get the ad displayed, and to
listen for the key ad events a client publisher needs to respond to, ultimately to control how to resume the main video.

```javascript
import { TruexAdRendererCTV } from '@truex/ad-renderer';

...

videoController.pause();

let adFreePod = false;

const tar = new TruexAdRendererCTV(vastConfigUrl); // or TruexAdRendererDesktop or TruexAdRendererMobile, as appropriate
tar.subscribe(handleAdEvent);

const vastConfig = await tar.init();
let adOverlay = await tar.start(vastConfig);

...

function handleAdEvent(event) {
  const adEvents = tar.adEvents;
  switch (event.type) {
    case adEvents.adFreePod:
      adFreePod = true; // the user did sufficient interaction for an ad credit
      break;

    case adEvents.adError:
      console.error('ad error: ' + event.errorMessage);
      resumePlayback();
      break;
      
    case adEvents.noAdsAvailable:
    case adEvents.adCompleted:
      // Ad is not available, or has completed. Depending on the adFreePod flag, either the main
      // video or the ad fallback videos are resumed.
      resumePlayback();
      break;
  }
}

function resumePlayback() {
    if (adFreePod) {
        // The user has the credit, resume the main video after the ad break.
        videoController.skipAdBreak();
        videoController.resumeMainVideo();
    } else {
        // Continue with playing the remaining fallback ad videos in the ad break.
        videoController.skipCurrentAd();
        videoController.resumeAdBreak();
    }
}
```

## Next Steps

[TruexAdRenderer-Web Documentation](DOCS.md) outlines use of the ad renderer, including its API interface and the observable events it fires.

### Reference Apps

Here is a [reference application example](https://github.com/socialvibe/truex-ctv-web-reference-app) that uses the `TruexAdRenderer`, demonstrating its use from a main video, as well as including several platform launcher projects that demonstrate how to sideload the reference application to various devices.

1. [Generic Reference App](https://github.com/socialvibe/truex-ctv-web-reference-app)
   * Presents a "Generic" CSAI style web app that uses the `TruexAdRenderer`, demonstrating its use from a main video, 
     as well as including several platform launcher projects that demonstrate how to sideload the reference application to various devices.
1. [Google Ad Manager: Server-side Ad Insertion Reference App](https://github.com/socialvibe/truex-ctv-web-google-ad-manager-reference-app)
   * Presents a 
1. [Google Ad Manager: Client-side Ad Insertion Reference App](https://github.com/socialvibe/truex-ctv-google-ima-csai-ref-app)

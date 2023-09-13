# true[X]  Web Integrations

This document describes the how to use Infillion's true[X] web ad renderer, a library that allows the presentation of
interactive ads, intended for use in "Full Experience Playback" (FEP) scenarios such as video web sites or applications.

The library provides a class called the `TruexAdRenderer` (TAR). TAR is supports running within any web content, 
specifically for desktop and mobile web content, as well as for the "Connected TV" (CTV) platforms, such as Smart TVs
and game consoles, i.e. for the so-called "10 foot" experience.

## Setup
The true[X] ad renderer is available as an `npm` module. For the typical web app based development around a `package.json` 
project file, one adds the true[X] dependency as follows:
```sh
npm add @truex/ad-renderer
```

### Code Sample

The following code provdes an example of the style of how to integration to TAR, once the video has detected a Truex ad.
For example, how to do the `init` and `start` calls to get the ad displayed, as well as listening for the key ad events
a client publisher needs to respond to, ultimately to control how to resume the main video.

```javascript
import { TruexAdRenderer } from '@truex/ad-renderer';

...

videoController.pause();

let adFreePod = false;

const tar = new TruexAdRenderer(vastConfigUrl);
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

[TruexAdRenderer-Web Documentation](DOCS.md) outlines use of the ad renderer, including its API interface and the observable events it fires

### Reference Apps

1. [Generic Reference App](https://github.com/socialvibe/truex-ctv-web-reference-app)
1. [Google Ad Manager: Server-side Ad Insertion Reference App](https://github.com/socialvibe/truex-ctv-web-google-ad-manager-reference-app)
1. [Google Ad Manager: Client-side Ad Insertion Reference App](https://github.com/socialvibe/truex-ctv-google-ima-csai-ref-app)

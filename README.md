# true[X]  Web Integrations
Documentation and reference apps for integrating true[X]'s CTV web ad renderer


## Setup
This document describes the initial steps needed to make use of the `TruexAdRenderer` in an HTML5 web application intended for Smart TVs and game consoles, i.e. for the so-called "10 foot" experience.

The true[X] ad renderer is available as an `npm` module. For the typical web app based around a `package.json` project file, one adds the true[X] dependency as follows:
```sh
npm add @truex/ctv-ad-renderer
```

## Next Steps

[TruexAdRenderer-Web Documentation](DOCS.md) outlines use of the ad renderer, including its API interface and the observable events it fires

### Reference Apps

1. [Generic Reference App](https://github.com/socialvibe/truex-ctv-web-reference-app)
1. [Google Ad Manager: Server-side Ad Insertion Reference App](https://github.com/socialvibe/truex-ctv-web-google-ad-manager-reference-app)
1. [Google Ad Manager: Client-side Ad Insertion Reference App](https://github.com/socialvibe/truex-ctv-google-ima-csai-ref-app)

### Code Sample

```javascript
import { TruexAdRenderer } from '@truex/ctv-ad-renderer';

...

videoController.pause();

tar = new TruexAdRenderer(vastConfigUrl, {supportsUserCancelStream: true});
tar.subscribe(handleAdEvent);

return tar.init()
  .then(vastConfig => {
    return tar.start(vastConfig);
  })
  .then(newAdOverlay => {
    adOverlay = newAdOverlay;
  })
  .catch(handleAdError);

...

function handleAdEvent(event) {
  const adEvents = tar.adEvents;
  switch (event.type) {
    case adEvents.adError:
      handleAdError(event.errorMessage);
      break;

    case adEvents.adStarted:
      // Choice card loaded and displayed.
      videoController.showLoadingSpinner(false);
      break;

    case adEvents.optIn:
      // User started the engagement experience
      break;

    case adEvents.optOut:
      // User cancelled out of the choice card, either explicitly, or implicitly via a timeout.
      break;

    case adEvents.adFreePod:
      adFreePod = true; // the user did sufficient interaction for an ad credit
      break;

    case adEvents.userCancel:
      // User backed out of the ad, now showing the choice card again.
      break;

    case adEvents.userCancelStream:
      // User backed out of the choice card, which means backing out of the entire video.
      closeAdOverlay();
      videoController.closeVideoAction();
      break;

    case adEvents.noAdsAvailable:
    case adEvents.adCompleted:
      // Ad is not available, or has completed. Depending on the adFreePod flag, either the main
      // video or the ad fallback videos are resumed.
      closeAdOverlay();
      resumePlayback();
      break;
  }
}

function handleAdError(errOrMsg) {
  console.error('ad error: ' + errOrMsg);
  if (tar) {
    // Ensure the ad is no longer blocking back or key events, etc.
    tar.stop();
  }
  closeAdOverlay();
  resumePlayback();
}
```

## 

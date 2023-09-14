[//]: # (Not sure if I should show the Infillion one instead, or I could just hide it entirely.)
[//]: # (![TrueX logo]&#40;media/truex.png&#41;)

# TruexAdRenderer Web Integration

## Contents

* [Overview](#overview)
  * [Supported Platforms](#supported-platforms)
* [Product Flows](#product-flows)
* [How to use TruexAdRenderer](#how-to-use-truexadrenderer)
    * [Setup](#setup)
    * [When to Show a TrueX Ad](#when-to-show-a-truex-ad)
    * [Code Sample](#code-sample)
    * [Handling Ad Events](#handling-ad-events)
      * [Terminal Events](#terminal-events)
    * [Handling Ad Elimination](#handling-ad-elimination)
* [TruexAdRenderer API](#truexadrenderer-api)
    * [`TruexAdRenderer` Methods](#truexadrenderer-methods)
        * [`constructor`](#constructor)
        * [`init`](#init)
        * [`start`](#start)
        * [`stop`](#stop)
        * [`pause`](#pause)
        * [`resume`](#resume)
        * [`subscribe`](#subscribe)
        * [`unsubscribe`](#unsubscribe)
    * [`TruexAdRenderer` Ad Events](#truexadrenderer-ad-events)
        * [`adFetchCompleted`](#adfetchcompleted)
        * [`adStarted`](#adstarted)
        * [`adDisplayed`](#addisplayed)
        * [`adCompleted`](#adcompleted)
        * [`adError`](#aderror)
        * [`noAdsAvailable`](#noadsavailable)
        * [`adFreePod`](#adfreepod)
        * [`userCancelStream`](#usercancelstream)
        * [`optIn`](#optin)
        * [`optOut`](#optout)
        * [`skipCardShown`](#skipcardshown)
        * [`userCancel`](#usercancel)
        * [`popupWebsite`](#popupwebsite)
        * [`xtendedViewStarted`](#xtendedviewstarted)

## Overview

In order to support interactive ads on on mobile and desktop web sites, as well as web-enabled smart TVs and game consoles, 
Infillion has created the npm package `@truex/ad-renderer` to add to host web sites and web apps that can renderer TrueX interactive ads and leverage your app's existing ad server and content 
delivery mechanism (e.g. SSAI).

For simplicity, publisher implemented code will be referred to as "host app code" while TrueX implemented code will be referred to as "renderer code" or TAR.

This library provides a class, `TruexAdRenderer`, that will need to be instantiated, initialized, and displayed by the host app code, as described below in [TruexAdRenderer API](#truexadrenderer-api).

At this point, the renderer code will take on the responsibility of requesting ads from TrueX server, creating the native UI for the TrueX choice card and interactive ad unit, as well as communicating events to the host app code when action is required.

The host app code will still need to parse out the ad response, detect when a TrueX ad is supposed to display, pause the stream, instantiate `TruexAdRenderer`, and handle any events emitted by the renderer code.

It will also need to handle skipping ads in the current ad pod, if it is notified to do so.

### Supported Platforms

The TruexAdRender runs on any modern web browser, so mobile and desktop platforms are naturally supported.

For connected tv platforms such as smart TVs and gaming consoles, the following platforms are supported:
* Comcast X1 / Flex
* Fire TV / Android TV
* Vizio Smartcast
* LG WebOS
* Samsung Tizen
* PS4, PS5
* XboxOne

## Product Flows

There are two distinct product flows supported by `TruexAdRenderer`: Sponsored Stream (full-stream ad-replacement) and Sponsored Ad Break (mid-roll ad-replacement).

In a Sponsored Ad Break flow, once the user hits a mid-roll break with a TrueX tag flighted, they will be shown a "choice-card" offering them the choice between watching a normal set of video ads or a fully interactive TrueX ad:

![choice card](media/choice_card.png)

_**Fig. A** example TrueX mid-roll choice card_

If the user opts for a normal ad break, or if the user does not make a selection before the countdown timer expires, the TrueX UI will close and playback of normal video ads can continue as usual.

If the user opts to interact with TrueX, an interactive ad unit will be shown to the user:

![ad](media/ad.png)

_**Fig. B** example TrueX interactive ad unit_

The requirement for the user to "complete" this ad is for them to spend at least the allotted time on the unit and for at least one interaction (e.g. navigating anywhere through the ad).

![true attention timer example](media/true-attention-timer-example.png)

_**Fig. C** example TrueX attention timer_

Once the user fulfills both requirements, a "Watch Your Show" button will appear in the bottom right, which the user can select to exit the TrueX ad. Having completed a TrueX ad, the user will be returned directly to content, skipping the remaining ads in the current ad pod.

The Sponsored Stream flow is quite similar. In this scenario, a user will be shown a choice-card in the pre-roll:

![choice card](media/choice_card.png)

_**Fig. D** example TrueX pre-roll choice card (full-stream replacement)_

Similarly, if the user opts-in and completes the TrueX ad, they will be skipped over the remainder of the pre-roll ad break. However, every subsequent mid-roll break in the current stream will also be skipped over. In this case instead of the regular pod of video ads, the user will be shown a "hero card" (also known as a "skip card"):

![skip card](media/skip_card.png)

_**Fig. E** example true\[X\] mid-roll skip card_

This messaging will be displayed to the user for several seconds, after which they will be returned directly to content.

## How to use TruexAdRenderer

### Setup
The TrueX ad renderer is available as an `npm` module via the `@truex/ad-renderer` package. For the typical web based development around a `package.json`
project file, one adds the TrueX dependency as follows:
```sh
npm add @truex/ad-renderer
```
this will add an entry in the `"dependencies"` section in the `package.json` file, something like:
```json
    "dependencies": {
        "@truex/ad-renderer": "1.11.0",
```
One then builds and runs their web app like usual, e.g. invoking `npm start` for webpack-based projects.

### When to show a TrueX Ad

Upon receiving an ad break from your ad provider, you should be able to detect whether or not TrueX is returned in 
any of the pods. In a typical [VAST-based](https://www.iab.com/guidelines/vast) ad integration 
(for example Google Ad Manager or FreeWheel), the `<AdSystem>` element within the ad's VAST tag should have the value 
of `trueX`.

A standard VAST ad response from TrueX contains a `vast_config_url` to pass to TAR within a JSON blob in the
`<AdParameters>` field and a placeholder video in the `<MediaFile>` field. Work with your TrueX contact if you have 
questions about interpreting our ad payloads.

Ad provider vendors differ in the way they convey information back about ad schedules to clients. 
Certain vendors such as Verizon / Uplynk expose API’s which return details about the ad schedule in a JSON object. 
For other vendors, for instance Google DAI / IMA, the TrueX payload will be encapsulated as part of a companion payload
on the returned VAST ad. 

Please work with your Infillion point of contact if you have difficulty identifying the right approach to detecting the 
TrueX placeholder, which will be the trigger point for the ad experience.

Once the player reaches a TrueX placeholder, it should pause, instantiate a `TruexAdRenderer` instasnce and 
invoke the [`init`](#init) and [`start`](#start) methods.

Alternatively, you can call `init` on the `TruexAdRenderer` in preparation for an upcoming placeholder, pre-loading the ad. This will give the `TruexAdRenderer` more time to complete its initial ad request, and will help streamline TrueX load time and minimize wait time for your users. Once the player reaches a placeholder, it can then call `start` to notify the renderer that it can display the unit to the user.

### Code Sample

The following code provides an example of the style of how to integration to TAR, once a Truex ad has been detected
during playback. For example, we can see how to call the `init` and `start` methods to get the ad displayed, and to
listen for the key ad events a client publisher needs to respond to, ultimately to control how to resume the main video.

```javascript
import { TruexAdRenderer } from '@truex/ad-renderer';

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

### Handling Ad Events

Once the renderer has been initialized and started, it will begin to emit ad events (see [`TruexAdRenderer` Ad Events](#truexadrenderer-ad-events)).
These ad events that can be monitored via the [`subscribe`](#subscribe) method as shown in the above code sample.

One of the first events you will receive is `adStarted`. This notifies the host app that the renderer has received an 
ad for the user and has started to show the unit to the user. The app does not need to do anything in response, 
however it can use this event to facilitate a timeout. If an `adStarted` event has not fired within a certain 
amount of time, the host app can proceed to normal video ads.

At this point, the host app code must listen for the renderer's *terminal events* (described below), 
while paying special attention to the `adFreePod` event. 
A *terminal event* signifies that the renderer is done with its activities and the app may now resume playback of its 
stream. 

The `adFreePod` event signifies that the user has earned a credit with TrueX and all linear video ads remaining in the 
current ad break should be skipped. If the `adFreePod` event did not fire before a terminal event is emitted, 
the app should resume playback without skipping any ads, so the user receives a normal video ad payload.

#### Terminal Events

The *terminal events* are:
* `adCompleted`: the user has exited the TrueX ad
* `userCancelStream`: the user intends to exit the current stream entirely
* `noAdsAvailable`: there were no TrueX ads available to the user
* `adError`: the renderer encountered an unrecoverable error

It's important to note that the player should not immediately resume playback once receiving the `adFreePod` event 
-- rather, it should note that it was fired and continue to wait for a terminal event.

### Handling Ad Elimination

Skipping video ads is completely the responsibility of the host app code. The ad provider SDKs like Google IMA should
provide enough information for the host app to determine where the current pod end-point is, and the app, 
when appropriate, should fast-forward directly to this point when resuming playback.

## TruexAdRenderer API

Here we describe the essential methods of the `TruexAdRenderer` class, as well as ad events that can be listened for.

Note that there are actually 3 possible classes to use:
```javascript
TruexAdRendererCTV
TruexAdRendererDesktop
TruexAdRendererMobile
```
One should use the appropriate one for your platform, so as to allow the correct ad tracking to be used.

If one uses `TruexAdRenderer`, that maps to `TruexAdRendererCTV`

### TruexAdRenderer Methods

#### `constructor`

```javascript
/**
 * Derives a url for a query to download a vast config object.
 *
 * @param {*} params uses as a url if a string, or else an object with a vast_config_url field, or else as
 *   aa vast config object directly.
 *
 * @param {Object} options for optional overrides
 *
 * @param {String} options.showLoadingSpinner if true (default), a loading indicator is shown while the
 *   choice card display elements are loading, i.e. while waiting for image assets to load, etc.
 *
 * @param {String} options.userAdvertisingId The id used for user ad tracking.
 *   If omitted, the existing network_user_id in the vast config url is used.
 *   If that is missing, the fallbackAdvertisingId is used.
 *
 * @param {String} options.fallbackAdvertisingId The id used for user ad tracking when no explicit user ad id is
 *   provided or available from the platform. By default it is a randomly generated UUID. Specifying allows one to
 *   control if limited ad tracking is truly random per ad, or else shared across multiple TAR ad sessions.
 *
 * @param {String} options.supportsUserCancelStream if true, enables the userCancelStream event for back actions
 *   from the choice card. Defaults to false, which means back actions cause optOut/adCompleted events instead.
 *
 * @param {String} options.choiceCardUrlOverride url to load the choice card script from.
 *
 * @param {String} options.containerUrlOverride url to load the container for the engagement ad.
 *
 * @param {String} options.optOutTrackingEnabled override value to set opt out tracking
 *
 * @param {String} options.appId set the application id/ package name for tracking
 *
 * @param {String} options.useWebMAF if true and running on the PS4, all ad videos are played via the WebMAF API,
 *   so as to work around any WebMAF load failures due to the creation of HTML5 Video elements.
 *
 * @param {Object} options.keyMapOverrides provides a map keyed by input actions that describe the key codes
 *   to use to map key events to input actions. This will override the platform's built in support, and is intended
 *   to allow the use of the TruexAdRenderer on unsupported platforms where the key mapping has not been
 *   implemented yet in TAR itself. E.g.
 * @example
 * {
 *     select: [32, 13], // space or enter
 *     back: [27, 8, 127], // esc or backspace or del
 *     moveUp: 38,
 *     moveDown: 40,
 *     moveLeft: 37,
 *     moveRight: 39
 * }
 */
constructor(parameters, options)
```

The usual case is to construct the renderer with the vast config url, e.g. `const tar = new TruexAdRendererCTV(vastConfigUrl)`.

#### `init`

```javascript
/**
 * Queries for the vast config result
 *
 * @returns {Promise} resolves to the resulting the vast config result object
 */
function init()
```

This should be called by the host app code in order to initialize the `TruexAdRenderer` instance. 
The renderer will use the VAST config url passed to it in the constructor and make a request to the TrueX ad server 
to see what ads are available.

You may initialize `TruexAdRenderer` early (event a few seconds before the next pod even starts) in order to give it extra 
time to make the ad request. The renderer will output an `adFetchCompleted` event at completion of this ad request. 
This event can be used to facilitate the implementation of a timeout or loading indicator, and when to make the call to `start`.

#### `start`

```javascript
/**
 * Starts the choice card and engagement flow
 *
 * @param {Object} vastConfig optional defaults to the value obtained via the init() call.
 *
 * @param {HTMLElement} parentElement parent DOM element to add the ad overlay/choice card to.
 *      Defaults to document.documentElement.
 *
 * @return {Promise} promise that completes when the choice card loads, resolves to the div holding the ad overlay.
 */
function start(vastConfig, parentElement)
```

This should be called by the host app code when the app is ready to display the TrueX unit to the user. 
This can be called anytime after the renderer is initialized.

The host app should have as much extraneous UI hidden as possible, including player controls, status bars and soft buttons/keyboards. 
On constrained devices, especially connected TV devices, it is preferrable to also unload the main video if possible, 
resuming it after the ad unit is completed.

After calling `start`, the host app code should wait for a [terminal event](#terminal-events) before taking further 
actions, while keeping track of whether the [`adFreePod`](#adfreepod) event has fired.

In a non-error flow, the renderer will first wait for the ad request triggered in `init` to finish if it has not already. 
It will then display the TrueX ad unit to the user in a new component (added to the `TruexAdRenderer` parent component) 
and then fire the [`adStarted`](#adstarted) event. `adFreePod` and other events may fire after this point, 
depending on the user's choices, followed by one of the [terminal events](#terminal-events).

#### `stop`

```javascript
/**
 * Allow ads to be forcibly closed by the host app or test script.
 * (Needs to be possible if the app needs to reset, otherwise back actions can be permanently
 * blocked if the ad is not cleaned up.)
 */
function stop()
```

The `stop` method is only called when the host app needs the renderer to immediately stop and destroy all related resources. 

Examples include:
- the user backs out of the video stream to return to the normal app UI
- there was an unrelated error that requires immediate halt of the current ad unit
- the app code has reached a custom timeout waiting for either the [`adFetchCompleted`](#adfetchcompleted) or [`adStarted`](#adstarted) events

The renderer instance should not be used again after calling `stop` -- please remove all references to it afterwards.

#### `pause`

```javascript
/**
 * Pauses an ad if the hosting app needs to suspend/resume itself.
 * I.e. Any ad videos are paused and countdowns suspended.
 */
function pause()
```

#### `resume`

```javascript
/**
 * Resumes a previously paused ad. I.e. any paused ad videos are resumed, as well as suspended countdowns.
 */
function resume()
```

#### `subscribe`

```javascript
/**
 * Adds a callback subscription to receive ad events.
 *
 * An ad event object as a type field to describe which event has occurred (i.e. one of
 * the {@link adEvents} constants), along with other optional data fields depending on the event.
 *
 * @param type Indicates the type of event to subscribe to. If empty or if a single callback function,
 *   then the function is invoked on all ad events.
 * @param callback Describes the callback when the first arg is the type string.
 */
function subscribe(type, callback)
```

Use `subscribe` to listen to ad events emitted from the renderer. The key ones the host app should listen to are:
```javascript
adFreePod
adCompleted
noAdsAvailable
adError
```

To listen to all ad events, just pass in a single callback as the argument, e.g. `tar.subscribe(handleAdEvent)`.

To listen to a particular ad event, pass in the type to listen to and the callback, e.g. `tar.subscribe('adCompleted', handleAdCompleted)`

The ad event names are available as constants via the renderer's `adEvents` field, e.g. `tar.adEvents.adFreePod`

#### `unsubscribe`

```javascript
/**
 * Removes the callback subscription.
 *
 * @param type Indicates the type of event to unsubscribe from. If empty or if a single callback function,
 *   then the callback is unsubscribed subscribed for all ad events.
 * @param callback Describes the callback when the first arg is the type string.
 */
function unsubscribe(type, callback)
```

### `TruexAdRenderer` Ad Events

#### Main Ad Events

The following events signal the main flow of the `TruexAdRenderer` and may require action by the host app:

#### `adFetchCompleted`

This event fires in response to the `init` method when the TrueX ad request has successfully completed and the ad 
is ready to be presented. The host app may use this event to facilitate a loading screen for pre-rolls, or to 
facilitate an ad request timeout for mid-rolls.

For example: `init` is called for the pre-roll slot, and the host app code shows a loading indicator while waiting 
for `adFetchCompleted`. Then, either the event is received (and the app can call `start`) or a specific timeout is 
reached (and the app can discard the `TruexAdRenderer` instance and resume playback of normal video ads).

Another example: `init` is called well before a mid-roll slot to give the renderer a chance to preload its ad. 
If `adFetchCompleted` is received before the mid-roll slot is encountered, then the app can call `start` to 
immediately present the TrueX ad. If not, the app can wait for a specific timeout (if not already reached, 
in case the user has seeked to activate the mid-roll) before removing the `TruexAdRenderer` instance and 
resuming playback with normal video ads.


#### `adStarted`

This event fires after the `start` call, when the TrueX UI is being constructed and connected to the web page.

#### `adDisplayed`

This event fires once the TrueX UI is loaded and visible, i.e. all image assets are loaded, etc.

This is useful if the host app is implementing its own loading indiciator UX (i.e. its own loading spinner), 
and needs to know when the ad UI is actually ready to be disabled without delays.

#### `adCompleted`

This is a [terminal event](#terminal-events). This event will fire when the TrueX unit is finished with its activities
-- at this point, the app should resume playback and remove the TrueX ad renderer from its UI

Here are some examples where `adCompleted` will fire:

* The user opts for normal video ads (not TrueX)
* The choice card timeout runs out
* The user completes the TrueX ad unit
* After a "skip card" has been shown to a user for its duration

The event data field for this event is:

* `timeSpent`: The amount of time (in seconds) the user spent on the TrueX unit

#### `adError`

This is a [terminal event](#terminal-events). This event will fire when the TrueX unit has encountered an 
unrecoverable error. The host app code should handle this the same way as an `adCompleted` event -- resume playback
and remove the TrueX ad renderer from its UI.

The event data field for this event is:

* `errorMessage`: A description of the cause of the error.

#### `noAdsAvailable`

This is a [terminal event](#terminal-events). This event will fire when the TrueX unit has determined it has no 
ads available to show the current user. The host app code should handle this the same way as an `adCompleted` 
event - resume playback and remove the TrueX ad renderer from its UI.

#### `adFreePod`

This event will fire when the user has earned a credit with TrueX. The host app code should notate that this event 
has fired, but should not take any further action. Upon receiving a [terminal event](#terminal-events), 
if `adFreePod` was fired, the app should skip all remaining ads in the current slot. If it was not fired, the app 
should resume playback without skipping any ads, so the user receives a normal video ad payload.

#### `userCancelStream`

This is a [terminal event](#terminal-events), and is only enabled when the `supportsUserCancelStream` property is 
set to `true` when triggering the [`init`](#init) action.

When enabled, the renderer will fire this event when the user intends to exit the stream entirely. The app, at this 
point, should treat this the same way it would handle any other "exit" action from within the stream -- in most cases
this will result in the user being returned to an episode/series detail page.

If this event is not enabled, the renderer will emit an [`adCompleted`](#adcompleted) event instead.

#### Informative Ad Events

All following events are used mostly for tracking purposes -- no action is generally required.

#### `optIn`

This event will fire if the user selects to interact with the TrueX interactive ad.

Note that this event may be fired multiple times if a user opts in to the TrueX interactive ad and subsequently 
backs out.

#### `optOut`

This event will fire if the user opts for a normal video ad experience.

The event data field for this event is:

* `userInitiated`: This will be set to `true` if this was actively selected by the user, `false` if the user simply 
allowed the choice card countdown to expire.

#### `skipCardShown`

This event will fire anytime a "skip card" is shown to a user as a result of completing a TrueX Sponsored Stream 
interactive in an earlier pre-roll.

#### `userCancel`

This event will fire when a user backs out of the TrueX interactive ad unit after having opted in. 
This would be achieved by tapping the "Yes" link to the "Are you sure you want to go back and choose a different 
"ad experience" prompt inside the TrueX interactive ad. The user will be subsequently taken back to the Choice Card 
(with the countdown timer reset to full).

Note that after a `userCancel`, the user can opt-in and engage with an interactive ad again, so more `optIn` 
or `optOut` events may then be fired.

#### `popupWebsite`

On mobile platforms, ad units with links to external web sites fire this event when the user clicks on the 
web site link, e.g. typically via a "Learn More" link.

On mobile platforms, navigating to an external web site means leaving the current application. As such this ad event 
is used to communicate the user's intention, and the host app needs to open the external web page as appropriate 
for the platform. E.g. for Android, an intent is raised to open the web page in the phone's browser.  

The event data field for this event is:

* `url`: Indicates the web page to navigate to.

#### `xtendedViewStarted`

This event fires for TrueX ad creatives that their own fallback extended videos built in. 

From the host app's perspective, nothing more is needed to be done, and the regular ad event flow processing will occur, 
just with the extended ad content instead.
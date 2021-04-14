# Platform Issues

## PS4
* Embedded <video> now play fine, including autoplay in the latest WebMAF SDK.
  * Note however that we are using this setting in the webmaf_settings.ini file:
    ```properties 
    ; Use the latest browser for better debugging.
    WebBrowser=2018
    ```
* There are some video playback issues still if you try to create a new <video> tag without closing the old one properly. 
  * One needs to do video.src = “” first to clear it out.

* Multiple videos do not seem to work, one needs to clear out the old video before switching to a new one.

* Setting the initial time before playback does not work.
  * One has to wait until playback is underway before a seek can be done.

* Attempting to play a video before it is laid on in the DOM can cause video hangs.
  * E.g. attempting playback when “display: none” for the video’s style.
  * Or attempting playback when the video’s parent DOM element is still detached.

* Need to be careful with 3rd party libraries that still use ‘const’ in strict mode in their transpiled deployed library code.
  * E.g. we cannot use the uuid npm module version 7.0.3.
  * It fails with the error:
    ```
    Unexpected keyword ‘const’. Const declarations are not supported in strict mode.
    ``` 
  * Instead we have to use uuid 3.4.0

* Polyfills on the PS4 turned out to be mostly reasonable.
  * Babel/core-js transpilation did most of the work.
  * Missing fetch was polyfilled using the watwg-fetch module. I would prefer to see if it is available in core-js.
  * String.split was downgraded to not support splits with regex args, breaking fetch calls. 
    * For some reason loading the choicecard-ctv lib did this.
    * Filed CTV-1808: improve choicecard-ctv library polyfill support for String.split

* Updates from CTV-2095: Test ads/behaviors on PS4:
  * Embedded videos now working fine.
  * Play only one video at a time.
  * Add videos “lazily”, we now use wrapper divs to hold the video poster, only create an actual <video> element only when we are about to do playback.
  * Don’t invoke .play() in the same thread as creating a <video> element. We now invoke .play() about 20ms later via setTimeout().
  * Remove <video> elements when playback is completed to save memory.
  * Remove all <video> elements when suspending an app, restore just the one needed for the current play back on resume, as per the above points.
  * We no longer attach `<audio>` elements to the DOM. Playback works just fine with it detached.
  * For engagement ads we now just have one single active audio instead and one single current sound effect audio instance, that we just change the src for for different audio playbacks. Using multiple copies seem to consume more memory.
  * For both video and audio, take care to set the .src attribute to the empty string, i.e. `‘’` when removing it. This seems to help actually unload the media itself on the PS4. 
    * Without this I was getting consistent crashes in the CTV reference app when opening a choice card with voiceover, and returning to the main video.
  * `invert(1)` css filters do not work on the PS4. We had to work around it using the css mask-images instead.
  * For the PS4, seeking only works when the video is loaded with the duration known. Otherwise setting the video.currentTime in advance of starting playback does not work, unlike almost all other platforms.
    * This impacts resuming a video after an ad is finished, since we have to recreate the video at the position just after the ad slot.
    * The work around is to listen to the “playing” event, and then do the initial seek then.
  
## PS5
Platform specific work arounds:
* `visibilitychanged` event does not get fired on the app suspend/resume.
  * Needed to explicit monitor these events instead via the MedaiKit api:
  ```js
            const msdk = window.msdk;
            msdk.addEventListener('onBackground', this.suspendAd);
            msdk.addEventListener('onForeground', this.resumeAd);
            msdk.addEventListener('onSystemUI', this.suspendAd);
            msdk.addEventListener('onAppResume', this.resumeAd);
  ```

* Sounds clips are not currently supported on the PS5, so we use .mp4 video streams instead, which do work.
  * We now use ffmpeg to auto-create corresponding .mp4 from uploaded audio assets, typically .mp3 files.
  
* Like the PS4, only a single media stream can be played at once, so audio is ignored if there is another current video playing, sound effects are ignored if there is an audio or video element playback.

* Audio clip invocations for unsupported sound formats block subsequent video play calls.
  * We thus invoke sound effect play calls in a background timeout to plae them after other play() calls.

## AndroidTV/FireTV
* Does not support http: image queries when running under https

* Need to also vet “pure web” app type.

* RGBA hexadecimal color values are not supported, e.g. #34DF3401; use rgba() instead.

* Need to take care with history.back interception via window ‘popstate’ event handling.
  * In general Web apps on the FireTV cannot (properly) intercept the back action remote control key event (i.e. ESC or key code 27).
  * See https://developer.amazon.com/docs/fire-tv/web-app-faq.html
  * TAR needs to do its own handling to intercept backing out of an ad so as to show the “Are You Sure” overlay.
  * See the FireTV README.

* Video fullscreen animation looks too jerky, at least on the FireTV stick (Gen 2).
  * See CTV-2149 video fullscreen animations too jerky on FireTV stick

* Advertising id has a web query that works only for “pure” web apps or cordova based ones.
  * E.g. also in the Amazon Web App Tester
  * But pure native apps with a simple Web View will need to implement the ad id query natively, inject it into TAR.
  * See CTV-2152: use FireTV advertising id in vast config queries
  * NOTE: our latest practice is to require the host app developer to pass in and desired advertising id in via the `userAdvertisingId` option to the `TruexAdRenderer` constructor.
  
* Video playback for Android TV would often not render unless some other part of the web page was also changing.
  * The work around was to overlay a small div that toggle some invisible text on and off on every video time update.

## Vizio
* Need to avoid the use of window.scrollTo to scroll large pages?
* Will need to vet any https vs http issues, this platform is very strict about https compliance.
* Skyline is running fine.

## Tizen
* Need to avoid the use of window.tizen api object

* We currently use it derive key code mappings.

* We can hard code those instead into tmx_platform.js

* Developers need to know to register each key they wish the app to use.
  * They will tend to do this anyway.
  * True[x] will need these keys defined:
    ```javascript
    var keyNames = [
      "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "Exit",
      "MediaFastForward", "MediaPause", "MediaPlay",
      "MediaPlayPause", "MediaRewind", "MediaStop",
      "MediaTrackNext", "MediaTrackPrevious", "Extra"
    ];
    keyNames.forEach(keyName => {
      tizen.tvinputdevice.registerKey(keyName);
    });
    ```

## LG
* Need to avoid the use of window.scrollTo to scroll large pages?

* LG WebOS 4 has a high screen resolution, this exercises layout issues
  * Auto-scaling as demonstrated by Skyline is working fine.

* LG dev account needed to side load, seems to auto-logout and uninstall test apps after some weeks.
  * Normally not a problem, but does impact the notion of having permanently deploy regression test devices.
  * This means that it will be a part of the process to periodically manually reinstall Skyline to the LG TVs.

# XBoxOne
* Test if there is CSS image mask support

* Skyline is running fine

* .jsproj web app projects (e.g. provide a single web page url) are no longer supported in Visual Studio 2019.
  * Nevertheless, I find this project type is so useful that I expect people to continue to use it via Visual Studio 2017.

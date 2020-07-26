---
layout: post
title: Analysis of a Commercial Browser Fingerprinting Service
tags: [javascript, reverse engineering]
---

[Augur](https://web.archive.org/web/20190503172035/https://www.augur.io/) (2013-2017) was a commercial browser fingerprinting service, offered primarily to AdTech companies.  In this post I will analyze their tracking script and describe how it works and what information it collects.  The original copy of the script ([https://cdn.augur.io/augur.min.js](https://cdn.augur.io/augur.min.js)) is no longer up; however, the Internet Archive still has a copy at [https://web.archive.org/web/20170609184536js_/https://cdn.augur.io/augur.min.js](https://web.archive.org/web/20170609184536js_/https://cdn.augur.io/augur.min.js).

## Deobfuscating
A first glance at the script shows a bunch of seemingly random characters (including emojis).  The was a hint to me that they were probably passing a big string into some deobfuscation function and ```eval```ing it.  Passing the script through [beautifier.io](https://beautifier.io) shows exactly that.  The deobfuscation function has the following contents:

            ...
            if (!''.replace(_(12), String)) {
                var z = {};
                for (i = 0; i < 686; i++) {
                    var y = x(i);
                    z[y] = r[i] || y
                }
                t = _(15);
                y = function(a) {
                    return z[a] || a
                };
                o = o.replace(t, y)
            } else {
                for (j = a[a.length - 1] - 1; j >= 0; j--) {
                    (r[j]) ? o = o.replace(new RegExp('\b' + (j < 63 ? c.charAt(j) : c.charAt((j - 63) / 63) + c.charAt((j - 63) % 63)) + '\b', '\x67'), r[j]): 0
                }
            }
            return o.replace(_(11), "\n").replace(_(16), "\"")

To get the deobfuscated output, we can just copy the return statement but replace ```return``` with ```console.log```.

            console.log(o.replace(_(11), "\n").replace(_(16), "\""));
            return o.replace(_(11), "\n").replace(_(16), "\"")

Now, we can just run the script in our browsers developer console and copy-paste the large output. The deobfuscated version can be found at [https://pastebin.com/5Gdgxkxu](https://pastebin.com/5Gdgxkxu).

## Fingerprinted Attributes
We will now look at the attributes collected by the script.

    * Cookie enabled status - navigator.cookieEnabled
    * User agent - navigator.userAgent
    * Emails in input forms - all non-hidden or blurred inputs are checked with a Regex for emails which are then associated with a Device ID
    * If the site is using the Facebook SDK (window.FB is defined), call window.FB.api("/me") and return username and email.
    * Do Not Track status - navigator.doNotTrack || navigator.msDoNotTrack
    * Audio information - Creating a new HTML5 AudioContext and getting its sampleRate, maxChannelCount, numberOfInputs, numberOfOutputs, channelCount, channelCountMode, channelInterpretation
    * Timezone - Creating a new Date object and calling getTimezoneOffset()
    * Adblock status - Creating a new element with class "adsbox," adding some text to it, appending it to the body, and checking that its height equals 0 (if yes, then an adblocker is enabled)
    * Canvas fingerprinting - Creating a new Canvas, drawing "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/" over it in multiple different colors with a rainbow background, then using the toDataURL function to get the base64 PNG representation
    * Private Browser Mode detection - Trying to write data using localStorage, indexedDB, or requestFileSystem
    * Font detection - Getting the width/height of the browser "fallback" (default) serif, sans-serif, and monospaces fonts, then measuring the width of a bunch of fonts (Augur's list has over 800) and seeing if there is any differences between the tested font and the defaults
    * Media devices (speakers, cameras, microphones) enumeration - navigator.mediaDevices.enumerateDevices
    * Installed plugins enumeration - navigator.plugins
    * Spoofing detection
        * Browser build number (always 20030107 on Chrome/Safari) - navigator.productSub
        * toString length of eval (33 on Chrome, 37 on Firefox/Safari, 39 on Internet Explorer) - eval.toString().length
        * Error toSource support (Firefox only)
    * Touch support - navigator.maxTouchPoints || navigator.msMaxTouchPoints || "ontouchstart" in window || attempting to create a TouchEvent
    * Local and remote IP addresses with WebRTC STUN
    * GPU Information - Creating a Canvas with a WebGL context
        * MAX_TEXTURE_MAX_ANISOTROPY_EXT
        * getShaderPrecisionFormat
        * ALIASED_LINE_WIDTH_RANGE
        * VERSION
        * SHADING_LANGUAGE_VERSION
        * VENDOR
        * RENDERER
        * getContextAttributes().antialias
        * RED_BITS
        * GREEN_BITS
        * BLUE_BITS
        * ALPHA_BITS
        * DEPTH_BITS
        * STENCIL_BITS
        * MAX_RENDERBUFFER_SIZE
        * MAX_COMBINED_TEXTURE_IMAGE_UNITS
        * MAX_CUBE_MAP_TEXTURE_SIZE
        * MAX_FRAGMENT_UNIFORM_VECTORS
        * MAX_TEXTURE_IMAGE_UNITS
        * MAX_TEXTURE_SIZE
        * MAX_VARYING_VECTORS
        * MAX_VERTEX_ATTRIBS
        * MAX_VERTEX_TEXTURE_IMAGE_UNITS
        * MAX_VERTEX_UNIFORM_VECTORS
        * ALIASED_LINE_WIDTH_RANGE
        * ALIASED_POINT_SIZE_RANGE
        * MAX_VIEWPORT_DIMS
        * getSupportedExtensions().sort().toString()
        * UNMASKED_VENDOR_WEBGL
        * UNMASKED_RENDERER_WEBGL
    * Device Pixel Ratio - window.devicePixelRatio
    * Device DPI (IE only) - window.deviceXDPI/window.deviceYDPI
    * Screen resolution - screen.width/screen.height
    * Screen color depth - screen.colorDepth
    * Available resolution - screen.availWidth/screen.availHeight
    * CPU cores - navigator.hardwareConcurrency
    * Platform - navigator.platform
    * Languages - navigator.languages
    * Main Language - navigator.language || navigator.browserLanguage || navigator.systemLanguage || navigator.userLanguage
    * Battery level - navigator.getBattery()
    * Java enabled - navigator.javaEnabled()

## Conclusion
I was shocked to find that Augur was silently collecting user's Facebook information and all emails put in forms.  Apparently, this has been reported on [before](https://freedom-to-tinker.com/2018/04/18/no-boundaries-for-facebook-data-third-party-trackers-abuse-facebook-login/).  Otherwise, their fingerprinting was pretty comprehensive, but did not really feature any innovative or novel techniques.

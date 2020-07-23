---
layout: post
title: Understanding Distil Networks Anti-Bot Code
tags: [javascript, reverse engineering]
---

## Introduction
I was browsing [Whitepages](https://whitepages.com) when I noticed a request to [https://www.whitepages.com/dstl-wp.js](https://www.whitepages.com/dstl-wp.js)[1].  After beautifying the code, I found the string "Distil" in the code many times.  "Distil" refers to Distil Networks, one of the leading bot detection vendors until its acquisition by Imperva in 2019.  Whitepages seems to use them for their anti-scraping abilities.

## The Code
### Fingerprinting

The first part of the code is the fingerprinting module.  This code is not really obfuscated, just minified (unlike the headless detection code later).  It collects a bunch of information from the browser using (mostly) standard techniques.  [According to Imperva](https://www.imperva.com/blog/the-evolution-of-hi-def-fingerprinting-in-bot-mitigation/), their "Hi-def" fingerprints encompass over 200 device attributes and has a 0% false positive rate.  The fingerprinting code appears to be based off Valve's [fingerprintjs2](https://github.com/fingerprintjs/fingerprintjs2/blob/master/fingerprint2.js) (identical function names and contents in many cases), albeit with some additions.

This code collects (via JavaScript):

    * Browser User-Agent - ```navigator.userAgent```
    * Browser language - ```navigator.language```
    * System language (IE only) - ```navigator.systemLanguage```
    * User language (IE only) - ```navigator.userLanguage```
    * Screen resolution - ```screen.width```/```screen.height```
    * Timezone offset in hours - ```(new Date).getTimezoneOffset() / -60```
    * IndexedDB support - ```!!window.indexedDB```
    * HTMLElement addBehavior support (IE only) - ```!!document.body.addBehavior```
    * WebSQL support - ```!!window.openDatabase```
    * CPU Class (IE and Firefox only) - ```navigator.cpuClass || navigator.oscpu```
    * Browser platform - ```navigator.platform```
    * Do Not Track support - ```navigator.doNotTrack```
    * Browser plugins (non IE) - ```navigator.plugins``` enumeration
    * ActiveX plugins (IE only) - trying/catching creating ActiveXObjects with names from a list of popular plugins
    * Canvas fingerprinting - standard method.  Creating a Canvas element, drawing some rectangles and arcs on it, along with the string "Cwm fjordbank glyphs vext quiz,", then getting the base64 PNG representation via toDataURL and hashing it
    * Font detection - using a method popularized by [Lalit Patel](https://gist.githubusercontent.com/szepeviktor/d28dfcfc889fe61763f3/raw/6bbcda9766e65fd81e951400e0ceca03c14b6fad/fontdetect.js).  Essentially, they measure the dimensions of the browsers default (fallback) monospace, sans-serif, and serif fonts.  They then measure the dimensions of a large list of fonts.  If there is a difference in dimensions between a font in the list and the fallback font, the font is installed.
    * Video codec detection - creating a new video element and checking whether ```video.canPlayType()``` returns true for various codecs, including ```video/ogg; codecs="theora"``` (Theora with OGG), ```video/mp4; codecs="avc1.42E01E``` (H264 MP4), ```video/webm; codecs="vp8, vorbis"``` (WEBM).
    * Audio codec detection - creating an audio element and checking whether ```audio.canPlayType``` returns true for ```audio/ogg; codecs="vorbis"``` (OGG Vorbis), ```audio/mpeg;``` (MP3), ```audio/wav; codecs="1"``` (WAV), ```audio/x-m4a;``` (M4A), ```audio/aac;``` (AAC)
    * WebGL fingerprinting - Creates a Canvas element with a WebGL context.  Then draws a gradient object with shaders and gets the base64 string with toDataURL.  They then check for the system's max texture anisotropy.  After that, they gather GPU vendor and renderer information with the ```WEBGL_debug_renderer_info``` extension.
    * Touch support - ```navigator.maxTouchPoints || navigator.msMaxTouchPoints``` and attempting to create a ```TouchEvent``` handler (if it's successful then touch is supported)
    * Browser vendor - ```navigator.vendor```
    * Browser product (should always be equal to "Gecko", even on Chrome based browsers) - ```navigator.product```
    * Browser build number - ```navigator.productSub```
    * Internet explorer detection through parsing ```navigator.appName```
    * Chrome detection (worth noting that this tends to be defined on most Chromium derivatives, including Microsoft Edge) - presence of ```window.chrome```
    * Webdriver support (the [W3C specification](https://www.w3.org/TR/webdriver/#example-1) declares that tools implementing the Webdriver browser automation standard should declare ```navigator.webdriver``` as true) - presence of ```navigator.webdriver```
    * Current URL protocol - ```document.location.protocol```
    * Session history length - ```window.history.length```
    * CPU core count - ```navigator.hardwareConcurrency```
    * Battery interface support - presence of ```navigator.getBattery```
    * Media device (webcams, microphones, speakers) enumeration - ```navigator.mediaDevices```
    * Cookie support - ```navigator.cookieEnabled```
    * Browser appName (always "Netscape" on modern browsers) - ```navigator.appName```
    * Screen Color Depth - ```screen.colorDepth```
    * String representation of setTimeout function (used to detect tampering, it should return ```function setTimeout() { [native code] }``` but if a wrapper is being used it will return something different) - ```setTimeout.toString()```
    * String representation of setInterval function (same reasoning as above) - ```setInterval.toString()```

### Proof of Work

After collecting all of this information, Distil then runs its "proof of work."  Similar to the Cloudflare Under Attack mode, your browser basically generates some sort of token using JavaScript that the server then checks and (hopefully) ensures it was generated with the JavaScript.  Distil's "proof of work" seems to just be generating random alphanumeric characters and adding the current date to it.

    function t(e) {
      for (var t = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz", r = "", n = 0; e > n; ++n) r += t.substr(Math.floor(Math.random() * t.length), 1);
      return r;
    }
    h("DistilProofOfWorkStart");
    var r = new o, n = (new Date).getTime() + ":" + t(20);
    r.mine(n, 8, function (t) {
      h("DistilProofOfWorkStop"), e({proof: t});
    });


### Browser Automation Testing
Following the proof of work, the automated/headless browser detection code is run.  This code is obfuscated.  The first thing I noticed was that there was a large array of strings represented with hexadecimal escaping.  This is a technique most commonly used by [obfuscator.io](https://obfuscator.io).  This makes it very hard to read -- ```_0x159[54]``` is much harder to read than "document."  To undo this obfuscation, I used Jarrod Overson's great [shift-refactor](https://github.com/jsoverson/shift-refactor) library.  It uses Shape Security's Shift AST to allow you to parse and manipulate JavaScript.  To replace all member expressions (i.e. arr[1]) with their string values, I used the following code:

    const { RefactorSession } = require('shift-refactor');
    const { parseScript } = require('shift-parser');
    const Shift = require('shift-ast');

    const fileContents = require('fs').readFileSync('./distillnetworks.js', 'utf8');

    const tree = parseScript(fileContents);

    const refactor = new RefactorSession(tree);

    refactor.replace(
      `ComputedMemberExpression[expression.type="LiteralNumericExpression"][object.type="IdentifierExpression"]`,

      node => {
        var referencedArrayName = node.object.name;
        var arrayQuery = `VariableDeclarator[binding.name="` + referencedArrayName + `"][init.type="ArrayExpression"]`;
        var referencedArrayDeclaration = refactor.query(arrayQuery)[0];
        if (!referencedArrayDeclaration) {
          return node;
        }
        if (referencedArrayDeclaration.init.elements[node.expression.value].type !== "LiteralStringExpression") { // if we're working with a non string, return
          return node;
        }
        var newValue = referencedArrayDeclaration.init.elements[node.expression.value].value;
        if (!newValue) {
          return node;
        }
        else {
          return new Shift.LiteralStringExpression({
            value: newValue,
          });      
        }
      }
    );
    console.log("/* Replaced all array references with real value */")

    console.log(refactor.print());

Essentially, we find all of the member expressions (```ComputedMemberExpression```) that index the array (the ```IdentifierExpression``` -- though this can refer to any variable) using an integer (```LiteralNumericExpression```).  We then find the array that is being indexed (searching for the ```VariableDeclarator``` of type ```ArrayExpression``` with the same name as the referenced variable), get the actual value, make sure it is a string, and then return it if so.  Otherwise, we return the original ```node```.  After running this (```node distillnetworks_deobfuscator.js > distillnetworks_deobfuscated.js```), readability has massively increased, but there is still much to be desired.

I then noticed one more annoyance.  There was a function, ```_0x9e50x4```, that, if passed "O" as an argument, would return ```["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"]```.  Otherwise, it would return ```[]```.  These strings themselves are not actually used in the code, but their substrings are actually used to assemble other strings.  To replace the ```_0x9e50x4``` calls with their real values, I added the following code above the ```refactor.print``` call.

    var _0x9e50x4List = ["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"];
    refactor.replaceRecursive(
      `CallExpression[callee.name="_0x9e50x4"][callee.type="IdentifierExpression"]`,

      node => {
        if (node.arguments.value="O") {
          var newArray = new Shift.ArrayExpression({elements: []});
          for (var i = 0; i < _0x9e50x4List.length; ++i) {
            newArray.elements.push(new Shift.LiteralStringExpression({value: _0x9e50x4List[i]}));
          }
          return newArray;
        }
        else {
          return new Shift.ArrayExpression({
            elements: [],
          })
        }
        return node;
      }
    );

Then, there was another thing that was hurting my eyes.  ```_0x9e50x1``` was being referenced instead of ```window```.  To increase readability, I added another ```refactor.replace```:

    var windowObjectName = "_0x9e50x1";
    refactor.replace(
      `IdentifierExpression[name="` + windowObjectName + `"]`,

      node => {
        return new Shift.IdentifierExpression({
          name: "window",
        })
      }
    );
    console.log("/* Done renaming " + windowObjectName + " with window");

The code is finally legible... somewhat.  After reading the deobfuscated version, I notice another array of strings

    var _0x9e50x19 = ["__" + _0x9e50xe + "_" + _0x9e50xd + "uate", "__web" + _0x9e50xe + "_" + _0x9e50xd + "uate", "__s" + _0x9e50xf + "_" + _0x9e50xd + "uate", "__fx" + _0x9e50xe + "_" + _0x9e50xd + "uate", "__" + _0x9e50xe + "_unwrapped", "__web" + _0x9e50xe + "_unwrapped", "__s" + _0x9e50xf + "_unwrapped", "__fx" + _0x9e50xe + "_unwrapped", "__web" + _0x9e50xe + "_script_" + _0x9e50x10 + "tion", "__web" + _0x9e50xe + "_" + "script" + "_" + _0x9e50x10, "__web" + _0x9e50xe + "_" + "script" + "_fn"];
    var _0x9e50x1a = ["_S" + _0x9e50xf + "_IDE" + "_Recorder", "_p" + _0x9e50xc, "_s" + _0x9e50xf, _0x9e50x11 + "P" + _0x9e50xc, _0x9e50x11 + "S" + _0x9e50xf, _0x9e50x19[+[]][+!![]] + "_" + _0x9e50x12 + "e"];

Strings are being assembled by adding them together.  I recognized that the ```"_IDE" + "_Recorder"``` string probably refers to the ```Selenium_IDE_Recorder```, which would imply that the variable ```_0x9e50xf``` is equal to "elenium".  Instead of doing guesswork though, I decided that I would just find the original definitions of these variables and replace their values.  I found them below the function for creating an XMLHttpRequest object.

    _0x9e50xc = _0x9e50x3[3]["substring"](["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"]["length"] - !![], ["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"]["length"] + !![]), _0x9e50xd = [] + _0x9e50x3["slice"](-!![]), _0x9e50xe = _0x9e50x3[8][2 + !![]] + _0x9e50x3[["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"]["length"]]["substring"](_0x9e50xd["length"] + ![]), _0x9e50xf = _0x9e50x3[_0x9e50xd["length"] + 1]["slice"](-2) + (_0x9e50x3["slice"](-1) + [])[+[]] + "n" + _0x9e50x3[+!![] + !![] + !![]]["substr"](-(+!![] + !![] + !![])), _0x9e50x12 = _0x9e50xf["substring"](_0x9e50xe["length"], +[] + 5), _0x9e50x11 = _0x9e50xd["substring"](!![] + !![]), _0x9e50x12 = _0x9e50x12 + (_0x174c[30] + window["navigator"])["substring"](_0x9e50x3["length"] - !![], _0x9e50x3["length"] + _0x9e50x11["length"]), _0x9e50x10 = (_0x9e50x3[!["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"] + 1][+![]] + _0x9e50xf[_0x9e50xe["length"] + _0x9e50xe["length"] - !![]] + _0x9e50xf[_0x9e50xe["length"]] + _0x9e50x3[_0x9e50xe["length"] - !![]][-![]])["toLowerCase"](), _0x9e50x12 = (_0x9e50x12 + _0x9e50xc[_0x9e50xc["length"] - !![]] + _0x9e50x11[1 - ["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"] - !![]])["replace"]("a", "h"), _0x9e50x11 = _0x9e50x10[_0x9e50x10["length"] - !![]] + _0x9e50x11 + _0x9e50x11[+!![]], _0x9e50xc = ["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"][+!![]]["substring"](_0x9e50xf["length"] + _0x9e50xd["length"] - !![], _0x9e50xf["length"] + _0x9e50xe["length"] * 2)["replace"](["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"][+!![]][+!![]], _0x174c[30]) + "t" + _0x9e50xc;
    _0x9e50xe = _0x9e50xe + (_0x9e50x3["slice"](-!!["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"]) + [])["substring"](-!["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"], ["Snow Leopard", "Lion/Mountain Lion", "Yosemite", "Mavericks"]["length"] - !![] - !![])["replace"](/(.)(.)/, "$2$1") + _0x9e50xe[+!![]], _0x9e50xc = "h" + _0x9e50xc, _0x9e50x12 = _0x9e50x12 + _0x9e50xe[+!![]];

This seemed to be very complicated, and given that there was only a few variables that needed replacing, I decided I would do it without the script.  I then ran this in my browser console and got an error that ```_0x174c``` and some other variables were undefined, so I copy pasted the definition from the original code.

    var _0x9e50x2 = "/dstl-wp.js?PID=FA07FD5E-619C-38C3-83F2-EB07F1B68C83";
    var _0x9e50x3 = ["Internet Explorer", "Firefox", "Chrome", "Chromium", "Safari", "MacIntel", "Win32", "Win64", "Windows", "WinNT", "OSX", "Linux", "eval"];
    var _0x174c = ["/dstl-wp.js?PID=FA07FD5E-619C-38C3-83F2-EB07F1B68C83", "\x49\x6E\x74\x65\x72\x6E\x65\x74\x20\x45\x78\x70\x6C\x6F\x72\x65\x72", "\x46\x69\x72\x65\x66\x6F\x78", "\x43\x68\x72\x6F\x6D\x65", "\x43\x68\x72\x6F\x6D\x69\x75\x6D", 
    ...

I then replaced there values in manually in the rest of the code.  The variables were mostly smaller part of larger strings (i.e. "hantom", "elenium", "etc").  My final version can be found at [https://pastebin.com/rp2Xu0Jt](https://pastebin.com/rp2Xu0Jt). 

This gave me:
    var _0x9e50x19 = ["__driver_evaluate", "__webdriver_evaluate", "__selenium_evaluate", "__fxdriver_evaluate", "__driver_unwrapped", "__webdriver_unwrapped", "__selenium_unwrapped", "__fxdriver_unwrapped", "__webdriver_script_function", "__webdriver_script_func", "__webdriver_script_fn"];
    var _0x9e50x1a = ["_Selenium_IDE_Recorder", "_phantom", "_selenium", "callPhantom", "callSelenium", "__nightmare"];

Now, we can finally understand the headless detection portion of the code.

The automated/headless browser detection code checks for the presence of:

    * Selenium - ```document.__driver_evaluate```
    * Selenium - ```document.__webdriver_evaluate```
    * Selenium - ```document.__selenium_evaluate```
    * Selenium - ```document.__fxdriver_evaluate```
    * Selenium - ```document.__driver_unwrapped```
    * Selenium - ```document.__webdriver_unwrapped```
    * Selenium - ```document.__selenium_unwrapped```
    * Selenium - ```document.__fxdriver_unwrapped```
    * Selenium - ```document.__webdriver_script_function```
    * Selenium - ```document.__webdriver_script_func```
    * Selenium - ```document.__webdriver_script_fn```
    * Selenium - ```window._Selenium_IDE_Recorder```
    * PhantomJS - ```window._phantom```
    * Selenium - ```window._selenium```
    * PhantomJS - ```window.callPhantom```
    * Selenium - ```window.callSelenium```
    * NightmareJS - ```window.__nightmare```
    * Regex to check for the presence $cdc and $wdc variables in document which are created by Selenium and other headless browsers.  See this [StackOverflow](https://stackoverflow.com/a/41220267) answer for more.
    * Sequentum (web scraping company) check - ```window["external"].toString()["indexOf"]("Sequentum") != -1)```
    * Selenium - ```window["document"]["documentElement"]["getAttribute"]("selenium")```
    * Selenium - ```window["document"]["documentElement"]["getAttribute"]("webdriver")```
    * Selenium - ```window["document"]["documentElement"]["getAttribute"]("driver")```


## Conclusion
The above effort was a few hours of work and further shows the ineffectiveness of "security through obscurity" (unless you're Google - I don't think anyone has figured out Botguard).  I also found that while the fingerprinting was pretty comprehensive, the bot detection was lack luster.  A cursory glance at any ad fraud/viewability vendor script will confirm this (CSS checks, browser specific API checks, and many more ideas are not implemented).

1.  An archived copy of the code can be found at [https://pastebin.com/y5iu5iih](https://pastebin.com/rp2Xu0Jt).  My final deobfuscated version can be found at [https://pastebin.com/rp2Xu0Jt](https://pastebin.com/rp2Xu0Jt).

---
layout: post
title: How Websites Know You're Lying About Your User-Agent
tags: [javascript, reverse engineering, fingerprinting]
---

Spoofing a browser's user agent is often hailed as a privacy enhancing technique.  On the Chrome Store, there are [dozens of extensions](https://chrome.google.com/webstore/search/user%20agent?_category=extensions) allowing you to switch your user agent.  Unfortunately, due to the abundance of methods to detect browser and operating system information (as will be discussed in this article), these extensions do not meaningfully enhance privacy.  There are some other reasons to change a browser's user agent, for example to test the mobile version of a website, or to bypass rate limits while scraping.  Bot detection and fingerprinting vendors like Distil Networks, Imperva, WhiteOps, etc are all getting smarter about detecting this kind of spoofing.  This post will reveal some of the techniques they use and illustrate the futility of many of these browser extensions.

## Preface: What is a user agent?

Wikipedia states:

>In computing, a user agent is any software, acting on behalf of a user, which "retrieves, renders and facilitates end-user interaction with Web content."

It is essentially your web browser.  Firefox is a user agent, Chrome is a user agent, Opera is a user agent.  Each (or most) user agents have a User-Agent string that reveals information about them.  This string is communicated via HTTP and can be accessed through Javascript as well.  For instance, when making a request to google.com, `curl` sends the following data:


    GET / HTTP/1.1
    Host: google.com
    User-Agent: curl/7.64.0
    Accept: */*

We can see the user agent (`User-Agent`) header has the value `curl/7.64.0`.  When visiting google.com on Chrome, we can see Chrome sends:

    GET / HTTP/1.1
    accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
    accept-encoding: gzip, deflate, br
    accept-language: en-US,en;q=0.9
    cache-control: max-age=0
    sec-ch-ua: " Not A;Brand";v="99", "Chromium";v="90"
    sec-ch-ua-mobile: ?0
    sec-fetch-dest: document
    sec-fetch-mode: navigate
    sec-fetch-site: same-origin
    sec-fetch-user: ?1
    upgrade-insecure-requests: 1
    user-agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.212 Safari/537.36

We can see that the Chrome user agent string reveals OS and browser engine information.  This string is important because it can be used by websites for platform detection, bot detection, etc.

### How User Agent Spoofing is Detected

This post will begin at the lower levels of the web technology stack and move up from there.  Thus, we begin with TCP.

#### TCP

HTTP uses TCP as the underlying transport protocol.  Given that there are variations in TCP implementations across operating systems and operating system versions, TCP can be used as a fingerprinting vector.  The [p0f](https://lcamtuf.coredump.cx/p0f3/) tool, released over 2 decades ago, was the first to do this.  For instance, look at this sample output from p0f

    .-[ 1.2.3.4/1524 -> 4.3.2.1/80 (syn) ]-
    |
    | client   = 1.2.3.4
    | os       = Windows XP
    | dist     = 8
    | params   = none
    | raw_sig  = 4:120+8:0:1452:65535,0:mss,nop,nop,sok:df,id+:0

p0f works by mapping signatures of TCP headers and flags to specific OS versions.  In practice, TCP fingerprinting means that it is possible to detect that you are running a scraping bot on a Linux server but are using a user agent indicating Chrome on Windows 10.

\[Note for proxy users: Your provider may have an option to select the operating system of devices your requests are routed through. See [Luminati's](https://brightdata.com/faq#os-proxy-peer) documentation.\]

#### HTTP

HTTP requests have headers associated with them.  For example, recall the above example of the HTTP request from Chrome.  It has the User-Agent header, but it also has headers revealing cache information, language information, and more.

Compare the headers from Chrome with the headers from Firefox:

    GET / HTTP/1.1
    Host: www.google.com
    User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate, br
    Connection: keep-alive
    Upgrade-Insecure-Requests: 1
    Pragma: no-cache
    Cache-Control: no-cache

Notice that Firefox is missing many headers Chrome has, mostly the ones beginning with `sec-`.  The values of other headers are different too, like `Accept`, and `Accept-Language`.  Furthermore, the order of headers is also different.  In Firefox, the `User-Agent` header appears at the top, whereas in Chrome it is at the bottom.  `Accept-Encoding` appears before `Accept-Language` in Chrome too.  This is another vector that can be used to detect spoofing.  Read more [here](http://geocar.sdf1.org/browser-verification.html).

#### JavaScript

There are many ways JavaScript can be used to detect browser spoofing.  Here are some common ones.  For the sake of brevity, I compare Firefox and Chrome, but the ideas contained in this section are applicable to any browser that supports JS.

##### CSS Queries

Browsers often have their own experimental/non-standard CSS features.  We can test the presence of these using `CSS.supports`.  Here are a few illustrative examples

Chrome:

    CSS.supports("-moz-user-focus", "normal") -> false
    CSS.supports("-moz-box-sizing", "content-box") -> false
    CSS.supports("-webkit-border-vertical-spacing", 0) -> true

Firefox:

    CSS.supports("-moz-user-focus", "normal") -> true
    CSS.supports("-moz-box-sizing", "content-box") -> true
    CSS.supports("-webkit-border-vertical-spacing", 0) -> false

Due to prefixed features, detection *across* engines is very easy.  Detection within browser engines but across versions can also be done by checking for new CSS attributes.

##### Math

Subtle variations in rounding can be used to detect browser engines:

Chrome:

    Math.hypot(-24.42, -50.519999999999925) -> 56.1124478168614

Firefox:

    Math.hypot(-24.42, -50.519999999999925) -> 56.11244781686139

There used to be other methods involving trigonometric functions with very large numbers, but after testing these on the latest versions of Chrome and Firefox, they seem to produce identical results.

##### Presence of Certain Window Attributes
Prefixed (`moz`, `webkit`) attributes are often useful for distinguishing engines in this aree, but there are others like `window.chrome` too.

Chrome:

    window.webkitCancelAnimationFrame !== undefined -> true
    window.mozInnerScreenX !== undefined -> false
    window.PERSISTENT !== undefined -> true
    window.chrome !== undefined -> true
    window.InstallTrigger !== undefined -> false

Firefox:

    window.webkitCancelAnimationFrame !== undefined -> false
    window.mozInnerScreenX !== undefined -> true
    window.PERSISTENT !== undefined -> false
    window.chrome !== undefined -> false
    window.InstallTrigger !== undefined -> true

##### Navigator Object
`window.navigator` provides information about the operating system and browser of the client.  Most spoofing extensions update the values here but many do not.  `navigator.userAgent` should equal the header sent over HTTP, for instance.  If your browser purports to be Chrome, `navigator.vendor` should equal "Google Inc."  Furthermore, plugins across browsers often vary.  Chrome always has the Chrome PDF viewer installed whereas Firefox often does not have any plugins at all (making `navigator.plugins.length` equal 0).  There are other details like [`navigator.productSub`](https://stackoverflow.com/questions/13880858/why-does-navigator-productsub-always-equals-20030107-on-chrome-and-safari) that also may be of interest.  The architecture and OS returned by `navigator.platform` should also match the value in the User-Agent string.

##### Others
Here are some additional miscellaneous techniques I've learned from reading browser fingerprinting scripts.  Another one that comes to mind is checking that the resolution of the device makes sense (using `screen.height` and `screen.width`), i.e. an iPhone is not going to have a 4096x2160 resolution.

Chrome:
    
    // eval is a "native code" function so its string representation looks something like "function eval() { [native code] }".  chrome and firefox format this slightly differently, with chrome keeping it all on one line and firefox putting it on multiple lines, which makes firefox's eval.toString().length longer.

    eval.toString().length -> 33

    navigator.oscpu !== undefined -> false

    // only true on old versions of Firefox
    ({}.toSource) !== undefined -> false

    // error messages - chrome gives "Cannot read property '0' of undefined," firefox "undefined has no properties"
    try {
        undefined[0];
    } catch (e) {
        console.log(e.toString().indexOf("read property") !== -1); // true
    }

    // properties of errors - firefox supports several non standard error attributes like lineNumber
    try {
        undefined[0];
    } catch (e) {
        console.log(e.lineNumber !== undefined); // false
    }

Firefox:

    // eval is a "native code" function so its tostring looks something like "function eval() { [native code] }".  chrome and firefox format the tostring differently, with chrome keeping it all on one line and firefox putting it on multiple lines, which makes firefox's eval.toString().length longer.

    eval.toString().length -> 37

    navigator.oscpu !== undefined -> true

    // only true on old versions of Firefox
    ({}.toSource) !== undefined -> true

    // error messages - chrome gives "Cannot read property '0' of undefined," firefox "undefined has no properties"
    try {
        undefined[0];
    } catch (e) {
        console.log(e.toString().indexOf("read property") !== -1); // false
    }

    // properties of errors - firefox supports several non standard error attributes like lineNumber
    try {
        undefined[0];
    } catch (e) {
        console.log(e.lineNumber !== undefined); // true
    }

## Conclusion

Spoofing your user-agent may allow you to use the desktop version of Google Drive on your phone, but will not provide meaningful privacy against even the most mildly sophisticated actors.  Furthermore, there are several pitfalls one must be aware of when spoofing reported browser via User-Agent.  Pretending to be an older version of the same browser when scraping may pass some antibot checks, but spoofing a different engine (Chrome [Webkit/V8] -> Firefox [Gecko] etc) is a Sisyphean task.

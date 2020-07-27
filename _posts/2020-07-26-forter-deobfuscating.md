---
layout: post
title: Deobfuscating Forter's Fraud Prevention JavaScript
tags: [javascript, reverse engineering]
---
Forter is a fraud prevention vendor for ecommerce sites.  They collect a large number of signals in JavaScript to decide whether to allow a transaction or not.  I hacked together a script [1] again using Jarrod Overson's great [shift-refactor](https://github.com/jsoverson/shift-refactor) library.  A sample of their code can be found at [2].  This undoes their annoying string obfuscation (setting a bunch of strings at the top of the file and then using them later, combining a bunch of strings, and some XOR based things).  

## Analysis
Now that we have the deobfuscated (mostly) code [3], we can go over the indicators used in the code.

    * Shopify event monitoring - window.Shopify
    * User Agent - navigator.userAgent
    * Screen width/height - screen.width / screen.height
    * Available screen width/height - screen.availWidth / screen.availHeight
    * Screen orientation - window.orientation || window.mozOrientation
    * Device Pixel Ratio - window.devicePixelRatio
    * First available pixel available from the left side of the screen - screen.availLeft
    * Distance from the top side of the current screen - screen.top
    * Y-coordinate of the first pixel that is not allocated to permanent or semipermanent user interface features - screen.availTop
    * Screen color depth - screen.colorDepth
    * Screen pixel depth - screen.pixelDepth
    * Screen buffer depth (IE only) - screen.bufferDepth
    * Font smoothing status (IE only) - screen.fontSmoothingEnabled
    * Installed plugin count - navigator.plugins.length
    * Monitoring events for orientation changes - orientationchange event
    * Resize window events - resize event
    * System platform - navigator.platform
    * Device pixel ratio changes - setTimeout monitoring
    * Mouse movement analysis
        * Slow movement
        * Jitter
        * Total events seen
        * Total distance covered
        * Global distance ratio
        * Average movement time
        * Drop count
        * Indicators for "good" vs "bad"
    * Device frames per second (FPS) - requestAnimationFrame
    * Performance timing
        * connectEnd
        * secureConnectionStart
        * responseStart
        * domainLookupStart
        * domainLookupEnd
    * Creating a websocket connection
    * NetworkInformation API - navigator.connection
        * RTT
        * Downlink speed
        * Downlink max speed
        * Connection type change event listener
    * Headless detection
        * The presence of:
            * Selenium - document.$cdc_asdjflasutopfhvcZLmcfl_
            * Selenium - document.$chrome_asyncScriptInfo
            * Selenium - document.$wdc_
            * Selenium - navigator.webdriver
            * Selenium - document["documentElement"]["getAttribute"]("webdriver")
            * NightmareJS - window.__nightmare
            * Selenium IDE Recorder - window._Selenium_IDE_Recorder
            * Headless Chrome - window.domAutomation
            * Headless Chrome - window.domAutomationController
            * PhantomJS - window.callPhantom
            * PhantomJS - window._phantom
            * PhantomJS - window.__phantomas
            * NodeJS - window.Buffer
            * CouchJS - window.emit
            * Rhino - window.spawn
        * iMacros - Creating an element with the ID "imacros-highlight-div" and getting its colors
        * Querying permissions for MIDI and background-sync and looking for inconsistencies in permissions vs permissions.query (notifications are typically used for this - see https://antoinevastel.com/bot%20detection/2018/01/17/detect-chrome-headless-v2.html)
        * navigator.languages length
        * window["innerWidth"] === 800 && window["innerHeight"] === 600
    * Canvas
        * Max texture anisotropy - MAX_TEXTURE_MAX_ANISOTROPY_EXT
        * GPU Info - UNMASKED_RENDERER_WEBGL / UNMASKED_VENDOR_WEBGL
        * Max draw buffers - MAX_DRAW_BUFFERS_WEBGL
        * Shader precision formats - getShaderPrecisionFormat
        * Rendering context support for ["copyBufferSubData", "getBufferSubData", "blitFramebuffer", "framebufferTextureLayer", "getInternalformatParameter", "invalidateFramebuffer", "invalidateSubFramebuffer", "readBuffer", "renderbufferStorageMultisample", "texStorage2D", "texStorage3D", "texImage3D", "texSubImage3D", "copyTexSubImage3D", "compressedTexImage3D", "compressedTexSubImage3D", "getFragDataLocation", "uniform1ui", "uniform2ui", "uniform3ui", "uniform4ui", "uniform1uiv", "uniform2uiv", "uniform3uiv", "uniform4uiv", "uniformMatrix2x3fv", "uniformMatrix3x2fv", "uniformMatrix2x4fv", "uniformMatrix4x2fv", "uniformMatrix3x4fv", "uniformMatrix4x3fv", "vertexAttribI4i", "vertexAttribI4iv", "vertexAttribI4ui", "vertexAttribI4uiv", "vertexAttribIPointer", "vertexAttribDivisor", "drawArraysInstanced", "drawElementsInstanced", "drawRangeElements", "drawBuffers", "clearBufferiv", "clearBufferuiv", "clearBufferfv", "clearBufferfi", "createQuery", "deleteQuery", "isQuery", "beginQuery", "endQuery", "getQuery", "getQueryParameter", "createSampler", "deleteSampler", "isSampler", "bindSampler", "samplerParameteri", "samplerParameterf", "getSamplerParameter", "fenceSync", "isSync", "deleteSync", "clientWaitSync", "waitSync", "getSyncParameter", "createTransformFeedback", "deleteTransformFeedback", "isTransformFeedback", "bindTransformFeedback", "beginTransformFeedback", "endTransformFeedback", "transformFeedbackVaryings", "getTransformFeedbackVarying", "pauseTransformFeedback", "resumeTransformFeedback", "bindBufferBase", "bindBufferRange", "getIndexedParameter", "getUniformIndices", "getActiveUniforms", "getUniformBlockIndex", "getActiveUniformBlockParameter", "getActiveUniformBlockName", "uniformBlockBinding", "createVertexArray", "deleteVertexArray", "isVertexArray", "bindVertexArray"]
    * Workers support
    * CPU cores - navigator.hardwareConcurrency
    * Incognito detection
        * Chrome - navigator.storage.estimate < 12e7 (see https://mishravikas.com/articles/2019-07/bypassing-anti-incognito-detection-google-chrome.html)
        * Firefox (disabled in Incognito) - indexedDB/mozIndexedDB support
    * Memory performance indicators
        * window.performance.memory
            * JS Heap Size Limit - jsHeapSizeLimit
            * Total JS Heap Size - totalJSHeapSize
            * Used JS Heap Size - usedJSHeapSize
    * Media devices enumeration (and counts by type) - navigator.mediaDevices.enumerateDevices
    * Speech synthesis voices - speechSynthesis.getVoices();
    * Opera browser detection - window.opera
    * Yandex browser detection - window.yandex
    * Maxthon detection - window.mxZoomFactor
    * Silk browser detection - window.com_amazon_cloud9_immersion
    * Firefox mobile detection - Firefox in UserAgent but no NetworkInformation support
    * Firefox detection - typeof(InstallTrigger) !== "undefined"
    * IE detection - document.documentMode support
    * Brave detection - navigator.brave && typeof navigator.brave.isBrave === "function"
    * Safari detection
        navigator["getStorageUpdates"] && navigator["getStorageUpdates"]["toString"] && navigator["getStorageUpdates"]["toString"]()["indexOf"]("native") > 0 || l1I["test"](window["HTMLElement"]) || function (e1J) {
              return e1J && e1J["toString"]() === "[object SafariRemoteNotification]";
    * Mobile Safari detection - same as regular Safari detection but a TouchEvent can be created
    * Safari standalone status - navigator.standalone
    * IE version detection - IE templates (https://stackoverflow.com/questions/3958913/fix-css-if-lt-ie-8-in-ie)
          u1J = document["createElement"]("div");
          u1J["innerHTML"] = "<!--[if lt IE 8]><i></i><![endif]-->";
          Q1J["ieVerLessThan8"] = document["getElementsByTagName"]("i")["length"] === 1;
    * Touchscreen max touch points - navigator.maxTouchPoints
    * Browser engine detection - Creating a new element, iterating over its style attribute, and checking for prefixed CSS attributes (moz, webkit, ms)
    * STUN Local/Remote IP Address - RTCPeerConnection to 52.23.111.175:3478 and stun:ec2-52-23-111-175.compute-1.amazonaws.com:3478

1. Available at https://pastebin.com/feCCWqMW.
2. Originally available at https://cdn4.forter.com/script.js?sn=6bc22bdd02fc.  The version my deobfuscation script was tested on can be found at https://pastebin.com/ALqJxrvM.
3. Available at https://pastebin.com/QJxHYz0z.

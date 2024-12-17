## Why
My friends and I play [GeoGuessr](https://www.geoguessr.com/) every Wednesday. It's a small yet awesome tradition of ours, a reason to get together on Discord each week. Being a naturally curious person, I had a lot of questions to ask. What is our top score? How often do we mess up UK with New Zealand? Do we ever get Monaco wrong?
Imagine my disappointment, when I realized that GeoGuessr provides neither such stats, nor any kind of records of our games. So I had to collect this data myself.
Since I already had some experience [intercepting requests](https://demaga.github.io/jekyll/update/2024/05/29/crusade-against-youtube-shorts.html) with browser extensions, I figured this was the way to go. There was only one catch - GeoGuessr uses **WebSockets** to synchronize game state between players. But it shouldn't be much harder to intercept WebSocket messages rather than HTTP requests, right? **WRONG**!

## Browser APIs
If you look at the "webRequest" section of [JavaScript APIs for WebExtensions](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API), you'll see the following statement:

> "webRequest" - Add event listeners for the various stages of making an HTTP request, which **includes websocket requests** on `ws://` and `wss://`.

But don't get your hopes too high. When the browser wants to open a WebSocket connection, it has to communicate this desire to server somehow. It is done in the simplest way possible, with [HTTP 101 Switching Protocols](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/101) request. And you make this request to the same endpoint that would later be used to send WebSocket messages. So **this** is the request they're referring to:

``` json
{
  "requestId": "29739",
  "url": "wss://game-server.geoguessr.com/ws",
  "originUrl": "https://www.geoguessr.com/live-challenge/cd268fa6-5c92-4e9f-89ff-88c20c927722",
  "documentUrl": "https://www.geoguessr.com/live-challenge/cd268fa6-5c92-4e9f-89ff-88c20c927722",
  "method": "GET",
  "type": "websocket",
  "timeStamp": 1733773764879,
  "tabId": 68,
  "frameId": 0,
  "parentFrameId": -1,
  "incognito": false,
  "thirdParty": false,
  "cookieStoreId": "firefox-default",
  "proxyInfo": null,
  "ip": null,
  "frameAncestors": [],
  "urlClassification": {
    "firstParty": [],
    "thirdParty": []
  },
  "requestSize": 0,
  "responseSize": 0
}
```

It even has type "websocket"! I mean, it's not wrong, just not quite what I've been looking for. No other "websocket" type requests are being sent by the way.

Okay, so maybe there is another API for WebSockets? Well, yes, there is! Both Chrome and Firefox implement ["The WebSocket API"](The **WebSocket API**), but it is nothing like webRequest one. There are no events you can listen to, no way to list existing connections, nothing. It's only good for creating new connections.

So, how do you do it then?
## Hacky way

The are a couple of ways to approach this. First is to monkey-patch "WebSocket" object in your browser, so that you can execute any custom code before proceeding with an event.

It looks like this:
``` js
WebSocket.prototype.oldSend = WebSocket.prototype.send;

WebSocket.prototype.send = function (data) {
	console.log("ws: sending data", data);
	WebSocket.prototype.oldSend.apply(this, [data]);
};

var oldAddEventListener = WebSocket.prototype.addEventListener;
WebSocket.prototype.addEventListener = function (eventName, callback) {
	return oldAddEventListener.apply(this, [eventName,
		(e) => {
			if (e.type == "message")
				console.log("ws: receiving data", e.data);
			callback(e)
		}
	])
};
```

I made [a gist](https://gist.github.com/Demaga/1c4a4ef634204156667503eacda529a9) with full working example.
It's pretty concise, and, surprisingly, it works in both Chrome and Firefox (manifest v3)! There are only two issues with this solution:
- you're monkey-patching WebSocket object, so if you mess up, you might break all functionality that relies on WebSocket
- it's executed in content script, while you might want to access some features available only in service workers; in this case, send a message to service worker with [browser.runtime.sendMessage](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/runtime/sendMessage)
- it must be executed in "MAIN" world (so that you modify same WebSocket object that is used by JS on page), while you might want to communicate with other parts of your extension; in this case, send a message with [window.postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)

To make it work in my case, I had to send message to "ISOLATED" world first, then send another message to service worker.

## Even more hacky way

Believe it or not, but there is an even uglier solution. Chrome implements [debugger API](https://developer.chrome.com/docs/extensions/reference/api/debugger#concepts_and_usage), which lets you attach a debugger to a tab (or tabs), listen to debugger events there, and send Chrome DevTools Protocol (CDP) commands. It's a very powerful API. A lot of things that are possible to do in your DevTools are also possible to do with this API. Of course, certain features were not exposed due to security reasons. But it's still more than powerful enough for our use case. 

All we need to do is to attach debugger to our tab, listen to all events, and react once WebSocket event is fired. 
 ``` js
// background script (service worker)

// attach debugger when page is loaded
chrome.tabs.onUpdated.addListener(function (tabId, changeInfo, tab) {
    if (changeInfo.status === 'complete' && tab.url && tab.url.startsWith('http')) {
        attachDebuggerAndEnableNetwork(tabId);
    }
});

// listen to debugger events, react when we receive WebSocket message 
chrome.debugger.onEvent.addListener(function (source, method, params) {
    if (method == "Network.webSocketFrameReceived") {
        data = JSON.parse(params["response"]["payloadData"]);
        console.log("ws: receiving data", data);
    }
});
```

There are a couple of issues with this solution:
- impossible to make it work in Firefox, as it has no such API
- requires powerful "debugger" permission
- whenever the extension is run, an ugly "{extension_name} has started debugging this browser" bar appears on top of the page

## Let's talk "worlds"
I was quite surprised when I learned that there are two types of content scripts: ones that live in "MAIN" world, and the ones that live in "ISOLATED" world. It was crucial for my extension, because when I monkey-patched "WebSocket" object the first time, it was not the same "WebSocket" object that is being used by GeoGuessr page JS code. So it made no difference. Only when I made a separate content script that lives in "MAIN" world, I finally modified proper "WebSocket" object.
But with great power comes great responsibility. Therefore, I lost my ability to communicate with background scripts. I thought that was it, finita la commedia. Thankfully, I was wrong once again.
It turns out, content scripts in "ISOLATED" world are not that isolated. They share the same DOM, same "window" object with scripts in "MAIN" world. Thanks to "window.postMessage" method, those scripts can communicate!

This whole situation reminded me how background scripts themselves are separated from content scripts. They live in their own "background page" context, so they're basically on another "tab".
For example, when you save data to IndexedDB in content script (regadless of "world" it lives in), data is saved into "page" IndexedDB. And when you save data in background script, it is saved into "extension" IndexedDB. Thankfully, content scripts can communicate with background scripts as well via "browser.runtime.sendMessage"! 

It was a bit hard for me to wrap my head around this concepts, so I made a simple visualization of "worlds".
![visualization of "MAIN" and "ISOLATED" wolrds](/assets/worlds.png)

## GeoGuessr Stats
If you're interested in trying out this extension yourself, it is already available in both [Chrome](https://chromewebstore.google.com/detail/geoguessr-stats/epjjmfojjmbgignfnkhgnbhkjdanmlfp) and [Firefox](https://addons.mozilla.org/en-US/firefox/addon/geoguessr-stats/).

In extension tab you can only see top scores and filter for games on a particular map or with particular restrictions. However, the whole point of this endeavor was to get some deeper insights. So I exported data into JSON to analyze it later.

Guess entry looks like this:
``` json
{
	"roundNumber": 4,
	"lat": 23.48790806385134,
	"lng": 120.65365041800166,
	"distance": 111899.3277221051,
	"time": 40,
	"score": 4639,
	"wasCorrect": false,
	"gameId": "43b82fcf-29d8-4daf-84e9-80e029301229",
	"playerName": "Demaga Chill"
}
```
So the main obstacle would be reverse geocoding lat/lng pairs into country names. [GeoPandas](https://geopandas.org/en/stable/) is great for this kind of job, but I've been playing around with Polars lately and really wanted to use it here. [GeoPolars](https://geopolars.org/latest/) does exist, but I've been put off by project being halted for months. Developers of GeoPolars say that they're blocked by Polars supporting Arrow extension types. So until this issue is resolved, state of GeoPolars is unclear.

Thankfully, I am not the only one who came across this issue. [Marco Gorelli](https://github.com/MarcoGorelli) proposes a nice and clean solution based on Polars plugins and Rust's [reverse_geocoder](https://docs.rs/reverse_geocoder/latest/reverse_geocoder/), which surprisingly is a part of standard library. This small Polars plugin ["polars-reverse-geocode"](https://github.com/MarcoGorelli/polars-reverse-geocode/tree/main) is fast, accurate, and requires no additional dependencies. So I used it to reverse geocode coordinates for both guesses and answers. After that, I could finally get to analyzing this data.

Most common mistakes:

| country_code_answer | total | guessed | not_guessed | ratio    |
| ------------------- | ----- | ------- | ----------- | -------- |
| "AR"                | 78    | 13      | 65          | 0.166667 |
| "FI"                | 65    | 12      | 53          | 0.184615 |
| "ZA"                | 64    | 10      | 54          | 0.15625  |
| "BE"                | 61    | 10      | 51          | 0.163934 |
| "CL"                | 59    | 7       | 52          | 0.118644 |
| "UY"                | 58    | 9       | 49          | 0.155172 |
| "BM"                | 54    | 8       | 46          | 0.148148 |
| "PR"                | 51    | 7       | 44          | 0.137255 |
| "GT"                | 50    | 7       | 43          | 0.14     |
| "SK"                | 41    | 8       | 33          | 0.195122 |

Probably the most surprising thing here is how often we get spawned in Argentina! Unfortunately, we rarely get it right. I am also quite shocked with how rarely we get Belgium right! C'mon guys, it should've been easy.

Most common confusions:

| guess_answer | count |
| ------------ | ----- |
| "CA_US"      | 34    |
| "AT_DE"      | 31    |
| "GB_IE"      | 23    |
| "SZ_ZA"      | 20    |
| "ID_MY"      | 19    |
| "AU_US"      | 16    |
| "EE_FI"      | 16    |
| "EE_LT"      | 16    |
| "AU_NZ"      | 16    |
| "AU_ZA"      | 15    |

This is not surprising at all. If you get spawned in a location without many clues, it will be hard to distinguish Canada from US, or Austria from Germany, or Estonia from Finland.

Most successful guesses:


| country_code_answer | total | guessed | not_guessed | ratio    |
| ------------------- | ----- | ------- | ----------- | -------- |
| "KR"                | 30    | 26      | 4           | 0.866667 |
| "UA"                | 35    | 28      | 7           | 0.8      |
| "GR"                | 30    | 23      | 7           | 0.766667 |
| "NL"                | 62    | 43      | 19          | 0.693548 |
| "JP"                | 42    | 27      | 15          | 0.642857 |
| "IN"                | 28    | 18      | 10          | 0.642857 |
| "US"                | 88    | 55      | 33          | 0.625    |
| "TR"                | 45    | 28      | 17          | 0.622222 |
| "GB"                | 63    | 38      | 25          | 0.603175 |
| "IL"                | 32    | 19      | 13          | 0.59375  |


We guess Korea better than we guess our home country, wow! I guess it makes sense, as Korea is highly urbanized, and Korean language is easily recognizable. Same point for language goes to Greece.

I guess that's a good starting point. Of course, much more interesting insights can be extracted with some thought.

If you want, you can use [these Jupter Notebooks](https://github.com/Demaga/geoguessr-stats-analysis) to analyze your games too.

# Conclusions
1. Intercepting WebSocket messages in browser is not as easy as I thought, but absolutely possible.
2. Reverse geocoding in Polars is unexpectedly fast and accurate.
3. My friends and I are really good at guessing Korea in GeoGuessr.

# Links
1. [GeoGuessr Stats for Chrome](https://chromewebstore.google.com/detail/geoguessr-stats/epjjmfojjmbgignfnkhgnbhkjdanmlfp?authuser=2&hl=en)
2. [GeoGuessr Stats for Firefox](https://addons.mozilla.org/en-US/firefox/addon/geoguessr-stats/)
3. [GeoGuessr Stats source code](https://github.com/Demaga/geoguessr-stats)
4. [Jupyter Notebooks for games data analysis source code](https://github.com/Demaga/geoguessr-stats-analysis)

# Sources
1. [(SO) Override WebSocket onmessage event](https://stackoverflow.com/questions/67436474/how-to-override-websocket-onmessage-event)
2. [(SO) Manifest V3 "world" context](https://stackoverflow.com/questions/9515704/access-variables-and-functions-defined-in-page-context-from-an-extension/9517879#9517879) 
layout: page
title: "Lean YouTube"
permalink: /lean-youtube

# My struggle against YouTube Shorts
I love YouTube. But also, I hate YouTube.

![[Pasted image 20240527112425.png]](Ducks good... Rabbits bad)

The story so far: In the beginning the TikTok was created. This has made a lot of people very angry and been _widely regarded as a bad move_. However, it became so popular among younger generations, that all big tech media companies decided to ride the wave. That was the beginning of process known as TikTokification. That's why your Instagram feed now consists of random Reels instead of your friends' vacation photos and that's why YouTube now has Shorts.

Thanks to some kind individuals, there are several browser extensions that let you disable Shorts sections from YouTube if it bothers you too much. However, YouTube had "Shorts" long before they were a thing. I'm talking about those random 31 second 14 year old memes that randomly popped up in your recommendations. They were great! You see one, you click, you laugh, you move on with your day. 

Now, situation is different. YouTube *knows* that people like short-form content. YouTube *wants* to push it. And that is why after watching a couple of those videos, half of my recommendations now consist of sub-minute "non-shorts".

## Time to unhook
That's when I decided to take action. The most obvious^[Another solution is to touch grass.] solution would be to filter videos by duration. YouTube doesn't have such feature, but I figured there must be some extension for it, right? Well, maybe. But after searching for 10 minutes and not finding a working solution, I decided to do what every self-respecting programmer does - I decided to write it myself!

My initial idea was detecting video div's with content scripts and then deleting them if length is less than a minute. But it turns out that deleting a div element won't re-render grid layout, leaving ugly empty space on the screen. 
![[Pasted image 20240529011705.png]]

So that's no good. With help of browser developer tools I figured that video list is loaded through requests to one particular API endpoint. Then I read a little bit more about browser extensions and realized something.

## Intercepting requests is cooler than modifying content
There is a great thing called "background scripts"^[https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Background_scripts] which opens up a whole world of possibilities. Basically, it lets you control most of you browser's behavior. Requests, tabs, storage - it's all there!

So, in order to intercept requests (and modify responses!) in Firefox, you need a couple of things.
### 1. Define `manifest.json` file
It should contain something like this:
``` json
{
	"manifest_version": 3,
	"name": "My Extension",
	"version": "1",
	"background": {
		"scripts": [
			"background.js"
		],
	},
	"permissions": [
		"webRequest",
		"webRequestBlocking",
		"webRequestFilterResponse",
	]
}
```

In case if you don't know what `manifest.json` file is: don't worry, I got you! Browser doesn't know anything about your extension project structure. All it knows is that you must have a `manifest.json` file. This file is basically just a config with information about which scripts browser should run. Let's break down what's happening here.
- "manifest_version": version of this config file "language" 
- "version": version of your extension
- "name": name of your extension
- "description": description of your extension
- "background": list of scripts to run. Note that they should be defined as path relative to manifest
- "permissions": list of WebExtension APIs your extension needs. It should be as short as possible. In our case, we only need APIs related to requests

### 2. Create background script
First, we need to add a listener that will listen to new requests firing and triggering our function that will modify them.
``` js
browser.webRequest.onBeforeRequest.addListener(
	listener,
	{
		urls: ["https://www.youtube.com/youtubei/v1/browse?*"]
	},
	["blocking"],
);
```

- "browser": global object that is always available in background scripts
- "webRequest": name of WebExtension API we're using
- "onBeforeRequest": basically the first event from a series of events that are fired on every request. See diagram below and doc page [here](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest)
- "listener": our function that will be called when onBeforeRequest event fires

![[Pasted image 20240529014008.png]]

### 3. Modify response
Now we need to define listener function.
```js
function listener(details) {
    let filter = browser.webRequest.filterResponseData(details.requestId);
    
    const data = [];
    filter.ondata = (event) => {
        data.push(event.data);
    };
}
```
- "filter": object that we'll use to reference response data
- "filter.ondata": function that will be called when we start receiving response. Here it is defined as arrow function. Note that we don't get full response^[Now that I've written this I realize that I should have probably used onCompleted event and avoid unnecessary complications] as this API is a little lower level, so we store it in "data" array

Okay, so now we need to detect when youtube stops sending data and get to "modifying response part".
``` js
function listener(details) {
	...
    
    let decoder = new TextDecoder("utf-8");
    let encoder = new TextEncoder();
    filter.onstop = (event) => {
        let str = "";
        if (data.length === 1) {
            str = decoder.decode(data[0]);
        } else {
            for (let i = 0; i < data.length; i++) {
                const stream = i !== data.length - 1;
                str += decoder.decode(data[i], { stream });
            }
        }
        
        var obj = JSON.parse(str);
        var videos = obj["contents"]["twoColumnBrowseResultsRenderer"]["tabs"][0]["tabRenderer"]["content"]["richGridRenderer"]["contents"];
        
        videos = videos.filter((vid) => {
            let min_duration = 61;
            let max_duration = 3601;
            let total_seconds = 0;
            let seconds = 0;
            let minutes = 0;
            let hours = 0;
            let renderer = vid["richItemRenderer"]["content"]["videoRenderer"];
            let time = renderer["lengthText"]["simpleText"];
            time = time.trim();
            time = time.split(":");
            if (time.length == 3) {
                seconds = parseInt(time[2]);
                minutes = parseInt(time[1]);
                hours = parseInt(time[0]);
            } else {
                seconds = parseInt(time[1]);
                minutes = parseInt(time[0]);
            }
            total_seconds = seconds + minutes * 60 + hours * 3600;
            return total_seconds >= min_duration && total_seconds <= max_duration;
        });
        obj["contents"]["twoColumnBrowseResultsRenderer"]["tabs"][0]["tabRenderer"]["content"]["richGridRenderer"]["contents"] = videos;
        str = JSON.stringify(obj);
        filter.write(encoder.encode(str));
        filter.disconnect();
    };

    return {};
}
```
- "filter.onstop": function that will be called when we stop receiving data in response. Here it is defined as arrow function
- "filter.write()":  method that we call to actually return something. Without calling it explicitly, nothing will be returned to browser
- "filter.disconnect()": method that we call to disconnect the stream, as the default behavior is to keep the request open without a response
You can read more about filterResponseData object and it's behaviour [here](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/filterResponseData).

Once we receive full response, we need to decode it, find videos we want to delete, and delete them!

## Conclusions
Intercepting requests with Firefox extensions is simple yet awesome! It solved my problem, and if you encounter similar issues, please consider installing my Firefox Extension "Lean YouTube":  __MISSING_LINK__ and star its [Github project](https://github.com/Demaga/lean-youtube)
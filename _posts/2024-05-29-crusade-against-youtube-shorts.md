I love YouTube. But also, I hate YouTube.

![Ducks good... Rabbits bad](/assets/duck-rabbit.jpg)

The story so far: In the beginning the TikTok was created. This has made a lot of people very angry and been _widely regarded as a bad move_. However, it became so popular among younger generations, that all big tech media companies decided to ride the wave. That was the beginning of process known as TikTokification. That's why your Instagram feed now consists of random Reels instead of your friends' vacation photos and that's why YouTube now has Shorts.

Thanks to some kind individuals, there are several browser extensions that let you disable Shorts sections from YouTube if it bothers you too much. However, YouTube had "Shorts" long before they were a thing. I'm talking about those random 14-year-old 31-seconds-long memes that pop up in your recommendations every now and then.

# Time to unhook
One day I saw that my entire YouTube homepage was occupied by Shorts and "shorts" (regular short videos). That's when I decided to take action. The most obvious solution would be to filter videos by duration. YouTube doesn't have such feature, but I figured there must be some extension for it, right? Well, maybe. But after searching for 10 minutes and not finding a working solution, I decided to do what every self-respecting programmer does - I decided to write it myself!

My initial idea was detecting video div's with content scripts and then deleting them if length is less than a minute. But it turns out that deleting a div element won't re-render grid layout, leaving ugly empty space on the screen. 

![Ugly empty space](/assets/ugly-empty-space.png)

So that's no good. With help of browser's developer tools I figured that video list is loaded through requests to one particular API endpoint. So I had another epiphany.

# Intercepting requests is fun
There is a great thing called "background scripts" ([docs link](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Background_scripts)) which opens up a whole world of possibilities. Basically, it lets you control most of you browser's behavior. Requests, tabs, storage - it's all there!

So, in order to intercept requests (and modify responses!) in Firefox, you need a couple of things.
### 1. Define `manifest.json` file
It should contain something like this:
```json
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

Browser doesn't know anything about your extension project structure. All it knows is that you must have a `manifest.json` file. This file is basically just a config with information about which scripts browser should run. Let's break down what's happening here.
- "manifest_version": version of this config file "language" 
- "version": version of your extension
- "name": name of your extension
- "description": description of your extension
- "background": list of scripts to run. Note that they should be defined as path relative to manifest
- "permissions": list of WebExtension APIs your extension needs. It should be as short as possible. In our case, we only need APIs related to requests

### 2. Create background script
First, we need to add a listener that will *listen* to new requests firing and triger our function that will modify them.

``` js
browser.webRequest.onBeforeRequest.addListener(
   listener,
   {
        urls: [
            "https://www.youtube.com/youtubei/v1/browse?*"
        ]
   },
   ["blocking"],
);
```

- **browser**: global object available in background scripts
- **webRequest**: name of WebExtension API we're using
- **onBeforeRequest**: the first event from a series of events that are fired on every request. See diagram below and [doc](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest)
- **listener:** our function that will be called when onBeforeRequest event fires

![webRequests diagram](/assets/webrequests-diagram.png)

### 3. Modify response
"Listener" function will run when "onBeforeRequest" event fires. Let's look what objects we can work with.

``` js
function listener(details) {
    let filter = browser.webRequest.filterResponseData(details.requestId);
    
    const data = [];
    filter.ondata = (event) => {
        data.push(event.data);
    };
}
```
- **filter**: object that we'll use to reference response data
- **filter.ondata**: function that will be called when we recieve part of the response, as any response is sent in batches

Okay, so now we need to detect when youtube stops sending data and get to "modifying response" part.
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
        
        ...

        modify obj variable, it is just a JS object now
        
        ...

        let output_str = JSON.stringify(obj);
        filter.write(encoder.encode(output_str));
        filter.disconnect();
    };

    return {};
}
```
- "filter.onstop": (here - an arrow) function that will be called when we finish receiving data in response
- "filter.write()":  method that we call to actually return something to browser
- "filter.disconnect()": we need to disconnect the stream, as the default behavior is to keep the request open without a response

You can read more about filterResponseData object and it's behaviour [here](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/filterResponseData).

Once we receive full response, we need to decode it, find videos we want to delete, and delete them!

# Conclusions
Intercepting requests with Firefox extensions is simple yet awesome! 

Full source code is available on [Github](https://github.com/Demaga/lean-youtube). (UPD: not maintained)

There is also [a gist](https://gist.github.com/Demaga/4035a5c811d5a1a9ef758da43e6a3822) with minimal working example of requests interception.

# Sources

1. [https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Background_scripts](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Background_scripts)
2. [Lean YouTube for Firefox](https://addons.mozilla.org/en-US/firefox/addon/lean-youtube/) (UPD: no longer working)
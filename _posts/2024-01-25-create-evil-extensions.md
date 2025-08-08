---
title: Create Evil Firefox Extensions With JS-Tap Part 1
author: Zach Crosman
date: 2024-01-25 18:32:00 -0500
categories: [Pentesting, Tutorial, Red Team]
tags: [extensions, javascript, injection, firefox]
image:
  path: /images/firefox-js-tap/firefox-js-tap-small.png
  alt: Firefox JS-Tap extension modification
toc: true
pin: true
---

> The associated GitHub repository for scripts used in this blog post is located at [github.com/zcrosman/firefox-extension-js-tap](https://github.com/zcrosman/firefox-extension-js-tap)
{: .prompt-info }

## Intro
![Logo](/images/firefox-js-tap/firefox-js-tap-small.png){: height="200" .w-50 .right}
Browser extensions are like double agents in your browser - they offer handy features but often at the cost of your privacy. They can access all your online activities, and if they turn malicious or get compromised, it's game over for your data.

Modifying browser extensions to become malicious is alarmingly easy. Extensions are built using web technologies like JavaScript, HTML, and CSS. The source code for these extensions is already stored on your computer and easy to manipulate. A hacker can inject malicious code into an otherwise benign extension by directly modifying the extension's code to access EVERYTHING! This includes cookies, user input, local storage, and sites visited.

I'll walk through how to create a malicious Firefox extension with a payload from [JS-Tap](https://trustedsec.com/blog/js-tap-weaponizing-javascript-for-red-teams). JS-Tap is a versatile JavaScript payload designed for red team operations. It's used for attacking web applications, either as an XSS payload or as a general JavaScript payload. I'll review how to use this with Firefox extensions so EVERY loaded page is injected and we can collect all of the valuable information from a web portal.

[JS-Tap is capable of collecting the following](https://github.com/hoodoer/JS-Tap#data-collected):
- Client IP address, OS, Browser
- User inputs (credentials, etc.)
- URLs visited
- Cookies (that don't have httponly flag set)
- Local Storage
- Session Storage
- HTML code of pages visited (if feature enabled)
- Screenshots of pages visited
- Copy of XHR API calls (if monkeypatch feature enabled)
    - Endpoint
    - Method (GET, POST, etc.)
    - Headers set
    - Request body and response body
- Copy of Fetch API calls (if monkeypatch feature enabled)
    - Endpoint
    - Method (GET, POST, etc.)
    - Headers set
    - Request body and response body

## JS-Tap Setup
The JS-Tap installation has a very basic setup process. It's important to modify the `telemlib.js` file as required in the [setup guide](https://github.com/hoodoer/JS-Tap#installation-and-start). Make sure that this payload is working with the tools included in `tools/` (`clientSimulator.py` & `monkeyPatchLab.py`).

It's also important to note that although I'm using JS-Tap in this walkthrough, any JavaScript payload could be interchanged with this one. This payload will be fairly persistent in the browser and constantly collecting data on every site to send back to the server. Alternatively, payloads could be set to run only a single time or target specific websites. The possibilities are endless with this type of attack.

## Unpacking Extensions
The exact location of packed extensions in the Firefox application folder varies based on the operating system's file system, with specific locations for Windows, macOS, and Linux systems:

- **Windows**: `C:\Users\<UserName>\AppData\Roaming\Mozilla\Firefox\Profiles\<ProfileName>\extensions`
- **macOS**: `~/Library/Application Support/Firefox/Profiles/<ProfileName>/extensions`
- **Linux**: `~/.mozilla/firefox/<Profile Name>/extensions`

In these folders, Firefox extensions are packaged as `.xpi` files. This format can be unzipped to reveal all of the source code of the extension. A quick bash one-liner can be used to unpack all of these extensions in this folder:

```bash
# Unzip all of the extensions in the extensions folder
for f in *.xpi; do unzip "$f" -d "${f%*.xpi}"; done
```

## Extension Analysis
Once all of the extensions are unpacked, we can try to find a good target. Each of the extensions contains the file `manifest.json`. This is the extension's configuration file that includes permissions and additional extension information. Many of these extensions use an ID as the folder/extension name, which makes it slightly more confusing to tell them apart. Another bash one-liner can be used to quickly get more info about these extensions.

Quickly view the extension names and descriptions associated with each folder:
```bash
for d in */; do echo "$d"; jq '.name, .description' "$d/manifest.json"; done
```

Example output:
```
------SNIP-------
87677a2c52b84ad3a151a4a72f5bd3c4@jetpack/
Grammarly: Grammar Checker and AI Writing App
Improve your writing with Grammarly's communication assistance. Spell check, grammar check, and punctuation check in one tool. Real-time suggestions for improving tone and clarity help ensure your writing makes the impression you want.
------SNIP------
"
```

Some of these configurations are interesting for this project. Further analysis can be done to view more details by manually reviewing the manifest files or using a script to display additional information.

For example, using a custom search script:
```bash
./search-ext.sh extensions/
```

Example output:
```
------SNIP-------
Directory: extensions/wappalyzer@crunchlabz.com/
Name: Wappalyzer - Technology profiler
Description: Identify web technologies
Background Scripts: not set
Content Script Matches: http://*/*, https://*/*
Content Script Exclude Matches: not set
Content Script JS: js/content.js
Permissions: cookies, storage, tabs, webRequest, http://*/*, https://*/*
------SNIP-------
```

For the rest of this walkthrough, I'll use the Wappalyzer extension as an example. Wappalyzer is an extension that's used to identify the technology used on websites. I want to point out some important parts of the `manifest.json` file that are crucial to the implant:

- `content_scripts.matches` - Sites that the extension will run the specified JavaScript on
- `content_scripts.excludes` - Sites that are excluded from the JavaScript execution
- `content_scripts.js` - JavaScript files that are executed in the background of the specified sites

In this example, all that would need to be modified is the `content_scripts.js` since it will already run on all sites.

## Tap That Extension
I wrote a [bash script](https://github.com/zcrosman/firefox-extension-js-tap/) to easily update the `manifest.json` to include the necessary settings and drop the JavaScript file in the required folder. I'll walk through how it works.

This is what part of the `manifest.json` looks like before modification for the Grammarly extension. The extension runs `src/js/Grammarly-check.js` excluding the pages in "exclude_matches":

```json
"content_scripts": [
    {
      "all_frames": true,
      "match_about_blank": true,
      "css": [
        "src/css/Grammarly-fonts.styles.css"
      ],
      "js": [
        "src/js/Grammarly-check.js"
      ],
      "matches": [
        "<all_urls>"
      ],
      "exclude_matches": [
        "*://outlook.live.com/*",
        ----SNIP----
        "*://docs.google.com/document/*"
      ],
      "exclude_globs": [
        "*docs.google.com*"
      ],
      "run_at": "document_idle"
    }
 ]
```

**Updated Version:**
```json
"content_scripts": [
    {
        "all_frames": true,
        "match_about_blank": true,
        "css": [
            "src/css/Grammarly-fonts.styles.css"
        ],
        "js": [
            "src/js/Grammarly-check.js",
            "js/telemlib.js"
        ],
        "matches": [
            "http://*/*",
            "https://*/*"
        ],
        "run_at": "document_idle"
    }
]
```

The process involves injecting a JavaScript payload (like JS-Tap) into a Firefox extension. This script updates the extension's `manifest.json` file and slips in our payload to run in the background. It also ensures that the script runs on all pages if it wasn't configured to do so already.

### How Does It Work?

**Updating manifest.json**: The script adds our payload to the content scripts array, telling the extension to run our malicious script on every website.

**Moving the Payload**: The script then moves the payload (our `telemlib.js` file) to where the extension expects to find it. File location is crucial for proper execution.

The modification process involves:
1. Loading and parsing the existing `manifest.json`
2. Modifying the content scripts configuration to include our payload
3. Adding the malicious JavaScript file to the extension directory
4. Ensuring proper permissions are set for data collection

## Testing Your Tapped Extensions
**Load the Extension in Firefox**: Head to `about:debugging`, click "Load Temporary Add-on", and choose your modified `manifest.json`. Your extension now has enhanced capabilities! Once you load the extension, it will automatically be loaded into all of your existing Firefox tabs and future tabs or windows.

![Import Extension](/images/firefox-js-tap/firefox-extension-import.png)

### See It in Action
Open Firefox's Network tab to watch your payload in action. You can witness in real-time as your injected JavaScript interacts with each page, capturing and transmitting data seamlessly. If you're using an unmodified JS-Tap payload, you should also see some logs in the browser console.

### JS-Tap Dashboard Filled With Data
Watch as data starts flooding in from the malicious extension on the JS-Tap dashboard. The dashboard becomes a hub of activity, displaying information collected from the target browser sessions.

![JS-Tap Extensions](/images/firefox-js-tap/js-tap-connections.png)

## Conclusion
This walkthrough demonstrates how easy it is to weaponize legitimate browser extensions for red team operations. By modifying the `manifest.json` and injecting malicious JavaScript payloads, attackers can gain persistent access to all user browsing data. This technique highlights the importance of extension security and the need for users to be cautious about the extensions they install.



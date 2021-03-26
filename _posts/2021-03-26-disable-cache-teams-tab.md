---
layout: post
title: "Preventing Teams from caching your tab"
date: 2021-03-26
---

I was developing a Teams tab recently and noticed that the Teams client was caching my static content. So even when I made updates to the React frontend and reloaded the tab, Teams would show the cached version instead of the updated version. After some digging, I came across a way to prevent Teams from caching my pages.

<!--more-->

Typically I do most of my development in the browser version of Teams, where you can more easily open dev tools and check "Disable cache" under the Network tab. But often it's useful or even necessary to test things in the desktop client as well. You can [open DevTools for the desktop client](https://docs.microsoft.com/en-us/microsoftteams/platform/tabs/how-to/developer-tools), but if you try the same "Disable cache" method, you'll notice that reloading your tab immediately closes the DevTool window. And the "Disable cache" setting only applies while the DevTool window is open.

(By the way, if you don't see the DevTools option, try left-clicking on the Teams icon in the system tray 10-15 times. Then right-click and it should show up.)

On to the trick: After checking "Disable cache", instead of reloading your tab by clicking reload in Teams, open the console in DevTools and refresh the page using `window.location.reload()`. This keeps the overall page/iframe open, which keeps your DevTools open and disables cache for the reload. You can do the same thing for other Teams web content too, like task modules.

Credit to [Adrian Solis for the tip on GitHub](https://github.com/MicrosoftDocs/msteams-docs/issues/805#issuecomment-513004106). I just didn't trust myself to find that comment again in the future.
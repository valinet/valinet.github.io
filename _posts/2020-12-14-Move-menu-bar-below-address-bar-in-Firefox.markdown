---
layout: post
title:  "Move menu bar below address bar in Firefox"
date:   2020-12-14 00:00:00 +0000
categories: 
excerpt: "I have recently rethought my choices regarding the applications I use on a day to day basis. So, I have now officially walked away from Chromium-based forks into the truly libre Mozilla territory. To be honest, Firefox is really great, providing much better font rendering when compared to Chromium (yet still a bit worse than Internet Explorer!), and amazingly, it still even includes a functional menu bar so you are not forced to deal with the one-menu-to-rule-them-all approach of the hamburger button. The problem is, it kind of ruins the design, because it is drawn at the very top of the window. I don't want to enable the window manager's title bar, because that way I lose Fitt's Law benefits when the window is maximized and can quickly switch tabs. This article shows a potential solution, which invloves injecting custom CSS and JavaScript in Firefox's chrome."
---

I have recently rethought my choices regarding the applications I use on a day to day basis. So, I have now officially walked away from Chromium-based forks into the truly libre Mozilla territory. To be honest, Firefox is really great, providing much better font rendering when compared to Chromium (yet still a bit worse than Internet Explorer!), and amazingly, it still even includes a functional menu bar so you are not forced to deal with the one-menu-to-rule-them-all approach of the hamburger button. The problem is, it kind of ruins the design, because it is drawn at the very top of the window. I don't want to enable the window manager's title bar, because that way I lose Fitt's Law benefits when the window is maximized and can quickly switch tabs. This article shows a potential solution, which invloves injecting custom CSS and JavaScript in Firefox's chrome.

I have used Firefox in the past, back in the days when you could skin it to look like [Internet Explorer 8](https://media.askvg.com/articles/images/Use_Color_Tab.png) (indeed, that's Firefox, not actual IE8 which looked like [this](https://docs.microsoft.com/en-us/windows/win32/dwm/images/ie7-extendedborder-boxed.png)) for example. Then, Chrome came along and the reality today is that Chrome is the new IE, for better or for worse. I use Windows 10 daily and have long wanted to switch to Firefox, which is my choice when using GNU/Linux because it supports Wayland and has recently added support for hardware video decoding. Plus, Firefox is the true libre browser, and it is actually a very good product, so, why not make the switch already?

Anyway, enough of the background, how do you do this? Well, [Firefox's UI is amazingly written in Web technologies](https://briangrinstead.com/blog/firefox-webcomponents/) (and has been so since the Netscape days, which is even more impressing), so the layout is described by CSS, naturally.

So, the first step is to enable the use of userChrome.css in Firefox as described [here](https://www.userchrome.org/how-create-userchrome-css.html). Once that is in place, you can override the rules for any element in the UI.

Now, how do you find out the elements in the UI? Well, besides being able to 'inspect' pages, the F12 Developer Tools allow you to actually inspect the whole window. The feature is called *Browser Toolbox*, and you can activate it as described [here](https://developer.mozilla.org/en-US/docs/Tools/Browser_Toolbox).

Once that's done, you can peek at the DOM of the Firefox window. Sure enough, one can quickly identify the elements that we need to deal with, but the problem is that, even with tricks like `moz-box-ordinal-group` that can reorder stuff, it cannot do so if the things are not on the same level (have the same parent). Or at least that is how I see it. The menu bar is actually a child of the title bar (which includes the tabs), so changing its position from using `moz-box-ordinal-group` would allow me to have the address bar and navigation buttons on top, and tabs and the menus at the bottom. Not really what I want.

So, what's the solution? I will first explain what I haven't tried. Basically, someone [suggested](https://www.reddit.com/r/FirefoxCSS/comments/kd1gmk/hide_titlebar_on_windows_without_losing_window/) that I may play around with the positioning using the `position: relative` trick. I am no expert in CSS, so I tend to stay off complicated solutions, and this certainly gets pretty complicated when you think about all the edge cases. And it is a mere workaround for what is actually required: altering the DOM and moving the menu bar element under the address bar element.

The chrome of Firefox being built using web technologies, you can of course run JavaScript on it. The problem is how do you do so nowadays, after the numerous architectural changes? Well, Firefox still has a mechanism that allows creating a framework for custom JS injection that is meant for enterprise customizations, [autoconfig.js](https://support.mozilla.org/en-US/kb/customizing-firefox-using-autoconfig). 

Around this, people build loaders for custom JS in the user profile folder. So, I used xiaoxiaoflood's loader available [here](https://github.com/xiaoxiaoflood/firefox-scripts/).

After you install that, all that is left to do is to create a new JS script which will do the actual DOM modifications. To keep everything tidy up, the few CSS styles I have to perform for the mod are actually done in that script file as well, using JavaScript. Of course, these can be migrated to the userChrome.css file.

So, create a new file in the "chrome" folder in your user profile folder (to find that out, in Firefox, go to Help - Troubleshooting Information and click on Open Profile); I called mine "menubar.uc.js". What is important is that it ends in ".uc.js":

{% highlight javascript%}
// ==UserScript==
// @name            menubar on bottom
// @include         main
// @author          valinet
// ==/UserScript==

UC.menubarbottom = {
  resize: function() {
	var elem = document.getElementById("TabsToolbar").getElementsByClassName("toolbar-items")[0];
	if (window.windowState == 1) // 1 = maximized, 3 = normal
	{
	  elem.style = "";
	}
	else
	{
	  elem.style = "padding-top: var(--space-above-tabbar)";
	}
  },
  
  init: function () {
	  
	let menubar = document.getElementById("toolbar-menubar");
	let tbParent = document.getElementById("navigator-toolbox");
	
	// hide Min, Max, Close buttons on the menu bar
	menubar.getElementsByClassName("titlebar-buttonbox-container")[0].style = "display: none";
	
	// style menu bar with semi-transparent overlay (same as address bar)
	menubar.style = "background-color: var(--toolbar-bgcolor);";
	
	// move menu bar bellow the address bar
	tbParent.appendChild(menubar);
	
	// add/remove top padding (drag space) according to window being normal/maximized
	// https://developer.mozilla.org/en-US/docs/Archive/Mozilla/XUL/Events#Window_events
	UC.menubarbottom.resize();
	window.addEventListener('sizemodechange', UC.menubarbottom.resize);
	
	// move bookmarks bar bellow menu bar
	tbParent.appendChild(document.getElementById("PersonalToolbar"));
  }
}

UC.menubarbottom.init();
{% endhighlight %}

That's it. Feel free to tweak its behavior if you see fit (for example, I actually have the bookmarks toolbar underneath the menu bar as well as I think it is more natural like that, but you can skip that by commenting out that line of code). There are also some rules I like to apply in my userChrome.css related to this:

{% highlight css%}
/* disable grabber space before the tab bar when window is not maximized */
.titlebar-spacer[type="pre-tabs"] {
  max-width: 0px !important;
}

/* disable hamburger menu */
#PanelUI-button {
  display: none;
}

/* disable line below tabs */
#nav-bar {
  box-shadow: 0px 0px #000000 !important;
}
{% endhighlight %}

What's left is, of course, to maybe use an awesome theme like [Firefox Alpenglow](https://addons.mozilla.org/ro/firefox/addon/firefox-alpenglow/) and even a different, more colorful new tab page such as [Tabliss](https://addons.mozilla.org/ro/firefox/addon/tabliss/) (for it, I have the following custom CSS rule that inlines the quick links: `.Links { display: inline; }`).

That's it, feel free to share your thoughts below.
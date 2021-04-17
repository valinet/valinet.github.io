---
layout: post
title:  "Open SharePoint files in client apps directly in modern browsers"
date:   2021-04-10 00:00:00 +0000
categories: 
excerpt: "Recently, I had to set up a SharePoint instance at my workplace - the reasons are complicated, but mainly, it turns out that SharePoint it is way friendlier to use than free alternatives like Nextcloud. Indeed, I was easily able to set up permissions, version history, mandatory check out policies, but have faced a rather unpleasant issue: SharePoint seems to play best with Internet Explorer. Not anymore... read on!"
---

TL;DR: The project's source code and precompiled binaries can now be obtained from a dedicated Git repo [here](https://github.com/valinet/SharePointLinkHelper).

Recently, I had to set up a SharePoint instance at my workplace - the reasons are complicated, but mainly, it turns out that SharePoint it is way friendlier to use than free alternatives like Nextcloud. Indeed, I was easily able to set up permissions, version history, mandatory check out policies, but have faced a rather unpleasant issue: SharePoint seems to play best with Internet Explorer. Not anymore... read on!

To start off, let me tell you that SharePoint can be obtained for free, as well. And not illegally: in fact, Microsoft themselves provide a free version, albeit only for the older 2013 release. If you have only basic needs from a document management system, want SharePoint's ease of use and good integration with Windows, and do not mind firing up a Windows Server instance (or maybe you already run one), this edition is the way to go. It is called [SharePoint Foundation 2013](https://www.microsoft.com/en-us/download/details.aspx?id=42039) and available free of charge from Microsoft.

Now, the workflow I envisioned is for people to store their documents in SharePoint libraries and to have them edit documents directly in there, if needed, automatically checking them out (which means locking the document while edited by one person such that other people attempting to edit are prevented from doing so; it is a rudimentary, but efficient form of access control, in order to avoid having regular people deal with concepts like Git's manual merging, for example). This means users should be able to click the document in the SharePoint portal viewed in the browser, and have it directly open up in the corresponding application on the PC.

To pull this off, first, you need applications that are SharePoint aware, in that they know how to open and edit stuff directly from there. My users only use Office and PDF documents in SharePoint, and both Microsoft Office and Adobe Acrobat Reader have support for SharePoint.

But, the big problem is opening the files directly: modern browsers do not feature an 'Open' button like Internet Explorer did back in the day, for questionable security reasons; they instead download the file locally. You could have everyone use Internet Explorer for accessing SharePoint, but I still encountered issues with opening PDF files directly, despite applying [a fix that mostly worked](https://community.adobe.com/t5/enterprise-teams/sharepoint-2013-pdf-check-out-and-open-option/td-p/5833645), plus, no one wants to switch browsers. Then, you could keep your users in Chrome and force the site to render in IE in the Chrome window with something like the [IETab](https://chrome.google.com/webstore/detail/ie-tab/hehijbfgiekmjfkfjpbkbammjbdenadd?hl=ro) extension, but, again, it shows a doubled address bar, it is still IE, has some differences from the regular IE, the extension is very locked down as it is essentially shareware and so on... so, available solutions are half way there, but not all the way.

So, what did I do? Reading various sites online, I decided to [apply the latest SharePoint Foundation 2013 update](https://support.microsoft.com/en-us/topic/march-9-2021-cumulative-update-for-sharepoint-foundation-2013-kb4493235-2d0d8fab-562c-a16b-81a9-cbe51c8f5c3d), thinking maybe it fixes some of the issues. As far as I have noticed, visual changes include slight CSS tweaks, like bolding selected items in the navigation bars, while regarding the functionality I was looking for, it now made Chrome and non-IE browsers pop up messages asking whether I'd like to open Office files in the client applications, like it does in IE. PDFs still no luck, but nevertheless an improvement.

But I wanted something better, that worked seamlessly: the user clicks a document and it opens instantly. Is that possible?

Having previous experience, I decided that registering a custom protocol on the system is the best option. Then, modify all links to documents in SharePoint and prepend this protocol scheme to them, so that the browser fires up my handler app where I take the URL provided by the browser and properly open it in corresponding client app (I determine this by looking at the extension).

To edit all links in the document libraries view, I added some JavaScript to the theme file of the web site, in my case, `seattle.master`. I edited that in [SharePoint Designer 2013](https://www.microsoft.com/en-us/download/details.aspx?id=35491) and added something like this:

{% highlight javascript %}
<script>
function LoadFunc(){
if (document.getElementById("Ribbon.Document-title"))
{
	var app_elems = document.getElementsByClassName("ms-vb");
	for (var i = 0; i < app_elems.length; i++)
	{
		app_elems[i].setAttribute("app", "");
	}
	var link_elems = document.getElementsByClassName("ms-listlink");
	for (var i = 0; i < link_elems.length; i++)
	{
		link_elems[i].setAttribute("href", "splinkhelper://" + link_elems[i].href);
	}
}
</script>
...
<body onload="LoadFunc();">
{% endhighlight %}

This works well, but the browser still pops up a confirmation. Can we get rid of that? Well, modern Chrome versions include a new policy registry key that allows this: `AutoLaunchProtocolsFromOrigins` (described [here](https://docs.microsoft.com/en-us/deployedge/microsoft-edge-policies)). For my use case, I configure it like this: `[{"allowed_origins": ["http://sharepoint"], "protocol": "splinkhelper"}]`. The key to set is located in HKLM or HKCU, at: `\SOFTWARE\Policies\Google\Chrome` (for Google Chrome) or `\SOFTWARE\Policies\Microsoft\Edge` (for Microsoft Edge Chromium).

And now, you only need an application that will fire up when the protocol is invoked, determine the extension from the URL and open it up in the default application handling that file type; this is enough for the application to start working directly off the server.

For this task, I wrote a small C app based on my previous [Thunderbird Toasts](https://github.com/valinet/ThunderbirdToasts/) that just registers itself as a handler for the `splinkhelper` protocol, takes whatever arguments it received as input, determines the extension, finds out which application is supposed to open that extension, and fires it up with the input as arguments. To install, simply run the file. Similarly, to uninstall, run the file again. You can delete the downloaded file after  installation, as it copies itself into the application data folder.

Get the project's source code and precompiled binaries from its dedicated Git repo [here](https://github.com/valinet/SharePointLinkHelper).

Current limitations include the fact that the `AutoLaunchProtocolsFromOrigins` is overwritten at install with a value suitable for using just this app properly. In the future, I'll have to properly check and alter that respective location in the registry from the code.

Later edit: Also, almost forgot: you need to edit `INIT.JS` in `C:\Program Files\Common Files\microsoft shared\Web Server Extensions\15\TEMPLATE\LAYOUTS` and change the following 2 functions to always return `true`: `IsSTSPageUrlValid` and `SupportsNavigateHttpFolder`. The first one makes it so that SharePoint does not throw an error when clicking a link with this custom protocol of ours, while the second one makes the `Open in Explorer` button available in non-IE browsers as well. To make it functional with the help of this helper app, execute this JavaScript when the page loads:

{% highlight javascript %}
if (!browseris.ie5up)
{
	NavigateHttpFolder = function(a, b) { 
		window.location.href = "splinkhelper://#" + a;
	};
}
{% endhighlight %}

P.S. I think that changing the links in the document list views can be done using JSLink or something like that as well, instead of injecting JavaScript directly in the master page, yet I did not remember at the moment how it was done.

P.P.S. SharePoint Foundation 2013 comes bundled with and installs SQL Server 2008 R2 Express as its database. The Express version is limited to 10GB per database. Beware that SharePoint, by default, stores the files added to the libraries inside the database. Thus, depending on what your users upload, the database may fill up rather soon. To avoid this, there is a tech that allows it to offload the binary blobs to the separate files on disk, leaving the database untouched. It is called RBS and a pretty solid installation guide for it is available on YouTube (embedded below).

{%- include youtube_embed.html id="Bs2iGX_zEtw" ratio="9/16" -%}

RBS needs periodic management: a garbage collector runs on the binary blobs, checks whether they correspond to files deleted for good from SharePoint (there are a number of retention periods; files in Recycle Bin get deleted automatically after some time, and from there, it takes another period until the file is made available for garbage colleciton). 

To test the garbage collector, you can lower the retention periods. For that, run this in SQL Management Studio on your 'WSS_Content' database:

```
exec mssqlrbs.rbs_sp_set_config_value ‘garbage_collection_time_window’, ‘time 00:00:00’;
exec mssqlrbs.rbs_sp_set_config_value ‘delete_scan_period’, ‘time 00:00:00’;
exec mssqlrbs.rbs_sp_set_config_value ‘orphan_scan_period’, ‘time 00:00:00’;
```

Then, run the garbage collector using this:

```
C:\Program Files\Microsoft SQL Remote Blob Storage 10.50\Maintainer\Microsoft.Data.SqlRemoteBlobs.Maintainer.exe -connectionstringname RBSMaintainerConnection -operation GarbageCollection ConsistencyCheck ConsistencyCheckForStores -GarbageCollectionPhases rdo -ConsistencyCheckMode r -TimeLimit 120
```

P.P.P.S. If SharePoint ever refuses to upload a file with the same name of a previous file in some library, consider flushing the blob cache of SharePoint - just type this into an elevated PowerShell.

```
Add-PSSnapin Microsoft.Sharepoint.Powershell
$webApp = Get-SPWebApplication "http://sharepoint/"
Add-Type -Path 'C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.SharePoint.Publishing\v4.0_15.0.0.0__71e9bce111e9429c\Microsoft.SharePoint.Publishing.dll'
[Microsoft.SharePoint.Publishing.PublishingCache]::FlushBlobCache($webApp)
Write-Host "Flushed the BLOB cache for:" $webApp
```


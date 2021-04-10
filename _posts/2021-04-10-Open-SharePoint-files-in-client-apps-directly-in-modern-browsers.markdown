---
layout: post
title:  "Open SharePoint files in client apps directly in modern browsers"
date:   2021-04-10 00:00:00 +0000
categories: 
excerpt: "Recently, I had to set up a SharePoint instance at my workplace - the reasons are complicated, but mainly, it turns out that SharePoint it is way friendlier to use than free alternatives like Nextcloud. Indeed, I was easily able to set up permissions, version history, mandatory check out policies, but have faced a rather unpleasant issue: SharePoint seems to play best with Internet Explorer. Not anymore... read on!"
---

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

[Here]() is the binary application, with the full source code available below. The code is short, I bet half of it is just error checking. Copy the contents to a new Visual Studio solution file, in a new .c file and you are good to go. Just make sure to link with `Shlwapi.lib` as well. or, you can try compiling from the command window using something like:

`cl.exe /nologo /DUNICODE main.c Shlwapi.lib /link /RELEASE /ENTRY:wWinMain`

The command above ensures a pretty small file size, as the C Runtime is skipped from the executable, for example. 

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

And here is the source code, enjoy!

{% highlight c %}
//#undef UNICODE
#pragma comment(linker,"\"/manifestdependency:type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' processorArchitecture='*' publicKeyToken='6595b64144ccf1df' language='*'\"")
#include <Windows.h>
#include <Shlobj.h>
#include <shlwapi.h>
//#pragma comment(linker, "/NODEFAULTLIB")
//#pragma comment(linker, "/ENTRY:wWinMain")

#ifdef UNICODE
#define len wcslen
#define cat wcscat_s
#define str wcsstr
#define chr wcschr
#else
#define len strlen
#define cat strcat_s
#define str strstr
#define chr strchr
static CHAR* widestr2str(const WCHAR* wstr)
{
    int wstr_len = (int)wcslen(wstr);
    int num_chars = WideCharToMultiByte(CP_UTF8, 0, wstr, wstr_len, NULL, 0, NULL, NULL);
    CHAR* strTo = (CHAR*)malloc((num_chars + 1) * sizeof(CHAR));
    if (strTo)
    {
        WideCharToMultiByte(CP_UTF8, 0, wstr, wstr_len, strTo, num_chars, NULL, NULL);
        strTo[num_chars] = '\0';
    }
    return strTo;
}
#endif

#define APP_NAME                                                                          "SharePointLinkHelper"
#define APP_NAME_LENGTH                                                                      len(TEXT(APP_NAME))
#define PROTOCOL_NAME                                                                             "splinkhelper"
#define EXE_NAME                                                                     TEXT(APP_NAME) TEXT(".exe")
#define SP_SITE                                                                              "http://sharepoint"
#define WELCOME_PAGE                            TEXT(SP_SITE) TEXT("/SitePages/SharePoint%20Link%20Helper.aspx")

const char reg[] =
"Windows Registry Editor Version 5.00\r\n"
"[HKEY_CURRENT_USER\\SOFTWARE\\Policies\\Microsoft\\Edge]\r\n"
"\"AutoLaunchProtocolsFromOrigins\" = \"[{\\\"allowed_origins\\\": [\\\"" SP_SITE "\\\"], \\\"protocol\\\": \\\"" PROTOCOL_NAME "\\\"}]\"\r\n"
"[HKEY_CURRENT_USER\\SOFTWARE\\Policies\\Google\\Chrome]\r\n"
"\"AutoLaunchProtocolsFromOrigins\" = \"[{\\\"allowed_origins\\\": [\\\"" SP_SITE "\\\"], \\\"protocol\\\": \\\"" PROTOCOL_NAME "\\\"}]\"\r\n"
;

HRESULT GetWorkingDirectory(TCHAR* directory)
{
    TCHAR* szAppName = EXE_NAME;
    BOOL bRet = FALSE;
    HRESULT hr = SHGetFolderPath(
        NULL,
        CSIDL_APPDATA,
        NULL,
        SHGFP_TYPE_CURRENT,
        directory
    );
    if (SUCCEEDED(hr))
    {
        memcpy(
            directory + len(directory),
            TEXT("\\") TEXT(APP_NAME),
            (1 + APP_NAME_LENGTH + 1) * sizeof(TCHAR)
        );
        bRet = CreateDirectory(
            directory,
            NULL
        );
        if (bRet || (bRet == 0 && GetLastError() == ERROR_ALREADY_EXISTS))
        {
            memcpy(
                directory + len(directory),
                TEXT("\\"),
                (1 + 1) * sizeof(TCHAR)
            );
            memcpy(
                directory + len(directory),
                szAppName,
                (len(szAppName) + 1) * sizeof(TCHAR)
            );
        }
        else
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
    }
    else
    {
        return hr;
    }
    return S_OK;
}

HKEY GetRootKey()
{
    /*
    if (IsUserAnAdmin())
    {
        return HKEY_CLASSES_ROOT;
    }
    */
    return HKEY_CURRENT_USER;
}

HRESULT InstallOrUninstall(HRESULT uninstall)
{
    HRESULT hr = S_OK;
    LSTATUS ret = 0;
    DWORD dwLen;
    TCHAR szCommandName[MAX_PATH + 7];
    TCHAR szAppName[MAX_PATH];
    HKEY root = GetRootKey();
    DWORD dwDisposition;
    DWORD dwVal = 2162688;
    HKEY key;
    HMODULE hModule;
    HINSTANCE hErr;

    if ((hModule = GetModuleHandle(NULL)) == NULL)
    {
        return HRESULT_FROM_WIN32(GetLastError());
    }
    if (!GetModuleFileName(
        hModule,
        szAppName,
        MAX_PATH
    ))
    {
        return HRESULT_FROM_WIN32(GetLastError());
    }
    if (FAILED(hr = GetWorkingDirectory(szCommandName)))
    {
        return hr;
    }
    if (uninstall == S_FALSE)
    {
        if (!CopyFile(
            szAppName,
            szCommandName,
            FALSE
        ))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
    }
    else
    {
        if (!DeleteFile(
            szCommandName
        ))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
        cat(
            szCommandName,
            MAX_PATH,
            TEXT(".reg")
        );
        if (!DeleteFile(
            szCommandName
        ))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
        for (int i = len(szCommandName) - 1; i >= 0; i--)
        {
            if (szCommandName[i] == '\\')
            {
                szCommandName[i] = 0;
                break;
            }
        }
        if (!RemoveDirectory(szCommandName))
        {
        }
    }
    if (FAILED(hr = GetWorkingDirectory(szCommandName + 1)))
    {
        return hr;
    }
    dwLen = len(szCommandName);
    szCommandName[0] = '"';
    szCommandName[dwLen] = '"';
    szCommandName[dwLen + 1] = ' ';
    szCommandName[dwLen + 2] = '"';
    szCommandName[dwLen + 3] = '%';
    szCommandName[dwLen + 4] = '1';
    szCommandName[dwLen + 5] = '"';
    szCommandName[dwLen + 6] = 0;
    if (uninstall == S_FALSE)
    {
        if ((ret = RegCreateKeyEx(
            root,
            TEXT("SOFTWARE\\Classes\\")
            TEXT(PROTOCOL_NAME),
            0,
            NULL,
            REG_OPTION_NON_VOLATILE,
            KEY_ALL_ACCESS,
            NULL,
            &key,
            &dwDisposition
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        if ((ret = RegSetValueEx(
            key,
            TEXT(""),
            0,
            REG_SZ,
            TEXT("URL:")
            TEXT(PROTOCOL_NAME)
            TEXT(" protocol"),
            (4 + len(TEXT(PROTOCOL_NAME)) + 9) * sizeof(TCHAR)
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        if ((ret = RegSetValueEx(
            key,
            TEXT("EditFlags"),
            0,
            REG_DWORD,
            &dwVal,
            sizeof(DWORD)
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        if ((ret = RegSetValueEx(
            key,
            TEXT("URL Protocol"),
            0,
            REG_SZ,
            L"",
            sizeof(TCHAR)
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        if ((ret = RegCloseKey(key)) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }

        if ((ret = RegCreateKeyEx(
            root,
            TEXT("SOFTWARE\\Classes\\")
            TEXT(PROTOCOL_NAME)
            TEXT("\\shell\\open\\command"),
            0,
            NULL,
            REG_OPTION_NON_VOLATILE,
            KEY_ALL_ACCESS,
            NULL,
            &key,
            &dwDisposition
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        if ((ret = RegSetValueEx(
            key,
            L"",
            0,
            REG_SZ,
            szCommandName,
            (len(szCommandName) + 1) * sizeof(TCHAR)
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        if ((ret = RegCloseKey(key)) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }

        if (FAILED(hr = GetWorkingDirectory(szCommandName)))
        {
            return hr;
        }
        cat(
            szCommandName,
            MAX_PATH,
            TEXT(".reg")
        );
        HANDLE hFile = CreateFile(
            szCommandName,
            GENERIC_WRITE,
            FILE_SHARE_READ,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );
        if (hFile == INVALID_HANDLE_VALUE)
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
        DWORD bytesWritten;
        if (!WriteFile(
            hFile,
            reg,
            len(reg) * sizeof(TCHAR),
            &bytesWritten,
            NULL
        ))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
        if (!CloseHandle(hFile))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
        if ((hErr = ShellExecute(
            NULL,
            TEXT("open"),
            WELCOME_PAGE,
            NULL,
            NULL,
            SW_SHOWDEFAULT
        )) <= 32)
        {
            return hErr;
        }
        if ((hErr = ShellExecute(
            NULL,
            TEXT("open"),
            szCommandName,
            NULL,
            NULL,
            SW_SHOWDEFAULT
        )) <= 32)
        {
            return hErr;
        }
    }
    else
    {
        if ((ret = RegDeleteTree(
            root,
            TEXT("SOFTWARE\\Classes\\")
            TEXT(PROTOCOL_NAME)
        )) != ERROR_SUCCESS)
        {
            return HRESULT_FROM_WIN32(ret);
        }
        MessageBox(
            NULL,
            TEXT("The product has been uninstalled successfully."),
            TEXT(APP_NAME),
            MB_ICONINFORMATION
        );
    }
    return S_OK;
}

HRESULT CheckIfInstalled()
{
    LSTATUS ret = 0;
    HKEY root = GetRootKey();

    HKEY key;
    if ((ret = RegOpenKeyEx(
        root,
        TEXT("SOFTWARE\\Classes\\")
        TEXT(PROTOCOL_NAME),
        0,
        KEY_ALL_ACCESS,
        &key
    )) != ERROR_SUCCESS)
    {
        return S_FALSE;
    }
    if ((ret = RegCloseKey(key)) != ERROR_SUCCESS)
    {
        return HRESULT_FROM_WIN32(ret);
    }
    return S_OK;
}

HRESULT LaunchExecutableForUrl(TCHAR* url)
{
    HINSTANCE hErr;
    HRESULT hr = CoInitializeEx(NULL, COINIT_APARTMENTTHREADED | COINIT_DISABLE_OLE1DDE);
    if (FAILED(hr))
    {
        return hr;
    }
    /*MessageBox(
        NULL,
        url,
        NULL,
        NULL
    );*/
    if (str(url, TEXT(PROTOCOL_NAME ":///#")))
    {
        TCHAR explorerPath[MAX_PATH];
        ZeroMemory(explorerPath, MAX_PATH * sizeof(TCHAR));
        hr = SHGetFolderPath(
            NULL,
            CSIDL_WINDOWS,
            NULL,
            SHGFP_TYPE_CURRENT,
            explorerPath
        );
        if (SUCCEEDED(hr))
        {
            cat(
                explorerPath,
                MAX_PATH,
                TEXT("\\explorer.exe")
            );
            // 5 = len(":///#")
            // 7 = len("http://")
            // 2 = len("\"") + len("\"")
            // 2 = len("\\\\")
            // 10 = len("davwwwroot")
            // 100 = some safety padding
            TCHAR* path = HeapAlloc(
                GetProcessHeap(),
                HEAP_ZERO_MEMORY,
                (len(url) - len(TEXT(PROTOCOL_NAME)) - 5 - 7 + 2 + 2 + 10 + 100) * sizeof(TCHAR)
            );
            if (!path)
            {
                return HRESULT_FROM_WIN32(STATUS_NO_MEMORY);
            }
            path[0] = TEXT('"');
            path[1] = TEXT('\\');
            path[2] = TEXT('\\');
            memcpy(
                path + 3,
                url + len(TEXT(PROTOCOL_NAME)) + 5 + 7,
                (len(url) - len(TEXT(PROTOCOL_NAME)) - 5 - 7) * sizeof(TCHAR)
            );
            TCHAR* test = chr(path, TEXT('/'));
            memcpy(
                test + 1,
                TEXT("davwwwroot"),
                10 * sizeof(TCHAR)
            );
            TCHAR* p = chr(url + len(TEXT(PROTOCOL_NAME)) + 5 + 7, '/');
            memcpy(
                test + 11,
                p,
                len(p) * sizeof(TCHAR)
            );
            test[11 + len(p)] = TEXT('\0');
            DWORD i;
            for (i = 0; i < len(path); ++i)
            {
                if (path[i] == TEXT('/'))
                {
                    path[i] = TEXT('\\');
                }
            }
            path[i] = TEXT('"');
            path[i + 1] = TEXT('\0');
            /*MessageBox(
                NULL,
                path,
                NULL,
                NULL
            );*/
            DWORD pcchUnescaped;
            hr = UrlUnescape(
                path,
                NULL,
                &pcchUnescaped,
                URL_UNESCAPE_INPLACE | URL_UNESCAPE_AS_UTF8
            );
            if (FAILED(hr))
            {
                return hr;
            }
            /*MessageBox(
                NULL,
                path,
                NULL,
                NULL
            );*/
            if ((hErr = ShellExecute(
                NULL,
                NULL,
                explorerPath,
                path,
                NULL,
                SW_SHOWDEFAULT
            )) <= 32)
            {
                return hErr;
            }
            if (!HeapFree(
                GetProcessHeap(),
                0,
                path
            ))
            {
                return HRESULT_FROM_WIN32(GetLastError());
            }
        }
    }
    else
    {
        TCHAR* ext = PathFindExtension(url);
        TCHAR* path;
        DWORD len = 0;
        hr = AssocQueryString(
            ASSOCF_NONE,
            ASSOCSTR_EXECUTABLE,
            ext,
            NULL,
            NULL,
            &len
        );
        if (hr != S_FALSE)
        {
            return hr;
        }
        path = HeapAlloc(
            GetProcessHeap(),
            0,
            len * sizeof(TCHAR)
        );
        if (!path)
        {
            return HRESULT_FROM_WIN32(STATUS_NO_MEMORY);
        }
        hr = AssocQueryString(
            ASSOCF_NONE,
            ASSOCSTR_EXECUTABLE,
            ext,
            NULL,
            path,
            &len
        );
        if (FAILED(hr))
        {
            return hr;
        }
        // 1 = string null terminator
        // 3 = len("://")
        if ((hErr = ShellExecute(
            NULL,
            NULL,
            path,
            url + (sizeof(TEXT(PROTOCOL_NAME)) / sizeof(TCHAR) - 1) + 3,
            NULL,
            SW_SHOWDEFAULT
        )) <= 32)
        {
            return hErr;
        }
        if (!HeapFree(
            GetProcessHeap(),
            0,
            path
        ))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
    }
    CoUninitialize();
    return S_OK;
}


int WINAPI wWinMain(
    _In_ HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_ LPWSTR lpCmdLine,
    _In_ int nShowCmd
)
{
    HRESULT hr = S_OK;

    if (!SetProcessDpiAwarenessContext(
        DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2
    ))
    {
        return HRESULT_FROM_WIN32(GetLastError());
    }

    size_t argc = 0;
    LPCWSTR cmdLine = GetCommandLineW();
    LPWSTR* Wargv = CommandLineToArgvW(cmdLine, &argc);
    if (argc == 2)
    {
#ifdef UNICODE
        hr = LaunchExecutableForUrl(Wargv[1]);
#else
        char* argv1 = widestr2str(Wargv[1]);
        hr = LaunchExecutableForUrl(argv1);
        free(argv1);
#endif
    }
    else if (argc == 1)
    {
        hr = CheckIfInstalled();
        if (SUCCEEDED(hr) || hr == S_FALSE)
        {
            hr = InstallOrUninstall(hr);
        }
    }
    if (LocalFree(Wargv))
    {
        return HRESULT_FROM_WIN32(GetLastError());
    }
    /*TCHAR buf[10];
    _itow_s(hr, buf, 10, 10);
    MessageBox(
        NULL,
        buf,
        NULL,
        NULL
    );*/
    ExitProcess(hr);
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


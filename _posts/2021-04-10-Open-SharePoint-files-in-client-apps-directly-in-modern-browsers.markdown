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

{% highlight javascript%}
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

`cl.exe /nologo /DUNICODE /Oi /GS- main.c Kernel32.lib User32.lib Shlwapi.lib /link /RELEASE /NODEFAULTLIB /ENTRY:wWinMain`

The command above ensures a pretty small file size, as the C Runtime is skipped from the executable, for example. 

Current limitations include the fact that the `AutoLaunchProtocolsFromOrigins` is overwritten at install with a value suitable for using just this app properly. In the future, I'll have to properly check and alter that respective location in the registry from the code.

Anyway, here is the source code, enjoy!

{% highlight c%}
#pragma comment(linker,"\"/manifestdependency:type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' processorArchitecture='*' publicKeyToken='6595b64144ccf1df' language='*'\"")
#include <Windows.h>
#include <Shlobj.h>
#include <shlwapi.h>
#pragma comment(linker, "/NODEFAULTLIB")
#pragma comment(linker, "/ENTRY:wWinMain")

#ifdef UNICODE
#define stringlen wcslen
#else
#define stringlen strlen
#endif

#define APP_NAME                                                                          "SharePointLinkHelper"
#define APP_NAME_LENGTH                                                                stringlen(TEXT(APP_NAME))
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
            directory + stringlen(directory),
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
                directory + stringlen(directory),
                TEXT("\\"),
                (1 + 1) * sizeof(TCHAR)
            );
            memcpy(
                directory + stringlen(directory),
                szAppName,
                (stringlen(szAppName) + 1) * sizeof(TCHAR)
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
    TCHAR szCommandName[_MAX_PATH + 7];
    TCHAR szAppName[_MAX_PATH];
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
        _MAX_PATH
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
        wcscat(
            szCommandName,
            TEXT(".reg")
        );
        if (!DeleteFile(
            szCommandName
        ))
        {
            return HRESULT_FROM_WIN32(GetLastError());
        }
        for (int i = stringlen(szCommandName) - 1; i >= 0; i--)
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
    dwLen = stringlen(szCommandName);
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
            (4 + stringlen(TEXT(PROTOCOL_NAME)) + 9) * sizeof(TCHAR)
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
            (stringlen(szCommandName) + 1) * sizeof(TCHAR)
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
        wcscat(
            szCommandName,
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
            stringlen(reg) * sizeof(TCHAR),
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
    HRESULT hr = S_OK;
    HINSTANCE hErr;
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
    if ((hErr = ShellExecute(
        NULL,
        NULL,
        path,
        url + stringlen(TEXT(PROTOCOL_NAME)) + 3,
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
    LPCWSTR cmdLine = GetCommandLine();
    LPWSTR* argv = CommandLineToArgvW(cmdLine, &argc);
    if (argc == 2)
    {
        hr = LaunchExecutableForUrl(argv[1]);
    }
    else if (argc == 1)
    {
        hr = CheckIfInstalled();
        if (SUCCEEDED(hr) || hr == S_FALSE)
        {
            hr = InstallOrUninstall(hr);
        }
    }
    if (LocalFree(argv))
    {
        return HRESULT_FROM_WIN32(GetLastError());
    }

    ExitProcess(hr);
}
{% endhighlight %}

P.S. I think that changing the links in the document list views can be done using JSLink or something like that as well, instead of injecting JavaScript directly in the master page, yet I did not remember at the moment how it was done.
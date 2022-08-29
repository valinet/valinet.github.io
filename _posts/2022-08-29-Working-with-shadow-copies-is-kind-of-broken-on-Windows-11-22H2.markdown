---
layout: post
title:  "Working with shadow copies is kind of broken on Windows 11 22H2"
date:   2022-08-29 00:00:00 +0000
categories: 
excerpt: "As a continuation of my previous article on shadow copies in Windows, I investigate how they behave under 22621+-based builds of Windows 11 (version 22H2, yet to be released)."
---

As a continuation of my [previous article](https://valinet.ro/2022/08/28/Shadow-copies-under-Windows-10-and-11.html) on shadow copies in Windows, I investigate how they behave under 22621+-based builds of Windows 11 (version 22H2, yet to be released).

Under build 22622, the Previous Versions tab displays no snapshots:

![Previous Versions tab on 22622-based builds](https://user-images.githubusercontent.com/6503598/187235227-9ff0c900-e6bb-4195-b3a3-f7fa110d6cef.png)

Attaching with a debugger and looking on the disassembly of `twext.dll`, the component that hosts the "Previous Versions" tab, I found out that the following call to `NtFsControlFile` errors out:

```cpp
__int64 __fastcall IssueSnapshotControl(__int64 a1, _DWORD *a2, unsigned int a3)
{
  HANDLE EventW; // rsi
  int v7; // ebx
  unsigned __int64 v8; // r8
  signed int v9; // ecx
  signed int v11; // eax
  signed int LastError; // eax
  int v13; // [rsp+50h] [rbp-18h] BYREF
  wil::details::in1diag3 *retaddr; // [rsp+68h] [rbp+0h]

  memset_0(a2, 0, a3);
  EventW = CreateEventW(0i64, 1, 0, 0i64);
  if ( EventW )
  {
    v7 = NtFsControlFile(a1, EventW, 0i64, 0i64, ..., FSCTL_GET_SHADOW_COPY_DATA, NULL, 0, a2, a3);
...
```

This is called when the window tries to enumerate all snapshots in the system. The call stack looks something like this:

```cpp
CTimewarpResultsFolder::EnumSnapshots
CTimewarpResultsFolder::_AddSnapshotShellItems
SHEnumSnapshotsForPath
QuerySnapshotNames
IssueSnapshotControl
```

Of note is that during `CTimewarpResultsFolder::EnumSnapshots`, the extension also loads in items corresponding to the following, in addition to shadow copy snapshots:

* File History (`AddFileHistoryShellItems`)
* Windows 7 Backup (`AddSafeDocsShellItems`)

At first, I thought that the device call no longer supports being called with insufficient space in the buffer (the first call to this `IssueSnapshotControl` has the buffer of size `0x10`, enough for the call to populate it with information about the space it requires to contain all the data; you can read more about this device IO control call [here](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/nf-ntifs-ntfscontrolfile), [here](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb/5a43eb29-50c8-46b6-8319-e793a11f6226), and [here](https://wiki.wireshark.org/SMB2/Ioctl/Function/FILE_DEVICE_NETWORK_FILE_SYSTEM/FSCTL_GET_SHADOW_COPY_DATA)).

This being said, I quickly scratched out a project in Visual Studio where I set a large enough buffer. Unfortunately, this still did not produce the expected result. The call still said there are no snapshots available despite being offered enough space in the buffer to copy the names there.

```cpp
    HRESULT hr = S_OK;
    DWORD sz = sizeof(L"@GMT-YYYY.MM.DD-HH.MM.SS");

    HANDLE hFile = CreateFileW(L"\\\\localhost\\C$", 1, (FILE_SHARE_READ | FILE_SHARE_WRITE), NULL, (CREATE_NEW | CREATE_ALWAYS), FILE_FLAG_BACKUP_SEMANTICS, NULL);
    if (!hFile || hFile == INVALID_HANDLE_VALUE)
    {
        printf("CreateFileW: 0x%x\n", GetLastError());
        return 0;
    }
    HANDLE hEvent = CreateEventW(NULL, TRUE, FALSE, NULL);
    if (!hEvent)
    {
        printf("CreateEventW: 0x%x\n", GetLastError());
        return 0;
    }
	typedef struct _IO_STATUS_BLOCK {
		union {
			NTSTATUS Status;
			PVOID    Pointer;
		};
		ULONG_PTR Information;
	} IO_STATUS_BLOCK, * PIO_STATUS_BLOCK;
    NTSTATUS(*NtFsControlFile)(
        HANDLE           FileHandle,
        HANDLE           Event,
        PVOID  ApcRoutine,
        PVOID            ApcContext,
        PIO_STATUS_BLOCK IoStatusBlock,
        ULONG            FsControlCode,
        PVOID            InputBuffer,
        ULONG            InputBufferLength,
        PVOID            OutputBuffer,
        ULONG            OutputBufferLength
        ) = GetProcAddress(GetModuleHandleW(L"ntdll"), "NtFsControlFile");
    if (!NtFsControlFile)
    {
        printf("GetProcAddress: 0x%x\n", GetLastError());
        return 0;
    }
    NTSTATUS rv = 0;
    DWORD max_theoretical_size = sizeof(DWORD) + sizeof(DWORD) + sizeof(DWORD) + 512 * sizeof(L"@GMT-YYYY.MM.DD-HH.MM.SS") + sizeof(L"");
    char* buff = calloc(1, max_theoretical_size);
    DWORD* buff2 = buff;
    IO_STATUS_BLOCK status;
    ZeroMemory(&status, sizeof(IO_STATUS_BLOCK));
#define FSCTL_GET_SHADOW_COPY_DATA 0x144064
#define STATUS_PENDING 0x103
    rv = NtFsControlFile(hFile, hEvent, NULL, NULL, &status, FSCTL_GET_SHADOW_COPY_DATA, NULL, 0, buff, max_theoretical_size);
    if (rv == STATUS_PENDING)
    {
        WaitForSingleObject(hEvent, INFINITE);
        rv = status.Status;
    }
    if (rv)
    {
        printf("NtFsControlFile (NTSTATUS): 0x%x\n", rv);
        return 0;
    }
    printf("%d %d %d\n", buff2[0], buff2[1], buff2[2]);
```

Okay, so maybe the entire call is broken altogether. Indeed, if we craft a replacement for `NtFsControlFile` when `FsControlCode` set to `FSCTL_GET_SHADOW_COPY_DATA` that uses the Volume Shadow Service APIs instead of this device IO control call and run the program as administrator, we indeed get the snapshots. It is interesting to note that on my machine running Windows 11 22000.856 this method returned all the snapshots that both `vssadmin list shadows` and [ShadowCopyView](https://www.nirsoft.net/utils/shadow_copy_view.html) listed, while the original `NtFsControlFile` call returned less snapshots, for some reasons. I compared the returned snapshots, but couldn't find anything relevant regarding the missing ones.

```cpp
NTSTATUS MyNtFsControlFile(
    HANDLE           FileHandle,
    HANDLE           Event,
    PVOID            ApcRoutine,
    PVOID            ApcContext,
    PIO_STATUS_BLOCK IoStatusBlock,
    ULONG            FsControlCode,
    PVOID            InputBuffer,
    ULONG            InputBufferLength,
    PVOID            OutputBuffer,
    ULONG            OutputBufferLength
)
{
    NTSTATUS(*NtFsControlFile)(
        HANDLE           FileHandle,
        HANDLE           Event,
        PVOID  ApcRoutine,
        PVOID            ApcContext,
        PIO_STATUS_BLOCK IoStatusBlock,
        ULONG            FsControlCode,
        PVOID            InputBuffer,
        ULONG            InputBufferLength,
        PVOID            OutputBuffer,
        ULONG            OutputBufferLength
        ) = GetProcAddress(GetModuleHandleW(L"ntdll"), "NtFsControlFile");

    if (FsControlCode == FSCTL_GET_SHADOW_COPY_DATA)
    {
        HRESULT hr = S_OK;
        DWORD sz = sizeof(L"@GMT-YYYY.MM.DD-HH.MM.SS");
        hr = CoInitialize(NULL);

        BY_HANDLE_FILE_INFORMATION fi;
        ZeroMemory(&fi, sizeof(BY_HANDLE_FILE_INFORMATION));
        GetFileInformationByHandle(FileHandle, &fi);

        GUID zeroGuid;
        memset(&zeroGuid, 0, sizeof(zeroGuid));
        HMODULE hVssapi = LoadLibraryW(L"VssApi.dll");
        if (!hVssapi) return STATUS_INSUFFICIENT_RESOURCES;

        FARPROC CreateVssBackupComponents = GetProcAddress(hVssapi, "?CreateVssBackupComponents@@YGJPAPAVIVssBackupComponents@@@Z");
        if (!CreateVssBackupComponents) CreateVssBackupComponents = GetProcAddress(hVssapi, "?CreateVssBackupComponents@@YAJPEAPEAVIVssBackupComponents@@@Z");
        if (!CreateVssBackupComponents) return STATUS_INSUFFICIENT_RESOURCES;

        FARPROC VssFreeSnapshotProperties = GetProcAddress(hVssapi, "VssFreeSnapshotProperties");
        if (!VssFreeSnapshotProperties) return STATUS_INSUFFICIENT_RESOURCES;

        IVssBackupComponents* pVssBackupComponents = NULL;
        hr = CreateVssBackupComponents(&pVssBackupComponents);
        if (!pVssBackupComponents) return STATUS_INSUFFICIENT_RESOURCES;

        if (SUCCEEDED(hr = pVssBackupComponents->lpVtbl->InitializeForBackup(pVssBackupComponents, NULL)))
        {
            if (SUCCEEDED(hr = pVssBackupComponents->lpVtbl->SetContext(pVssBackupComponents, VSS_CTX_ALL)))
            {
                IVssEnumObject* pEnumObject = NULL;
                if (SUCCEEDED(hr = pVssBackupComponents->lpVtbl->Query(pVssBackupComponents, &zeroGuid, VSS_OBJECT_NONE, VSS_OBJECT_SNAPSHOT, &pEnumObject)))
                {
                    ULONG cnt = 0;
                    VSS_OBJECT_PROP props;
                    DWORD* data = calloc(3, sizeof(DWORD));
                    while (pEnumObject->lpVtbl->Next(pEnumObject, 1, &props, &cnt), cnt)
                    {
                        DWORD sn = 0;
                        GetVolumeInformationW(props.Obj.Snap.m_pwszOriginalVolumeName, NULL, 0, &sn, NULL, NULL, NULL, 0);
                        if (fi.dwVolumeSerialNumber != sn) continue;

                        data[0]++;
                        SYSTEMTIME SystemTime;
                        ZeroMemory(&SystemTime, sizeof(SYSTEMTIME));
                        BOOL x = FileTimeToSystemTime(&props.Obj.Snap.m_tsCreationTimestamp, &SystemTime);
                        WCHAR Buffer[MAX_PATH];
                        swprintf_s(
                            Buffer,
                            MAX_PATH,
                            L"@GMT-%4.4d.%2.2d.%2.2d-%2.2d.%2.2d.%2.2d",
                            SystemTime.wYear,
                            SystemTime.wMonth,
                            SystemTime.wDay,
                            SystemTime.wHour,
                            SystemTime.wMinute,
                            SystemTime.wSecond);
                        void* new_data = realloc(data, 3 * sizeof(DWORD) + data[0] * sz);
                        if (new_data) data = new_data;
                        else break;
                        memcpy_s((char*)data + 3 * sizeof(DWORD) + (data[0] - 1) * sz, sz, Buffer, sz);
                        VssFreeSnapshotProperties(&props.Obj.Snap);
                    }
                    void* new_data = realloc(data, 3 * sizeof(DWORD) + data[0] * sz + 2 * sizeof(WCHAR));
                    if (new_data)
                    {
                        DWORD* OutBuf = OutputBuffer;
                        data = new_data;
                        *(WCHAR*)((char*)data + 3 * sizeof(DWORD) + data[0] * sz) = 0;
                        *(WCHAR*)((char*)data + 3 * sizeof(DWORD) + data[0] * sz + sizeof(WCHAR)) = 0;
                        data[2] = data[0] * sz + 2;
                        if (OutputBufferLength < data[2] + 3 * sizeof(DWORD))
                        {
                            data[1] = 0;
                            OutBuf[0] = data[0];
                            OutBuf[1] = data[1];
                            OutBuf[2] = data[2];
                        }
                        else
                        {
                            data[1] = data[0];
                            OutBuf[0] = data[0];
                            OutBuf[1] = data[1];
                            OutBuf[2] = data[2];
                            memcpy_s(OutBuf + 3, data[2], data + 3, data[2]);
                        }
                        printf("%d %d %d\n", OutBuf[0], OutBuf[1], OutBuf[2]);
                    }
                    free(data);
                    pEnumObject->lpVtbl->Release(pEnumObject);
                }
            }
        }

        pVssBackupComponents->lpVtbl->Release(pVssBackupComponents);

        return 0;
    }
    return NtFsControlFile(FileHandle, Event, ApcRoutine, ApcContext, IoStatusBlock, FsControlCode, InputBuffer, InputBufferLength, OutputBuffer, OutputBufferLength);
}
```

In fact, if I run File Explorer under the built-in Administrator account (necessary so that calls using the VSS API work, specifically `CreateVssBackupComponents`) and replace IAT patch the call to `NtFsControlFile` with the function above (of course, I do this by modifying ExplorerPatcher), I do indeed get the snapshots to show under the "Previous versions" tab, but only if I set `ShowAllPreviousVersions` (DWORD) to `1` under `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer`:

![Previous Versions tab with ShowAllPreviousVersions](https://user-images.githubusercontent.com/6503598/187245754-3884a4c9-ef70-4f62-95a0-3405a0c7be51.png)

What is `ShowAllPreviousVersions`? It's a setting that tells the window whether to display all snapshots or filter only the ones that actually contain the file that we are querying. The check is performed in `twext.dll` in `BuildSnapshots`:

```cpp
    if ( (unsigned int)SHGetRestriction(
                         L"Software\\Microsoft\\Windows\\CurrentVersion\\Explorer",
                         0i64,
                         L"ShowAllPreviousVersions") == 1 )
```

Okay, so it seems that this solves the problem. Well, no really, since trying to open files that are clearly in the snapshot we have selected produces an error (which also seems to read from some illegal buffer, as signified by the Chinese characters, but that's a bug for some other time):

![Error when opening the file from Previous Versions tab](https://user-images.githubusercontent.com/6503598/187246098-f9c98125-f922-4c27-8c63-f11800681a25.png)

Looking a bit on the code, after the call to `QuerySnapshotNames`, there are calls to `BuildSnapshots -> TestSnapshot -> GetFileAttributesExW`. The first parameter to `GetFileAttributesExW` is a UNC-like path that references the snapshot we are looking into for the file, things like: `\\localhost\C$\@GMT-2022.08.29-14.29.52\Users\Administrator\Downloads\ep_setup.exe`.

Traditionally, we can explore the contents of a snapshot using File Explorer by visiting a path like: `\\localhost\C$\@GMT-2022.08.29-14.29.52`. Also, opening a path like `\\localhost\C$\@GMT-2022.08.29-14.29.52\Users\Administrator\Downloads\ep_setup.exe` using "Run", for example, will launch the file in the associated application, or simply execute it, if it's an executable. But under build 22622, this is also broken. For example, trying to visit the folder `\\localhost\C$\@GMT-2022.08.29-14.29.52` (by pasting the line in "Run" and hitting Enter) yields this error:

![Error when opening a VSS path](https://user-images.githubusercontent.com/6503598/187240832-64f2d036-c827-42cd-84e2-d677c05064c1.png)

At this point, I kind of stopped my investigation. It's clear that something big changed in the backend. It seems like calls that interact with the snapshots using any other means then the VSS API, so things like `NtFsControlFile` with `FSCTL_GET_SHADOW_COPY_DATA`, or `GetFileAttributesExW` on a path like `\\localhost\C$\@GMT-2022.08.29-14.29.52\Users\Administrator\Downloads\ep_setup.exe` somehow get denied along the way. This may stem from the wish to close off programatic access to snapshot information to applications that run under a low priviledge context, as the VSS API only works when the app runs as an administrator. If that's the case, it's not necessarily bad, but it seems it also broke a lot of stuff which really should be fixed. The thing is, it's ind of expected for File Explorer to run under a standard user account and have the users use it to view and restore previous versions of their files originating from shadow copies. There's been a recent vulnerability related to VSS ([here](https://support.microsoft.com/en-us/topic/kb5015527-shadow-copy-operations-using-vss-on-remote-smb-shares-denied-access-after-installing-windows-update-dated-june-14-2022-6d460245-08b6-40f4-9ded-dd030b27850b)), maybe the fix for this had something to do with this change in behavior?

I would really appreciate an official resolution on the matter, as things are pretty broken at the moment, unfortunately. I really like the functionality offered by "Previous versions" powered by shadow copies and would like to see it continue to work in the traditional UI offered with past Windows 10 and 11 releases.
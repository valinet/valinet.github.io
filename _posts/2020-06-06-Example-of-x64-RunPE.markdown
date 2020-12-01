---
layout: post
title:  "Example of x64 RunPE"
date:   2020-09-06 00:00:00 +0200
categories: 
excerpt: Shows how to embed the code of another executable in a binary and execute it at runtime directly from memory (without extracting to disk etc). Deinitely a classic, guaranteed to work on amd64.
---

Code available at: <https://gist.github.com/valinet/e27e64927db330b808c3a714c5165b0a>.

Shows how to embed the code of another executable in a binary and execute it at runtime directly from memory (without extracting to disk etc). Deinitely a classic, guaranteed to work on amd64. The generic name for this kind of technique is called *RunPE* (run portable executable).

Compile using (make sure to use x64 cl):

```
cl runpe64.cpp
````
    
It should generate runpe64.exe.

Licensed under MIT.
---
layout: post
title:  "Convert files to PDF using Puppeteer (Chromium)"
date:   2020-09-06 00:00:00 +0200
categories: 
excerpt: This example shows how to convert various file types to pdf using the built-in 'Save to PDF' feature from Chromium. This is needlessly too complicated because Google decided to crap out the command line of the browser.
---

Code available at: <https://gist.github.com/valinet/8e9d3ca463003c8486b08856a0bc03fc>.

This example shows how to convert various file types to pdf using the built-in 'Save to PDF' feature from Chromium. This is needlessly too complicated because Google decided to crap out the command line of the browser. 

For my usecase, I converted a bunch of SVGX files (which represented pages from a book) to individual PDF files which in the end I merged into a single PDF file.

To install required dependencies, on Ubuntu run:
```
sudo apt install npm libatk-bridge2.0-0 libgbm1 libxss1 libgtk-3-0 default-jre
npm install puppeteer
npm install easy-pdf-merge
```

To run this unmodified example, execute the following:
```
node download.js 2 book.pdf
```

This will look for p1.svgx, and p2.svgx, convert them to 1.pdf, and 2.pdf respectively, and merge those two PDFs into a single one called book.pdf. Customize according to your needs, this is just a proof of concept.

Licensed under MIT.
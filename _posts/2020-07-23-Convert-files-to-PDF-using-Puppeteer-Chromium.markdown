---
layout: post
title:  "Convert files to PDF using Puppeteer (Chromium)"
date:   2020-09-06 00:00:00 +0200
categories: 
excerpt: This example shows how to convert various file types to pdf using the built-in 'Save to PDF' feature from Chromium. This is needlessly too complicated because Google decided to crap out the command line of the browser.
---

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

{% highlight javascript%}
/*
Convert files to PDF using Puppeteer (Chromium)
Copyright (c) 2020 Valentin-Gabriel Radu
Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
*/

const fs  = require('fs');
const puppeteer = require("puppeteer");
const merge = require('easy-pdf-merge');

// based on https://stackoverflow.com/questions/48966246/how-to-hide-margins-in-headless-chrome-generated-pdf
(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  var filenames = [];
  for (var i = 1; i <= process.argv[2]; i++) {
    const image = 'data:image/svg+xml;base64,' + base64_encode(`p${i}.svgz`);
    const test_html = `<html><head><style type="text/css">body { margin:0; }</style></head><body><img src="${image}"></body></html>`;
    await page.goto(`data:text/html,${test_html}`, { waitUntil: 'networkidle0' });
    const filename = `${i}.pdf`;
    filenames.push(filename);
    await page.pdf({
      path: filename,
      format: "A4",
      printBackground: false,
      displayHeaderFooter: false,
      margin: {
        left: 0,
        top: 0,
        right: 0,
        bottom: 0
      }
    });
    console.log("Processed: " + i);
  }

  await browser.close();

  await mergeMultiplePDF(filenames);
})();

// as suggested at https://github.com/puppeteer/puppeteer/issues/1643
function base64_encode(file) {
  var bitmap = fs.readFileSync(file);
  return new Buffer(bitmap).toString('base64');
}

// taken from https://stackoverflow.com/questions/48510210/puppeteer-generate-pdf-from-multiple-htmls
const mergeMultiplePDF = (pdfFiles) => {
    return new Promise((resolve, reject) => {
        merge(pdfFiles,process.argv[3],function(err){
            if(err){
                console.log(err);
                reject(err)
            }
            console.log('Saved as: ' + process.argv[3]);
            resolve()
        });
    });
};
{% endhighlight %}


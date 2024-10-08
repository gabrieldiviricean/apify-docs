---
title: Downloading files
description: Learn how to automatically download and save files to the disk using two of the most popular web automation libraries, Puppeteer and Playwright.
sidebar_position: 3
slug: /puppeteer-playwright/common-use-cases/downloading-files
---

# Downloading files

**Learn how to automatically download and save files to the disk using two of the most popular web automation libraries, Puppeteer and Playwright.**

---

Downloading a file using Puppeteer can be tricky. On some systems, there can be issues with the usual file saving process that prevent you from doing it in a straightforward way. However, there are different techniques that work (most of the time).

These techniques are only necessary when we don't have a direct file link, which is usually the case when the file being downloaded is based on more complicated data export.

## Setting up a download path {#setting-up-a-download-path}

Let's start with the easiest technique. This method tells the browser in what folder we want to download a file from Puppeteer after clicking on it.

```js
const client = await page.target().createCDPSession();
await client.send('Page.setDownloadBehavior', { behavior: 'allow', downloadPath: './my-downloads' });
```

We use the mysterious `client` API which gives us access to all the functions of the underlying [Chrome DevTools Protocol](https://pptr.dev/api/puppeteer.cdpsession) (Puppeteer & Playwright are built on top of it). Basically, it extends Puppeteer's functionality. Then we can download the file by clicking on the button.

```js
await page.click('.export-button');
```

Let's wait for one minute. In a real use case, you want to check the state of the file in the file system.

```js
await page.waitFor(60000);
```

To extract the file from the file system into memory, we have to first find its name, and then we can read it.

```js
import fs from 'fs';

const fileNames = fs.readdirSync('./my-downloads');

// Let's pick the first one
const fileData = fs.readFileSync(`./my-downloads/${fileNames[0]}`);

// ...Now we can do whatever we want with the data
```

## Intercepting and replicating a file download request {#intercepting-a-file-download-request}

For this second option, we can trigger the file download, intercept the request going out, and then replicate it to get the actual data. First, we need to enable request interception. This is done using the following line of code:

```js
await page.setRequestInterception(true);
```

Next, we need to trigger the actual file export. We might need to fill in some form, select an exported file type, etc. In the end, it will look something like this:

```js
await page.click('.export-button');
```

We don't need to await this promise since we'll be waiting for the result of this action anyway (the triggered request).

The crucial part is intercepting the request that would result in downloading the file. Since the interception is already enabled, we just need to wait for the request to be sent.

```js
const xRequest = await new Promise((resolve) => {
    page.on('request', (interceptedRequest) => {
        interceptedRequest.abort(); // stop intercepting requests
        resolve(interceptedRequest);
    });
});
```

The last thing is to convert the intercepted Puppeteer request into a request-promise options object. We need to have the `request-promise` package installed.

```js
import request from 'request-promise';
```

Since the request interception does not include cookies, we need to add them subsequently.

```js
const options = {
    encoding: null,
    method: xRequest._method,
    uri: xRequest._url,
    body: xRequest._postData,
    headers: xRequest._headers,
};

// Add the cookies
const cookies = await page.cookies();
options.headers.Cookie = cookies.map((ck) => `${ck.name}=${ck.value}`).join(';');

// Resend the request
const response = await request(options);
```

Now, the response contains the binary data of the downloaded file. It can be saved to the disk, uploaded somewhere, or [submitted with another form](./submitting_a_form_with_a_file_attachment.md).

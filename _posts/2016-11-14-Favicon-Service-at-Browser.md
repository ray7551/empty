---
published: true
layout: default
title: Favicon service at browser
keywords: favicon
tags: js favicon
---

## Some favicon services for free
1. yandex
http://favicon.yandex.net/favicon/google.com/yandex.ru/codepen.io/douban.com/readfree.me/dianying.fm
2. google
https://www.google.com/s2/favicons?domain=vdisk.weibo.com
https://s2.googleusercontent.com/s2/favicons?domain=css-tricks.com

And I found some other service may not so reliable, such as
[api.byi.pw/favicon](http://api.byi.pw/favicon),
and [The Favicon Finder](https://icons.better-idea.org/).
Like yandex and google favicon service, api.byi.pw/favicon will grab favicon, and store the favicon at their own server, support http and https.
The Favicon Finder support multiple size of favicon, but it actually give you a *redirect to origin favicon url*. The Favicon Finder is [open source](https://github.com/mat/besticon), you can deploy it at your own server.

I'd like to use yandex favicon server, although it seems like only return 16x16 favicons, but it can return multiple favicon at one time!

## Favicon Service at browser
The question come from my chrome extension, which need to get favicon by website url.

```html
<a class="icon-link" data-url="https://www.google.com" href="#">Google</a>
<a class="icon-link" data-url="https://www.baidu.com" href="#">Baidu</a>
```

Here i have to display favicon as link's background image:

```js
var $iconLinks = document.querySelectorAll('.icon-link');
$iconLinks.forEach(function($link){
  var url = new URL($link.getAttribute('data-url'));
  var faviconUrl = '//favicon.yandex.net/favicon/' + url.host
  $link.style.backgroundImage = "url('" + faviconUrl + "')";
});

```

## Use localStorage as cache
Let's store the favicon to the localStorage.
### 1. Get Blob object of image
```js
function fetchImgBlob(url) {
  return fetch(url).then(function(r) {
    return r.blob();
  });
}
```
### 2. Transform Bolb object to dataURI
Using [FileReader.readAsDataURL()](https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsDataURL), we can transform a Bolb object to dataURI:
```js
function blobToDataURI(blob) {
  return new Promise(function(resolve, reject) {
    var a = new FileReader();
    a.onload = function(e) {
      resolve(e.target.result);
    };
    a.onerror = function(e) {
      reject(e);
    };
    a.readAsDataURL(blob);
  });
}
```
I return a new Promise to make it thenable like `fetchImgBlob`.

### 3. Save the dataURI to localStorage
```js
var faviconUrl = '//favicon.yandex.net/favicon/' + url.host
fetchImgBlob(faviconUrl).then(function(blob) {
  return blobToDataURI(blob);
}).then(function(dataURI) {
  $img.src = dataURI;
  $dataURI.textContent = dataURI;
  localStorage.setItem(url.host, dataURI); // use host as key instead of the whole url
}).catch(function(e){
  console.log('Error: ' + e.toString());
});
```

### Demo
<a class="jsbin-embed" href="https://jsbin.com/hayoku/embed?js,output">JS Bin on jsbin.com</a><script src="https://static.jsbin.com/js/embed.min.js?3.40.2"></script>

## A downgrade solution in chrome extension
As a downgrade solution, we can use chrome favicon cache. Codes below based on chrome://resources/js/icon.js

```js
var FAVICON_URL_REGEX = /\.ico$/i;
function getChromeIcon(url, size, type) {
  size = size || 16;
  type = type || 'favicon';

  return 'chrome://' + type + '/size/' + size + '@1x/' +
    // Note: Literal 'iconurl' must match FAVICON_URL_REGEX
    (FAVICON_URL_REGEX.test(url.href) ? 'iconurl/' + url.href : url.origin);
}
```


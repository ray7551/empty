---
published: false
layout: default
title: <HTML5 Game Development> Notes
keywords: HTML5 game
tags: notes html5 game
---

## Canvas
### draw a image
```js
ctx = canvas.getContext('2d');
var img = new Image();
img.onload = onImageLoad;
img.src = "/media/img/gamedev/ralphyrobot.png";
img.onload = function(){
  ctx.drawImage(this, 0, 0);
}
```

### 24 fps movie
```js
var fps = 24;
var ctx = canvas.getContext('2d');
var images = [...];

window.setInterval(function() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.drawImage(images[i]);
}, 1000/fps);
```
## Atlases
### Browser Connections
Max Number of default simultaneous persistent connections per server/proxy:
 
Firefox 2:  2
Firefox 3+: 6
Opera 9.26: 4
Opera 12:   6
Safari 3:   4
Safari 5:   6
IE 7:       2
IE 8:       6
IE 10:      8
Chrome:     6
The limit is per-server/proxy

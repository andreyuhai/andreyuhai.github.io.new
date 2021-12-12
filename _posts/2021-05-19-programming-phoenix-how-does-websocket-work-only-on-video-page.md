---
title: Programming Phoenix 1.4 | How do the websockets connect only on the video page
tags: [elixir, phoenix, websockets]
categories: [TIL, Elixir]
---

In the book Programming Phoenix 1.4, we are creating a web application where users can annotate YouTube videos and annotations get brodcast by WebSockets to all the users that are on the same video page. We can also see that the browser tries to connect to a socket only on the video page, namely `rumbl_web/templates/watch/show.html.eex`. The chapter were this topic is covered in the book is the chapter 10. I was trying to understand how we are connecting to a socket only on the video page and here is how it happens.

If you take a look at `assets/js/app.js`, we can see that we call
```javascript
Video.init(socket, document.getElementById("video"))
```

If you inspect where `Video.init` is defined, which is `assets/js/video.js`, you can see the snippet below,
```javascript
let Video = {
  init(socket, element) { if(!element) { return }
    let playerId = element.getAttribute("data-player-id")
    let videoId  = element.getAttribute("data-id")
    socket.connect()
    Player.init(element.id, playerId, () => {
      this.onReady(videoId, socket)
    })
  },
```

What the `Video.init` function does is, if the `element` doesn't exist on the page we directly return without trying to connecting to the socket. However if you basically insert a `div` with an id `video` on any page, you can see that the browser will try to connect to a websocket just because we have the video element on the page.

In `templates/watch/show.html.eex` you can see the snippet below which basically creates an element with an id of `video`, hence, the browser tries to connect to a socket on the video page.
```elixir
<%= content_tag :div, id: "video", data: [id: @video.id, player_id: player_id(@video)] do %>
<% end %>
```


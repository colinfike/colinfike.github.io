---
layout: post
title:  "YouTube and SoundCloud Widget APIs"
date:   2017-03-23 17:58:33 -0400
categories: soundcloud youtube rails
---
I wasn’t really sure what to start with as my first blog post but I decided that I’d skip some sort of “get to know me” post especially considering that I don’t know if anyone will ever lay eyes on this besides me. I decided instead to just jump right into it with some YouTube/SoundCloud related work that I ran into while working on [Ploosic][ploosic-site] (I’m a programmer, not a professional name generator) which is a neat little app to let you play tracks from SoundCloud and YouTube on a single playlist.

The idea was to load videos/songs into the YouTube and SoundCloud embedded players and try and pull some sort of tomfoolery to enable me to load songs in as necessary and start/stop the players as you go. I wasn’t really expecting both sites to have publicly available Widget APIs that you can use to load/queue up songs, control the widget, and listen for events that the widgets throw when a song is loaded/starts/ends.

I was a tiny bit disappointed that I didn’t have to pull tomfoolery but was happy to see that what I wanted to do was possible and thought it was a good opportunity to work on my coffeescript and check out the [module pattern][thoughtbot]. A friend recently learned this and recommended it highly to me so I thought I'd see what the big deal was. I won't be focusing on the module pattern in this post much at all, the link above goes over it very well on the thoughtbot blog.

My main idea was to have a “main player” module that could manage starting/stopping the individual players, the state of the player/playlist, and everything like that. I would need to make individual modules as well for each of the players for the main player module to use. That ended up working pretty well since both of the widgets allowed me to load tracks as well as start and stop. Initialization and use of the widgets is fairly straightforward.

{% highlight coffescript %}
# ...
youtubePlayer = (->
  youtubeWidget = undefined

  setWidget: ->
    youtubeWidget = new (YT.Player)('youtube-player',
      height: '0'
      width: '0'
      videoId: ''
      events:
        'onReady': onPlayerReady
        'onStateChange': onPlayerStateChange
    )
    return

  start: ->
    youtubeWidget.loadVideoById getYoutubeId(playlist[trackIndex].url)

  resume: ->
    youtubeWidget.playVideo()

  pause: ->
    youtubeWidget.pauseVideo()
)()

# This provided a singleton rudimentary YouTube player that I could use in
# the main player

{% endhighlight %}


{% highlight coffescript %}

# SoundCloud was even easier
soundcloudPlayer = (->
  soundcloudWidget = SC.Widget('soundcloud-player')

  start: ->
    soundcloudWidget.load playlist[trackIndex].url, auto_play: true

  resume: ->
    soundcloudWidget.play()

  pause: ->
    soundcloudWidget.pause()

# ...

# Same singleton functionality for the soundCloudPlayer that I could use in the
# main player
{% endhighlight %}


An issue that I ran into was that if you paused during video load or skipped to the next song while a player was loading, it was possible for that player to start up when the video was done loading even though you told it to pause. This was easily fixed by having event listeners check if the widget is supposed to be playing when it throws an event that states the track is done loading and starting to play.

{% highlight coffescript %}

# This actually handles the problem where the player shouldn't be playing
# and the situation where the player should go to the next song when the song
# playing in the widget ends.

onPlayerStateChange = (event) ->
  console.log 'onPlayerStateChange'
  if (event.data == 1) and ((playlist[trackIndex].site_id == 1) or !songPlaying)
    youtubeWidget.pauseVideo()
  else if event.data == 0
    PlayerController.mainPlayer.next()


# Handles edge cases where the player can start up after it's been paused
soundcloudWidget.bind SC.Widget.Events.PLAY, ->
  if (playlist[trackIndex].site_id == 2) or !songPlaying
    soundcloudWidget.pause()

{% endhighlight %}

The onPlayerStateChange function is called whenever the state of the player changes (an astute observation!) and pauses the video if it ends up the YouTube widget is playing even though the player should be paused or the soundcloud widget should be playing instead.

The SoundCloud widget requires that you bind a function to be called when a certain even is called, in this case the PLAY event, and pauses it accordingly.

You can check out the [Ploosic repo][ploosic-repo] if you want to see how I pulled everything together in more detail. Check out the playlist.coffee file for the meat of the player functionality.

[ploosic-site]: https://ploosic.com
[ploosic-repo]: https://github.com/colinfike/ploosic
[thoughtbot]: https://robots.thoughtbot.com/module-pattern-in-javascript-and-coffeescript
[youtube-widget]: https://developers.google.com/youtube/iframe_api_reference
[soundcloud-widget]: https://developers.soundcloud.com/docs/api/html5-widget

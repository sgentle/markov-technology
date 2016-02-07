Markov Technology
=================

This is a demo showing the relationship between Markov chains and music. We
use Tricka Technology by A. Skillz & Kraft Kuts because it has the large- and
small-scale self-similarity that makes markov generation work best.


    $ = document.querySelector.bind(document)

    context = new (window.AudioContext || window.webkitAudioContext)

Audio loading
-------------

This loads audio data from the server, caching the audio buffer in memory for future plays.

    buffercache = {}
    maybeFetch = (src) ->
      if buffercache[src]
        Promise.resolve(buffercache[src])
      else
        fetch(src)
          .then (response) -> response.arrayBuffer()
          .then (audioData) -> new Promise (accept) -> context.decodeAudioData audioData, accept
          .then (buffer) -> buffercache[src] = buffer; buffer

    play = (src, at) ->
      maybeFetch src
        .then (buffer) ->
          node = context.createBufferSource()
          node.buffer = buffer
          node.connect context.destination
          if at then node.start(at) else node.start()
          node

Audio Markov node
-----------------

Our audio is played by slowly traversing a weighted directed graph of Markov
nodes. The nodes handle playing audio, and have an internal list of outgoing
links that they randomly choose from.

In order to achieve seamless playback, we have to schedule playing the next
node ahead of when our node actually finishes. We use a combination of
`setTimeout` and the Web Audio API's scheduler to do this.

    nodeProto =
      display: ->
        total = 0
        total += l.weight for l in @links
        "#{@name} ⇒ " + ("#{l.next?.name} (#{Math.round(l.weight/total*100)}%)" for l in @links).join ', '

      preload: ->
        console.log "Loading", @src
        maybeFetch @src

      play: (at=context.currentTime+0.5) ->
        console.log "Playing", @src, "at", at
        setTimeout =>
          $('#log').innerHTML += @display() + '<br>'
          window.scrollTo(0, document.body.scrollHeight)
        , (at-context.currentTime)*1000
        play @src, at
        .then (audio) =>
          time = audio.buffer.duration * 1000 - 500 # 500ms grace period
          scheduled = at + audio.buffer.duration
          next = @getNext()
          if next
            next.preload()
            setTimeout =>
              console.log "Timer fired for next load"
              next.play(scheduled)
            ,time
          audio.onended = @ended.bind(this)

      ended: ->
        console.log "Finished", @src

      getNext: ->
        total = @links.reduce ((total, link) -> total + link.weight), 0
        rand = Math.random()*total
        cur = 0
        for link in @links
          cur += link.weight
          return link.next if rand <= cur
        null

      link: (nexts, weight=1) ->
        nexts = [nexts] unless Array.isArray(nexts)
        @links.push {next, weight} for next in nexts
        this

    makeNode = (src) ->
      o = Object.create nodeProto
      o.src = src
      o.links = []
      o.name = src.split('/').pop().split('.')[0]
      o

Setup
-----

We create nodes for every media file we're using. The nodes are not preloaded,
which probably means it will stutter if the network connection is too slow and
cache is cold.

    n = {}
    n[k] = makeNode "media/#{k}.mp3" for k in [
      'intro', 'intro2'
      'tricka_1_intro', 'tricka_1_intro2'
      'tricka_1_plain', 'tricka_1_cmon', 'tricka_1_uh', 'tricka_1_trickawhat',
      'tricka_1_tricka_tricka', 'tricka_1_putemup'
      'tricka_2_plain', 'tricka_2_uh', 'tricka_2_yeah', 'tricka_2_tricka', 'tricka_2_putemup'
      'tricka_2_outro_tricka_cmon', 'tricka_2_outro_what_what', 'tricka_2_outro_putemup'

      'instr1_1_intro', 'instr1_1_plain',
      'instr1_2_plain', 'instr1_2_outro'

      'instr2_1_intro', 'instr2_1', 'instr2_1_scratch',
      'instr2_2_1', 'instr2_2_2', 'instr2_2_outro'

      'bridge1_1', 'bridge1_2'

      'hipstep1_1', 'hipstep1_2', 'hipstep1_2_outro'
      'hipstep2_1_intro', 'hipstep2_1_guitar', 'hipstep2_2', 'hipstep2_2_guitar_outro'

      'putemup_1', 'putemup_1_intro', 'putemup_2', 'putemup_2_outro'

      'bounce_1_intro', 'bounce_1_uh', 'bounce_1_yeah'
      'bounce_2_uh', 'bounce_2_yeah', 'bounce_2_outro'

      'youknowwhat_1', 'youknowwhat_2', 'youknowwhat_3', 'youknowwhat_outro'
    ]

Next we add each link. There are many links.

    addLinks = ->
      @intro.link @tricka_1_intro
      @intro2.link @tricka_1_intro2

      @tricka_1_intro.link @tricka_2_plain
      @tricka_1_intro2.link @tricka_2_plain

      for k1 in ['plain', 'cmon', 'uh', 'trickawhat', 'putemup']
        for k2 in ['plain', 'uh', 'yeah', 'tricka', 'putemup']
          this["tricka_1_#{k1}"].link this["tricka_2_#{k2}"] unless k1 is 'putemup' or k2 is 'putemup'
          this["tricka_2_#{k2}"].link this["tricka_1_#{k1}"]
        this["tricka_1_#{k1}"].link [@tricka_2_outro_tricka_cmon, @tricka_2_outro_what_what] unless k1 is 'putemup'

      @tricka_1_putemup.link [@tricka_2_putemup, @tricka_2_outro_putemup]

      for k in ['tricka_cmon', 'putemup', 'what_what']
        this["tricka_2_outro_#{k}"].link [@instr1_1_intro, @instr2_1_intro, @bounce_1_intro]

      @instr1_1_intro.link @instr1_2_plain
      @instr1_2_plain.link [@instr1_1_plain, @instr2_1]
      @instr1_1_plain.link [@instr1_2_plain, @instr1_2_outro]
      @instr1_2_outro.link [@bridge1_1, @intro2]

      @instr2_1_intro.link [@instr2_2_1, @instr2_2_2]
      @instr2_1.link [@instr2_2_1, @instr2_2_2]
      @instr2_2_1.link [@instr2_1, @instr2_1_scratch]
      @instr2_2_2.link [@instr2_1, @instr2_1_scratch]
      @instr2_1_scratch.link @instr2_2_outro
      @instr2_2_outro.link [@hipstep1_1, @hipstep2_1_intro]

      @bridge1_1.link @bridge1_2
      @bridge1_2.link @hipstep1_1

      @hipstep1_1.link @hipstep1_2
      @hipstep1_2.link [@hipstep1_1, @hipstep2_1_guitar]

      @hipstep2_1_intro.link @hipstep2_2
      @hipstep2_2.link @hipstep2_1_guitar
      @hipstep2_1_guitar.link(@hipstep2_2).link(@hipstep2_2_guitar_outro, 2)
      @hipstep2_2_guitar_outro.link @putemup_1_intro

      @putemup_1_intro.link @putemup_2
      @putemup_2.link @putemup_1
      @putemup_1.link [@putemup_2, @putemup_2_outro]

      @putemup_2_outro.link @tricka_1_intro

      @bounce_1_intro.link [@bounce_2_yeah, @bounce_2_uh]
      @bounce_1_yeah.link([@bounce_2_yeah, @bounce_2_uh]).link(@bounce_2_outro, 2)
      @bounce_1_uh.link([@bounce_2_yeah, @bounce_2_uh]).link(@bounce_2_outro, 2)
      @bounce_2_yeah.link @bounce_1_yeah
      @bounce_2_uh.link @bounce_1_uh
      @bounce_2_outro.link @youknowwhat_1

      for k1 in [1, 2, 3]
        for k2 in [1, 2, 3] when k1 != k2
          this["youknowwhat_#{k1}"].link this["youknowwhat_#{k2}"]
        this["youknowwhat_#{k1}"].link @youknowwhat_outro

      @youknowwhat_outro.link @tricka_1_intro2

    addLinks.apply(n)

And, finally, start the thing!

    n.intro.play()

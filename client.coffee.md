Markov Technology
=================

This is a demo showing the relationship between Markov chains and music. We
use Tricka Technology by A. Skillz & Kraft Kuts because it has the large- and
small-scale self-similarity that makes markov generation work best.


    $ = document.querySelector.bind(document)

    context = new (window.AudioContext || window.webkitAudioContext)
    console.log context.sampleRate


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
          node.connect groupPulser.node
          if at then node.start(at, 0, buffer.duration) else node.start(context.currentTime, 0, buffer.duration)
          node


Lag detector
------------

We do a few different graphical things that can stress out mobile devices and
lesser browsers. In case it's too much, we can turn off some things using this
lag detector.

    massiveLagCallbacks = []
    onMassiveLag = (f) -> massiveLagCallbacks.push f
    watchFrameDrop = ->
      framesDropped = 0
      oldt = 0
      frameTime = 1/30 * 1000
      checkFrameDrop = -> requestAnimationFrame (t) ->
        if t - oldt > frameTime
          framesDropped += Math.min(Math.round((t - oldt) / frameTime), 10)
          console.log "framedrop", framesDropped, t - oldt
        else if framesDropped > 0
          framesDropped -= 0.25

        if framesDropped > 100
          f() for f in massiveLagCallbacks
          return

        oldt = t
        checkFrameDrop()
      checkFrameDrop()
    watchFrameDrop()


Pulser
------

We make the current node pulse using a Web Audio Analyser Node hooked up to
some JS to set the opacity of the node's pulse element.

    makePulser = (opts={}) ->
      node = context.createAnalyser()
      node.fftSize = opts.fftSize or 256
      ary = new Float32Array(node.fftSize)
      min = null
      max = null
      oldavg = null
      attachedEl = null
      drawing = false
      decay = opts.decay ? 0.0001
      attack = opts.attack ? 0.5
      smoothing = opts.smoothing ? 0.66
      oldt = null
      stopped = false
      draw = -> requestAnimationFrame (t) ->
        return if stopped

        if node.getFloatTimeDomainData
          node.getFloatTimeDomainData(ary)
        else
          ary[0] = 1

        avg = 0
        avg += Math.abs(val) for val in ary
        avg = avg * (1-smoothing) + oldavg * smoothing
        oldavg = avg

        if !oldavg
          min = avg
          max = avg
          oldavg = avg
        min = min * (1-attack) + avg * attack if avg < min
        max = max * (1-attack) + avg * attack if avg > max

        val = Math.round(Math.min((avg-min)/(max-min), 1)*1000)/1000

        newmax = max * (1-decay) + min * decay
        newmin = min * (1-decay) + max * decay
        max = newmax
        min = newmin

        attachedEl.style.opacity = if opts.invert then 1-val else val
        draw()

      attach = (el) ->
        attachedEl.style.opacity = 0 if attachedEl
        attachedEl = el
        draw() unless drawing
        drawing = true

      stop = -> stopped = true

      {node, attach, stop}

    pulser = makePulser()
    groupPulser = makePulser(fftsize: 2048, attack: 1, smoothing: 0.96, invert: true)
    pulser.node.connect context.destination
    groupPulser.node.connect pulser.node

    onMassiveLag -> groupPulser.stop()


SVG
---

This sets up our SVG document, some helpers for creating new SVG elements, and
the top-level groups for different elements.

    svg = document.querySelector('svg')

    svgEl = (name, attribs={}) ->
      el = document.createElementNS 'http://www.w3.org/2000/svg', name
      el.setAttribute k, v for k, v of attribs
      el

    svgText = (text, attribs) ->
      el = svgEl 'text', attribs
      el.textContent = text
      el

    defs = svgEl 'defs'
    defs.innerHTML = """
      <marker id="markerArrow"
              viewBox="0 0 10 10"
              refX="1" refY="5"
              markerWidth="5"
              markerHeight="5"
              orient="auto">
          <path d="M 0 0 L 10 5 L 0 10 z" />
      </marker>
    """
    svg.appendChild defs

    labelGroup = svgEl 'g',
      "font-size": 96
      "text-anchor": "middle"
    linkGroup = svgEl 'g',
      "opacity": 0.8
    nodeGroup = svgEl 'g'
    shadeGroup = svgEl 'g'
    svg.appendChild shadeGroup
    svg.appendChild labelGroup
    svg.appendChild linkGroup
    svg.appendChild nodeGroup

    labelGroup.style.fontFamily = '"Helvetica Neue", "Helvetica", Sans Serif'
    labelGroup.style.fontWeight = "600"

Node group
----------

A node group is a group of Markov nodes, which is used to set the fill, stroke
and pulse colours for nodes in the subgroup, as well as add the fuzzy coloured
background shadow thing.

    groups = {}

    addGroup = (name, group) ->
      groups[name] = group

      group.el = el = svgEl 'g',
        fill: group.fill
        stroke: group.stroke

      group.pulseEl = pulseEl = svgEl 'g',
        opacity: 0
        stroke: group.stroke
        fill: group.pulse

      labelGroup.appendChild el
      labelGroup.appendChild pulseEl

      for t, i in group.label.text
        text = svgText t,
          x: group.label.x
          y: group.label.y + i * 96
        el.appendChild text
        pulseEl.appendChild text.cloneNode(true)


      if group.shade
        grad = svgEl 'radialGradient', id: "#{name}-gradient"
        grad.appendChild svgEl 'stop', offset: "0%", "stop-opacity": 1, "stop-color": group.shade.fill
        grad.appendChild svgEl 'stop', offset: "50%", "stop-opacity": 1, "stop-color": group.shade.fill
        grad.appendChild svgEl 'stop', offset: "100%", "stop-opacity": 0, "stop-color": group.shade.fill
        defs.appendChild grad

        shade = svgEl 'circle',
          cx: group.shade.x
          cy: group.shade.y
          r: group.shade.r * 1.5
          fill: "url(##{name}-gradient)"
          opacity: 0.8

        rate = 2.3529
        start = (Math.random() * -2 * rate).toFixed(4)
        shade.style.animation = "#{rate}s ease-in-out #{start}s infinite alternate pulse"

        onMassiveLag -> shade.style.animation = ""

        shadeGroup.appendChild shade


Markov node
-----------

Our audio is played by slowly traversing a weighted directed graph of Markov
nodes. The nodes handle playing audio, and have an internal list of outgoing
links that they randomly choose from.

In order to achieve seamless playback, we have to schedule playing the next
node ahead of when our node actually finishes. We use a combination of
`setTimeout` and the Web Audio API's scheduler to do this.

    nodeProto =
      render: ->
        return if @el
        @el = el = svgEl 'g',
          opacity: 0.7
        x = @x
        y = @y

        circle = svgEl 'circle',
          cx: x
          cy: y
          r: 40
          fill: @group.fill
          stroke: @group.stroke

        @pulse = pulse = svgEl 'circle',
          cx: x
          cy: y
          r: 40
          fill: @group.pulse
          stroke: @group.stroke
          opacity: 0

        nameparts = @name.split '_'
        topname = nameparts.slice(0,2).join '_'
        botname = nameparts.slice(2).join '_'

        if @label
          label = svgText @label,
            x: x
            y: y + 6
            "text-anchor": "middle"
            "font-size": 16
            "font-family": "Helvetica Neue"
            "font-weight": "800"

        el.appendChild circle
        el.appendChild pulse
        el.appendChild label if label
        nodeGroup.appendChild el

      renderLink: (next) ->
        return unless @el and next.el
        @linkEls ||= {}
        if !@linkEl
          @linkEl = svgEl 'g',
            stroke: '#111'
          linkGroup.appendChild @linkEl

        m = (next.y - @y) / (next.x - @x)
        d = Math.sqrt((next.y - @y)**2 + (next.x - @x)**2)
        x1 = @x + (next.x - @x) * (40/d)
        y1 = @y + (next.y - @y) * (40/d)
        x2 = next.x - (next.x - @x) * (48/d)
        y2 = next.y - (next.y - @y) * (48/d)

        line = svgEl 'line',
          x1: x1, y1: y1
          x2: x2, y2: y2
          'stroke-width': 2
          'marker-end': 'url(#markerArrow)'
        @linkEl.appendChild line
        @linkEls[next.name] = line


      display: ->
        total = 0
        total += l.weight for l in @links
        "#{@name} ⇒ " + ("#{l.next?.name} (#{Math.round(l.weight/total*100)}%)" for l in @links).join ', '

      activate: ->
        console.log @display()
        @el.setAttribute 'opacity', 0.8
        pulser.attach @pulse
        groupPulser.attach @group.pulseEl if @group.pulseEl

      deactivate: ->
        @el.setAttribute 'opacity', 0.7
        linkEl.setAttribute 'stroke', '#111' for k, linkEl of @linkEls if @linkEls

      preload: ->
        console.log "Loading", @src
        maybeFetch @src

      play: (at=context.currentTime+0.1, last) ->
        console.log "Playing", @src, "at", at

        setTimeout =>
          @activate()
          last?.deactivate()
        , (at-context.currentTime)*1000

        play @src, at
        .then (audio) =>
          scheduled = at + audio.buffer.duration
          time = (at - context.currentTime) * 1000 + 1000 # 500ms grace period
          next = @getNext()
          if next
            next.preload()
            setTimeout =>
              @linkEls[next.name].setAttribute 'stroke', '#FF5252'
              next.play(scheduled, this)
            , time
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
        for next in nexts
          linkEl = @renderLink next
          @links.push {next, weight}
        this

    makeNode = (src, data) ->
      o = Object.create nodeProto
      o.src = src
      o.links = []
      o.name = src.split('/').pop().split('.')[0]
      o[k] = data[k] for k in 'x y label'.split(' ')
      if data.group and groups[data.group]
        o.group = groups[data.group]
      else
        o.group = {fill: '#DDD', stroke: '#111', pulse: '#FFF'}
      o.render()
      o

Setup
-----

We create nodes for every media file we're using. The nodes are not preloaded,
which probably means it will stutter if the network connection is too slow and
the cache is cold.

Firefox has issues with inaccurate mp3 buffer length, so we need to format
sniff, use ogg, and fall back to mp3 if it's not available.

    canOgg = document.createElement('audio').canPlayType('audio/ogg')
    ext = if canOgg then 'ogg' else 'mp3'

    nodes = {}
    fetch('data.json')
    .then (result) -> result.json()
    .then (data) ->
      for groupName, group of data.groups
        addGroup groupName, group

      nodes[k] = makeNode "media/#{k}.#{ext}", v for k, v of data.nodes

      for k, v of data.nodes
        for weight, links of v.links
          nodes[k].link (nodes[l] for l in links), Number(weight)

      nodes.intro.preload()
    .then ->
      nodes.intro.play()

# Big-Bang [![Build Status](https://secure.travis-ci.org/rm-hull/big-bang.png)](http://travis-ci.org/rm-hull/big-bang)

Big-Bang is a ClojureScript game-loop / event-loop abstraction, loosely based on
Racket's [big-bang][a] and implemented on top of [core.async][b]. It is a pure
ClojureScript implementation with no external Javascript dependencies. Using
_Big-Bang_ encourages you to implement what would be otherwise stateful code
in a pure functional manner; of course, inevitably, at some point you have to
punch outside and twiddle some IO or paint some pixels - this can be entirely
encapsulated in the render handler however.

See http://rm-hull.github.io/big-bang/example.html and
http://programming-enchiladas.destructuring-bind.org/rm-hull/8623502
for some in-progress demos,
and for a code comparison between Big-Bang and [OM][c], see here:

[Om mouse move][d] vs. [Big Bang mouse move][e] ツ

[a]: http://docs.racket-lang.org/teachpack/2htdpuniverse.html#(form._world._((lib._2htdp/universe..rkt)._big-bang))
[b]: http://github.com/clojure/core-async
[c]: https://github.com/swannodette/om
[d]: http://programming-enchiladas.destructuring-bind.org/rm-hull/8617445
[e]: http://programming-enchiladas.destructuring-bind.org/rm-hull/8617788

### Pre-requisites

You will need [Leiningen](https://github.com/technomancy/leiningen) 2.3.4 or
above installed.

### Building & Testing

To build and install the library locally, run:

    $ lein cljsbuild once
    $ lein install

To test against [PhantomJS](http://phantomjs.org/), ensure that that
package is installed properly, and run:

    $ lein cljsbuild test

Alternatively, open ```resources/run-tests.html``` in a browser - this
executes the tests and the test results are displayed on the page.

### Including in your project

There is an 'alpha-quality' version hosted at [Clojars](https://clojars.org/rm-hull/big-bang).
For leiningen include a dependency:

```clojure
[rm-hull/big-bang "0.0.1-SNAPSHOT"]
```

For maven-based projects, add the following to your `pom.xml`:

```xml
<dependency>
  <groupId>rm-hull</groupId>
  <artifactId>big-bang</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</dependency>
```
## Basic Usage

### A simple example

The following code sample gives a simple high level overview of how to program
a big-bang world. See http://programming-enchiladas.destructuring-bind.org/rm-hull/8623502
for a running demo of this code:

```clojure
(ns big-bang.example.cat-animation
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [enchilada :refer [canvas ctx canvas-size proxy-request]]
            [cljs.core.async :as async :refer [<!]]
            [dataview.loader :refer [fetch-image]]
            [big-bang.core :refer [big-bang!]]
            [jayq.core :refer [show]]))

(def width (get (canvas-size) 0))
(def height (get (canvas-size) 1))

(defn increment-and-wrap [x limit]
  (if (< x limit)
    (inc x)
    0))

(defn update-state [[x y]]                              ; [1]
  [(increment-and-wrap x width) y])

(defn render-scene [ctx img [x y]]                      ; [2]
  (.clearRect ctx 0 0 width height)
  (.drawImage ctx img x y))

(show canvas)

(go
  (let [cat "https://gist.github.com/rm-hull/8859515c9dce89935ac2/raw/cat_08.jpg"
        img (<! (fetch-image (proxy-request cat)))]     ; [3]
    (big-bang!                                          ; [4]
      :initial-state [0 0]
      :on-tick update-state
      :to-draw (partial render-scene ctx img))))
```

At step [3], an image is fetched asynchronously bound to a local context
inside the ```go``` block - then big-bang is invoked at [4] with an initial world-state
of ```[0 0]``` - the co-ordinates of the canvas onto which we will render
the image.

The big-bang internal ticker is initialized to tick every 17ms (~60FPS) by default,
and call ```update-state``` defined at [1]. This function takes the world-state
(here the co-ordinates are destructured), and returnd a new world-state comprising
an updated X component along with the unmodified Y value.

On world-state being changed, the to-draw ```render-scene``` function is scheduled
to run inside a _requestAnimationFrame()_ callback. Internally the world-state is
advanced in a recursive call, ready to dispatch on the next incoming event: this is
but an implementation detail that needn't be dwelt on, however.

### Architectural Considerations

The ```:on-tick``` timer is just one type of event that big-bang listens on; it is
straightforward to add futher event sources to handle targetted browser events
(e.g. key presses, mouse movement, etc. on specified DOM components) as well as
externally defined core.async channels. Interfacing with native javascript libraries
that emit events is also possible with relatively little effort.

A big-bang handler can control a single or multiple DOM components, however complex
its internal representation.  Likewise it can be used as a smaller component in an
orchestrated multiverse, communicating with other big-bangs over channels in the CSP
style.

**So what is a Big-bang?** A big bang is something that started out of nothing and
continues to tick until the end of time (unless you implement a ```:stop-when?```
handler or specify a maximum lifespan with the ```:max-frames``` option, at which
point the world-state will cease and there will be no further rendering).

### Event Handling and ```IChannelSource```

The ```big-bang.event-handler``` namespace provides a function that installs
event-listeners onto DOM elements, and rather than implementing a callback
architecture, the handler instead returns a reified ```IChannelSource``` object;
this exposes a channel onto which events are placed and a facility to de-install
the event listener.

```clojure
(def listener (add-event-listener (.-body js/document) :click))

(go
  (loop []
    (when-let [e (<! (data-channel listener))]
      (.log js/console (str "Received: " e))
      (recur)))))

(go
  (<! (timeout 20000))
  (shutdown! listener))
```

Multiple event listeners are used internally to drive state transitions
in the ```big-bang!``` game loop on key presses, mouse events, etc.

### Regular Ticking

The ```big-bang.timer``` namespace provides a mechanism that wraps the
javascript ```setInterval``` callback, sending a predetermined payload
on a channel at regular intervals:

```clojure
(def ticker (interval-ticker 100))              ; [1]

(go
  (loop []
    (when-let [x (<! (data-channel ticker))]    ; [2]
      (.log js/console (str "Received: " x ))
      (recur))))

(go
  (<! (timeout 2000)) ; pause for a short time  ; [3]
  (shutdown! ticker))                           ; [4]
```

Notice the similarity to the prior event listener example above.

At [1], a ticker is created and started; note that, while there is no consumer
taking events from the timer channel, messages are dropped in order to prevent
a backlog. Similarly if the consumer loop cannot keep up with the rate that
the ticker is producing, then events will similarly be dropped.

At [2], inside a go block, a value is consumed from the ticker's timer channel.
If a ```nil``` value is returned from the channel, it can be assumed that the
ticker has elsewhere been stopped, and the loop can be terminated. Step [3] then
incurs a 2 second delay before stopping the ticker at step [4].

The ticker is used internally to drive state transitions in the ```big-bang!```
game loop.

## Differences from the Racket implementation

This library is written in the _spirit_ of Racket's big-bang - it is not intended
as a like-for-like copy, nor I have not poured over the implementation details of
big-bang. Inevitably, therefore, there will be differences between the two, which
I will attempt to document here:


## TODO

Migrated the TODO list to github issues: http://github.com/rm-hull/big-bang/issues

## Known Bugs

* ~~```lein cljsbuild test``` does not appear to be working properly, returns _"Could not locate test command."_~~

## References

* http://docs.racket-lang.org/teachpack/2htdpuniverse.html#(form._world._((lib._2htdp/universe..rkt)._big-bang))
* http://worrydream.com/Tangle/

## License

The MIT License (MIT)

Copyright (c) 2014 Richard Hull

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/rm-hull/big-bang/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

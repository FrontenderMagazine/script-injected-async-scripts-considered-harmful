By [Ilya Grigorik][1] on **May 20, 2014**

Synchronous scripts are bad because they force the browser to block DOM
construction, fetch the script, and execute it before the browser can continue 
processing the rest of the page. This, of course, should not be news, and is the
reason why we have been evangelizing the use of asynchronous scripts. Here is 
the canonical example:

    <span><!-- BAD: blocking external script -->
    <span><script <span>src=<span>"http://somehost.com/awesome-widget.js"<span>></script>
    
    <span><!-- GOOD: remote script is loaded asynchronously -->
    <span><script>
        <span>var <span>script <span>= <span>document<span>.<span>createElement<span>(<span>'script'<span>);
        <span>script<span>.<span>src <span>= <span>"http://somehost.com/awesome-widget.js"<span>;
        <span>document<span>.<span>getElementsByTagName<span>(<span>'head'<span>)[<span>0<span>].<span>appendChild<span>(<span>script<span>);
    <span></script>
    </span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span>

What's the difference? In the "bad" example we block DOM construction, wait to
fetch the script, execute it, and then continue to process the rest of the 
document. In the second example, we begin executing the inline script, which 
creates a script element pointing to an external resource, add it to the 
document, and continue processing the DOM.**The difference is subtle but very
important: script-injected scripts do not block on the network.**

**So, that's awesome, right? Script-inject all the things! Not so fast.**

The inline JavaScript solution has a subtle, but very important (and an often
overlooked) performance gotcha: inline scripts block on CSSOM before they are 
executed. Why? The browser does not know what the inline block is planning to do
in the script it is about to execute, and because JavaScript can access and 
manipulate the CSSOM, it blocks and waits until the CSS is downloaded, parsed, 
and the CSSOM is constructed and available. A hands-on network waterfall is 
worth thousands words, consider this example:

![][2]

The [above page][3] loads a CSS file at the top of the page and two script-
injected "async scripts" at the bottom. In other words, it follows all of the "
performance best practices". Except, the scripts themselves can't be executed 
until the CSSOM is ready, which delays the execution of the inline blocks and 
consequently the dispatch of the network requests. As a result, the scripts are 
executed ~3.5s after the page request is initiated.

Note that I'm intentionally forcing a large two second response delay on CSS
and one second delay on JavaScript to highlight the dependency between CSS/CSSOM
and JavaScript execution.

Now, let's compare this to our "bad" example, which 
[uses two blocking script tags][4]:

![][5]

Wait a second, what's going on? **Both scripts are fetched earlier and are
executed ~2.7 seconds after the page request is initiated.** Note that the
scripts are still executed only once the CSS is available (~2.7 second mark), 
but because the scripts are already fetched by the time the CSSOM is ready, we 
can execute them immediately, saving over a second in our processing times.**
Have we been doing it all wrong?**

Before we answer that, let's consider one more example, this time 
[with the "async" attribute][6]:

    <span><script <span>src=<span>"http://udacity-crp.herokuapp.com/time.js?rtt=1&a" <span>async<span>></script>
    <span><script <span>src=<span>"http://udacity-crp.herokuapp.com/time.js?rtt=1&b" <span>async<span>></script>
    </span></span></span></span></span></span></span></span></span></span>

![][7]

> If the async attribute is present, then the script will be executed
> asynchronously, as soon as it is available. If the async attribute is not 
> present ... then the script is fetched and executed immediately, before the user
> agent continues parsing the page.
>
**The async attribute on the script tag provides two critical properties: it
tells the browser to not block DOM construction, and it does not block script 
execution on CSSOM.** As a result, the scripts are executed as soon as they are
downloaded (at ~1.6 seconds) and without having to wait for the CSSOM. A quick 
summary of our results:

|                 | script execution | onload |

| script-injected | ~3.7s
| ~3.7s
|
| blocking script | ~2.7s
| ~2.7s
|
| async attribute | ~1.7s
| ~2.7s
|

So, why have we been advocating the use of this pattern for so long?

1.  **"async" is [not supported][8] by a few older browsers: IE 8/9, Android 2.
    2/2.3
   **. As a result, these older browsers ignore the attribute and treat it as a
    blocking script. This was a big problem back in the day, but this also brings us
    to the next point.
    ..

2.  **All modern browsers have a "preload scanner" (yes, 
    [even IE8/9 and Android 2.3/2.2][9])** which is invoked when the document
    parser is blocked and whose sole responsibility is to "peek ahead" in the 
    document and find resources that should be fetched as soon as possible to 
    unblock the critical rendering path.
   

The script-injected pattern offers no benefits over `<script async>`. The
reason it exists is because`<script async>` was not available and preload
scanners did not exist back when it was first introduced. However, that era has 
now passed, and we need to update our best practices to use async attribute 
instead of script-injected scripts. In short, script-injected "async scripts" 
considered harmful.

Also, note that the preload scanner will only discover resources that are
specified via`src/href` attributes on script and link tags. The preload scanner
cannot and does not execute inline JavaScript blocks, which means that any 
script-injected assets cannot be discovered by the preload scanner. As a result,
the new best practice:

    <span><!-- BAD: the pre async / pre preload scanner era -->
    <span><script>
        <span>var <span>script <span>= <span>document<span>.<span>createElement<span>(<span>'script'<span>);
        <span>script<span>.<span>src <span>= <span>"http://somehost.com/awesome-widget.js"<span>;
        <span>document<span>.<span>getElementsByTagName<span>(<span>'head'<span>)[<span>0<span>].<span>appendChild<span>(<span>script<span>);
    <span></script>
    
    <span><!-- GOOD: modern, simpler, faster, and better all around  -->
    <span><script <span>src=<span>"http://somehost.com/awesome-widget.js" <span>async<span>></script>
    </span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span></span>

| `<script src="...">`                                            | `<
script async src="...">
`                                                    |

| Blocks DOM construction                                               | Does
not block DOM construction
|
| Execution is blocked on CSSOM                                         |
Execution is not blocked on CSSOM
|
| Preload scanner discoverable                                          |
Preload scanner discoverable
|
| Ordered execution of scripts                                          |
Unordered execution
|
| Use where execution order matters, place these scripts at the bottom. | Can
be placed anywhere, ideal for scripts that can tolerate out-of-order execution.
|

## But wait, what about…

To be clear, that's not to say that all inline JavaScript should be avoided.
There is time and place where it may, in fact, be the right solution. A few 
things to consider and to keep in mind:

1.  **Async attribute makes no guarantees about execution order:** scripts are
    executed as they arrive, their order and location in the document has no effect.
    As a result, if you have dependencies, can you eliminate them? Alternatively, 
    can you defer their execution, or make the order of execution a non-issue? The
   [asynchronous function queuing][10] pattern is a good one to investigate.

2.  **The [asynchronous function queuing][10] pattern requires that I
    initialize some variables, which implies that I need an inline script block, are
    we back to square one?
   ** No. If you place your inline block above any CSS declarations - yes, you
    read that right, JavaScript above CSS in document
   `<head>` - then the inline block is executed immediately. The problem
    with inline blocks is that they must block on CSSOM, but if we place them before
    any CSS declarations, then they are executed immediately.
   

3.  **Wait, should I just move all of my JavaScript above the CSS then?** No.
    You want to keep your
   `<head>` lean to allow the browser to discover your CSS and begin
    parsing the actual page content as soon as possible - i.e. optimize the content 
    you deliver in your first round trip to enable the
   [fastest possible page render][11].

 [1]: https://www.igvita.com/
 [2]: img/xasync-injected.png.pagespeed.ic.Nz8cY9Q-Ow.png
 [3]: http://jsbin.com/qefefiyi/9/quiet
 [4]: http://jsbin.com/qefefiyi/8/quiet
 [5]: img/xasync-blocking.png.pagespeed.ic.Qf2_CnkjCV.png
 [6]: http://jsbin.com/qefefiyi/7/quiet
 [7]: img/xasync-async.png.pagespeed.ic.-aH-Uzp-RY.png
 [8]: http://caniuse.com/#search=async

 [9]: http://andydavies.me/blog/2013/10/22/how-the-browser-pre-loader-makes-pages-load-faster/

 [10]: http://stackoverflow.com/questions/6963779/whats-the-name-of-googla-analytics-async-design-pattern-and-where-is-it-used
 [11]: https://developers.google.com/speed/docs/insights/mobile
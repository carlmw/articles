# Optimising JavaScript animation

I'm going to use this as a place to collect the tips and tricks I've picked up for creating performant animations in CSS, and both 2D and 3D Canvases.

## Clustering DOM reads and writes
Setting any property on an element that changes it's position or appearance will mark the page's layout as 'dirty'. In order to reduce the ammount of work it needs to do the browser will wait until the last possible moment before it renders these changes.

This work cannot be postponed when trying to get the position or dimensions of an element. Take the following as an example:
    
    var width = domEl.clientWidth + 10;
    domEl.style.width = width + 'px';
    var height = domEl.clientHeight + 10;
    domEl.style.width = height + 'px';
    
We're just adding 10 pixels to the height and width of the element. But as soon as we set the element's width the page's layout becomes dirty. So on the next line when we sample the element's height the browser has to reflow the entire page so that it can he sure that it's giving us the correct value following the change.

The solution is to get the dimensions of our element in one go.

    var width = domEl.clientWidth + 10,
        height = domEl.clientHeight + 10;

    domEl.style.width = width + 'px';
    domEl.style.height = height + 'px';

By retrieving the element's width and height at the same time the browser can be sure that the page hasn't changed between each read.

The following list of methods and properties (and potentially others) can trigger a page reflow. 

### DOMElement
    clientHeight, clientLeft, clientTop, clientWidth, focus(), getBoundingClientRect(),
    getClientRects(), innerText, offsetHeight, offsetLeft, offsetParent, offsetTop, offsetWidth, outerText, scrollByLines(), scrollByPages(), scrollHeight, scrollIntoView(),
    scrollIntoViewIfNeeded(), scrollLeft, scrollTop, scrollWidth

### window
    getComputedStyle(), scrollBy(), scrollTo(), scrollX, scrollY

## Cache DOM reads
Once again we want to limit the number of reflows the browser has to deal with. So caching values where we can is very important.

For example:
    
    var loop = function () {
          var width = domEl.clientWidth + 10;
          domEl.style.width = width + 'px';

          loop();
        };

    loop();

This will force a reflow on every iteration (ignoring the over simplified loop).

We'd be better to cache the initial clientWidth value and increment it.

    var width = domEl.clientWidth,
        loop = function () {
          width = width + 10;
          domEl.style.width = width + 'px';

          loop();
        };

    loop();

Note that we're now reading the clientWidth of the element just once.

## Use requestAnimationFrame
The standard for any kind of repetition in JavaScript is setInterval. It takes a callback and fires it at an interval of your choosing. For animation this is pretty sucky. Regardless of whether your change is going to be rendered, setInterval will do it's work. This can cause a reduction in battery life and choppy animation.
    
    var width = domEl.clientWidth;
    setInterval(function () {
      width = width + 10;
      domEl.style.width = width + 'px';
    }, 1000 / 60);

This is where requestAnimationFrame comes in. It will tell the browser that you have something you want to render and the browser will trigger the callback you provide just before it performs a repaint.
    
For example:

    var width = domEl.clientWidth;
    (function loop () {
      width = width + 10;
      domEl.style.width = width + 'px';

      window.requestAnimationFrame(loop);
    })();

Using requestAnimationFrame will ensure that our tweening is only performed when the browser can actually render it. The browser can even pause animation when our page isn't active, again saving battery life.

Support is pretty widespread, but here's a simple shim with fallback to setTimeout for browsers that haven't caught up yet.

    window.requestAnimationFrame = (function () {
      return  window.requestAnimationFrame || 
              window.webkitRequestAnimationFrame || 
              window.mozRequestAnimationFrame || 
              window.oRequestAnimationFrame || 
              window.msRequestAnimationFrame || 
              function (cb) {
                window.setTimeout(cb, 1000 / 60);
              };
    }());

## Hardware acceleration
With CSS transforms came hardware acceleration. Why is hardware acceleration good? Well it takes tasks that would usually be performed by the CPU and offloads them to the GPU which is better at drawing and probably not very busy when you're only viewing a webpage.

The simplest way to offload animation to the GPU is to use CSS transitions.

For example:

domEl.style.transition = 'all 1s ease-out';
domEl.style.transform = 'translateX(100px)';

In most cases it's usually enough to simply use CSS transitions to offload the work to the GPU. However in some cases we may need to force the rendering of the element or it's parent onto the GPU. We can more strongly suggest that the browser should do this using a 3D transform.

We can achieve this by simply adding the following CSS property to the element in question.

    transform: translateZ(0px);

This should be used sparingly and only where needed since it can cause some undesirable side effects (particularly around the positioning of child elements) and may even worsen performance.

## Round pixel values
Sub pixel rendering can cause choppy animation since the browser is forced to anti-alias CSS pixels when they're positioned between real screen pixels. This is a lot of extra work and is often unnecesary.

We can avoid this by rounding values to full pixels

For example:
    
    domEl.style.width = 100 / 3; // 33.333...
    domEl.style.width = width + 'px';

Could be written as:

    domEl.style.width = Math.floor(100 / 3); // 33px
    domEl.style.width = width + 'px'; // 33px

Or slightly more efficiently with some bitwise shenanigans [http://jsperf.com/math-round-vs-hack](jsperf - math-round-vs-hack)
    
    var width = ~~(100 / 3);
    domEl.style.width = width + 'px'; // 33px


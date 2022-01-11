---
title:  "Implementing a timeline with scrolling and zooming - or how I failed at elementary school math"
date:   2022-01-10 22:18:32 +0200
categories: blog
excerpt: "I had to implement a timeline that can be scrolled and zoomed. I faced some challenges, but was doing quite well. Until I ran into some issues that took quite some time to work out. In the end, I failed measurably at some very basic elementary school math."
header:
  og_image: /assets/images/2022-01-10/timeline-scrolling-zooming-og.png
---


Recently I had the task of implementing a timeline. Not a big deal? You're probably right, it's just a bit of Flexbox magic. But there are three challenges:

- The events in the timeline are not equally distributed and the space between the events need to follow that.
- Since the timeline might contain a huge amount of events, a user needs to be able to zoom into the timeline.
- If the full timeline is not visible anymore, the user needs to be able to scroll left and right by grabbing the timeline.

I'll omit any labels or fancy details and focus on the main challenges only.

1. [Fully working example](#fully-working-example)
1. [Easy Peasy - the Flexbox magic](#easy-peasy-the-flexbox-magic)
1. [Scrolling](#scrolling)
1. [Zooming first attempt](#zooming-first-attempt)
1. [Fix the calculations](#fix-the-calculations)


## Fully working example

First, let's take a look at the fully working example. It will give a better impression of the solution than the dry list of requirements.

Use the mouse wheel to scroll in and out. Grab the timeline to move it left and right.

<iframe src="https://thomas.preissler.me/timeline/" width="100%" style="border: 0; height: 104px; margin-bottom: 1.3rem;"></iframe>

It's implemented with pure HTML, CSS and JavaScript without a single dependency. 😇

You can find the full source code on [Github](https://github.com/ThomasPr/timeline).


## Easy Peasy - the Flexbox magic
{: #easy-peasy-the-flexbox-magic }

Let's get started: A simple timeline requires only very few lines of CSS. It fills the whole screen and rescales well for different screen sizes. It employs a simple Flexbox layout with a `flex-direction: row` to keep the events in a row. But the events should not be distributed evenly, the space between them should differ and be customizable.

Thanks to Flexbox it's quite simple: Just put a `div` between the events that has a `flex-grow` property. A `div` with `flex-grow: 8` will get four times the space of a `flex-grow: 2`. The browser will then scale the size between the events accordingly to the `flex-grow` property. To put this abstract property to the timeline we can assume that the number of days between two events is set to the `flex-grow` property, e.&thinsp;g. if two events are 30 days apart, we would use `flex-grow: 30`. Easy peasy, right? I cannot image how painful an implemention would look like without the power of Flexbox.

The result looks quite good. And yet, it wasn't a challenge to make it happen. So far, there is no reason for a longish blog post like this. But the task will get much harder in the next section. Be warned!

For now the `scrollable` and `zoomable` classes are needless, but they will get important very soon. I want to drop the full HTML and CSS here so that I don't need to paste it later on again.


```html
<div class="scrollable">
  <div class="timeline zoomable">
    <div class="spacer" style="width: 26px;"></div>
    <div class="event"></div>
    <div class="spacer" style="flex-grow: 1;"></div>
    <div class="event"></div>
    <div class="spacer" style="flex-grow: 2;"></div>
    <div class="event"></div>
    <div class="spacer" style="flex-grow: 3;"></div>
    <div class="event"></div>
    <div class="spacer" style="flex-grow: 5;"></div>
    <div class="event"></div>
    <div class="spacer" style="flex-grow: 8;"></div>
    <div class="event"></div>
    <div class="spacer" style="width: 26px;"></div>
  </div>
</div>
```

```css
.scrollable {
  overflow-x: hidden;
  width: 100%;
}

.timeline {
  display: flex;
  flex-direction: row;
  align-items: center;
  height: 104px;
  background-color: #f2f3f3;
}

.event {
  width: 26px;
  height: 26px;
  border-radius: 100%;
  background-color: rgb(0, 127, 255);
}

.spacer {
  height: 6px;
  background-color: rgba(0, 127, 255, 0.5);
}
```


## Scrolling

Scrolling consists of multiple features:
- change the mouse point to `grabbing` when pressing the mouse button and back to `grab` when releasing it
- save the mouse position whe pressing the mouse button
- move the timeline accordingly to the left or right when the mouse is moved and the button is pressed

The latter feature is implemented by the `scrollLeft` property of the container. Every `div` has a `scrollLeft` property which is set to 0 by default. If the content of the container is wider than the container itself, the `scrollLeft` property can be used to drag the content to left so that the wider content can be capped on the left side.

An example: The visible container is 350&thinsp;px wide and `overflow-x` is set to `hidden`. It contains an element with double width, i.&thinsp;e. 700&thinsp;px. Without further specification, the browser would set the child element left-aligned with the visible container and cut off the right half. But with the help of the `scrollLeft` property the too wide child element can be dragged to the left, so that an arbitrary section can become visible. To show the center and cut off the same amount on both sides, `scrollLeft` would have to be set to 175&thinsp;px in this example.

![Explaining the scrollLeft property](/assets/images/2022-01-10/scrollLeft.svg)

Since there is no way to ask the browser if a user has the mouse button pressed right now, we need to listen to the `mousedown` and `mouseup` events and to remember the state manually. In addition when the `mousedown` event fires, we save the current position of the mouse pointer and the current `scrollLeft` position as initial values. We need those values later on to calculate the mouse distance.

Actually, it's quite simple to implement that. The full source code can be viewed on [Github](https://github.com/ThomasPr/timeline/blob/main/scrollable.js), I'll walk through some major steps.


### mousedown

When the mouse button gets pressed, the current mouse position and the current `scrollLeft` value needs to be stored. In addition, the cursor style is changed to `grabbing`.


```javascript
scrollableElement.addEventListener('mousedown', (mouseEvent) => {
  mouseDown = true;
  scrollableElement.style.cursor = 'grabbing';
  initialGrabPosition = mouseEvent.clientX;
  initialScrollPosition = scrollableElement.scrollLeft;
});
```


### mouseup

When the user releases the mouse button, the values that were changed in the `mousedown` event must be reset.

```javascript
scrollableElement.addEventListener('mouseup', () => {
  mouseDown = false;
  scrollableElement.style.cursor = 'grab';
});
```


### mousemove

When the user moves the mouse, a very few mathematical calculations are required. The distance between the starting point of grabbing action and the current position of the mouse pointer must be computed and the value for `scrollLeft` must be reduced by exactly the same value.

```javascript
scrollableElement.addEventListener('mousemove', (mouseEvent) => {
  if (mouseDown) {
    const mouseMovementDistance = mouseEvent.clientX - initialGrabPosition;
    scrollableElement.scrollLeft = initialScrollPosition - mouseMovementDistance;
  }
});
```

That's it. Now the user can move the timeline left and right if it doesn't fit on the screen. By default, the timeline is exactly as wide as the window, so the next task will be to implement the zoom behavior to actually use the scrolling feature.


## Zooming - first attempt
{: #zooming-first-attempt }

The first challenge will be a draft implementation of the zooming feature. Spoiler: it won't work as excepted and it took me ages to figure out whats going wrong.

The idea of the zoom function is to enlarge the timeline beyond the visible width by using the `width` property and set it to more than 100&thinsp;% for the timeline itself and `overflow-x: hidden` for the its parent. The result is a timeline that is larger than the visible screen, but cropped to the original size. The spacing between events is doubled, so it feels like the timeline has been zoomed in.

If the width of the child element changes, it gets stretched to the right and therefore cropped on the right. To change the focus, the `scrollLeft` property needs to be changed accordingly.

My first attempt was to calculate the `scrollLeft` property according to the mouse position. If the mouse pointer is between the third and the fourth quarter and the user zooms in, the mouse pointer has to stay on that point. So the user has to move the timeline in such a way that the he gets the impression of thetimeline moving around his mouse pointer. It looks like he zoomed in at that exact spot. My first (and incorrect) assumption was to simply calculate the space difference and increase the `scrollLeft` property to match the mouse position accordingly.

An example: The timeline has a width of 100&thinsp;px, we zoom in to make the timeline 200&thinsp;px wide, so it has doubled in size. The mouse pointer is positioned between the third and fourth quarter, where it must be positioned also after zooming.

![View of the Timeline before zooming](/assets/images/2022-01-10/zooming-simple-before.svg)

`scrollLeft` must be computed in such a way that 75&thinsp;% of the timeline is still on the left of the mouse pointer after zooming. In this example `scrollLeft` is increased by 100&thinsp;px * 0.75, so it gets the value 263.

![View of the Timeline after zooming](/assets/images/2022-01-10/zooming-simple-after.svg)

Looks simple? Yes, it actually is. But ...


## Does it work?

Let's look again at the example of the timeline, which is zoomed in by the factor 2. This time, however, the example timeline will be in a more detailed way, representing the events and spaces between them.

![View of the Timeline before zooming](/assets/images/2022-01-10/zooming-timeline-before.svg)

The mouse pointer is again positioned at three-quarters, which in this example is at the end of the last element.

In the next step, the user zooms into the timeline so that the width increases to twice the original width.The algorithm is applied and the mouse pointer is moved back to its original position. Before and after zooming, the mouse pointer is over a point that is about three quarters wide.

![View of the Timeline after zooming](/assets/images/2022-01-10/zooming-timeline-after.svg)

The result is as expected. Or is it? The mouse pointer has the correct position relative to the timeline itself, but the mouse pointer is far away from the last element and no longer right next to it. To the user, it no longer seems as if the center of the zooming is below the mouse pointer. But that is exactly the expected behavior.

What should it look like properly?

![Expected Timeline movement](/assets/images/2022-01-10/expected-timeline-movement.svg)

It is clear to see that the timeline is pushed significantly further to the right than the algorithm did. As can be seen, the `scrollLeft` property must be only roughly half the size. Where is the difference compared to the first simple example?

Well, ... The timeline consists of variable spacing between the points and the points themselves with fixed width. When the user scales the timeline, the spaces are changed, but the events are not.

Simple example:
Assume a Timeline which consists on the left half of 2 points with one length unit each, in addition there are 2 spaces with two length units each. In total, the left half of this imaginary Timeline is 6 length units (2\*1 + 2\*2).
The right half consists of only one distance with 6 length units.

![before zooming](/assets/images/2022-01-10/example1.svg)

If the user increases the timeline by a factor of two, all spacings get doubled, but not the events. On the left side the resulting size is 10 (2\*1 + 2\*4), whereas on the right half the distance doubles to 12. Thus the center of the timeline has shifted.

![before zooming](/assets/images/2022-01-10/example2.svg)

For single elements like an image the simple algorithm works well. Even if all elements scale the same, it works. But the timeline consists of the variable spacing between the points and the points with fixed widths. The effect is that the timeline does not scale proportionally.

This finding took me several days (and sleepless nights). In the end, the problem was simply some elementary school math.


## Fix the calculations

The idea of the algorithm was quite correct. The timeline must be moved so that the event under the mouse pointer is fixed. The only difference is that this point not point of the timeline as a whole, but a child element of the timeline.

First, it needs to be identified at which event or space the mouse pointer is located and relative to that element the original algorithm can be applied. In the example from above, the last point of the timeline is identified as the element below the mouse pointer. Relative to this element, the mouse pointer is at the very end. And it is exactly this position that we would have to reach again after zooming.

This sounds quite simple, but for the calculation of the `scrollLeft` property there are some additional information required. The position of the mouse pointer is measured from the edge of the screen. To compute the correct value the algorithm also needs all other relevant values in relation to the edge of the screen. This includes the distance of the timeline, its parent container as well as the the child element within the timeline where the mouse pointer is positioned.

The code for this looks like this:

```javascript
const mousePosition = wheelEvent.clientX;
const elementUnderMouseLeft = getLeft(elementUnderMouse);
const zoomableLeft = getLeft(zoomableElement);
const containerLeft = getLeft(containerElement);
const moveAfterZoom = getWidth(elementUnderMouse) * mousePositionRelative;

containerElement.scrollLeft =
  elementUnderMouseLeft
  - zoomableLeft
  - mousePosition
  + containerLeft
  + moveAfterZoom;
```

You can find the full source of the zooming feature code on [Github](https://github.com/ThomasPr/timeline/blob/main/scrollable.js).

Comments are welcome on Twitter or LinkedIn.

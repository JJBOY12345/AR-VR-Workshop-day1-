# AR Shape Selector — How It Works

A beginner-friendly breakdown of a working WebAR project built with A-Frame and AR.js.
The goal of this document is not just to show what the code does, but to explain **why** each decision was made — including the mistakes that were made along the way and what they taught us.

---

## What This Project Does

Point your phone camera at a [Hiro marker](https://ar-js-org.github.io/AR.js-Docs/marker-based/) and a 3D shape appears floating above it. Buttons at the top of the screen let you switch between a Cube, Sphere, Cylinder, and Torus in real time. No app install needed — it runs entirely in the browser.

---

## The Technology Stack

This project sits on top of three layers, each doing a different job:

```
Your HTML/CSS/JS  ← what you write
      ↓
   A-Frame        ← turns HTML tags into a 3D world
      ↓
   AR.js          ← connects that 3D world to the camera + marker tracking
      ↓
  three.js        ← the underlying 3D engine (you never touch this directly)
```

### A-Frame
> **Official docs:** https://aframe.io/docs/

A-Frame is a web framework that lets you describe 3D scenes using HTML-style tags instead of writing raw 3D code. For example, instead of writing hundreds of lines of JavaScript to create a rotating cube, you write:

```html
<a-box color="blue" animation="property: rotation; to: 0 360 0; loop: true;"></a-box>
```

A-Frame is built on top of **three.js**, which is the actual 3D engine doing the rendering. A-Frame is the friendly wrapper that hides the complexity.

- A-Frame GitHub: https://github.com/aframevr/aframe
- A-Frame primitives (all the shapes): https://aframe.io/docs/1.0.0/primitives/

### AR.js
> **Official docs:** https://ar-js-org.github.io/AR.js-Docs/

AR.js adds augmented reality on top of A-Frame. Specifically, it watches the camera feed every frame, looks for a known marker pattern (the Hiro image), calculates where the marker is in 3D space, and moves the A-Frame scene to match. From A-Frame's perspective, the camera is just moving — AR.js is secretly doing all the tracking work to create that illusion.

- AR.js GitHub: https://github.com/AR-js-org/AR.js
- Marker-based AR docs: https://ar-js-org.github.io/AR.js-Docs/marker-based/
- Hiro marker image (print or display this): https://ar-js-org.github.io/AR.js/data/images/hiro.png

### The Hiro Marker
The Hiro marker is a specific black-and-white image that AR.js's tracking algorithm is pre-programmed to recognise. Think of it like a QR code, but one that also gives the software enough geometric information to figure out angle, distance, and orientation in 3D. You can display it on another screen or print it — either works.

---

## Version Pairing — Why This Matters

```html
<script src="https://aframe.io/releases/1.0.4/aframe.min.js"></script>
<script src="https://raw.githack.com/AR-js-org/AR.js/master/aframe/build/aframe-ar.js"></script>
```

This is one of the most important and least documented things about this project.

**A-Frame and AR.js must be compatible versions.** AR.js hooks into A-Frame's internal scene lifecycle at startup — it reads the DOM, builds its tracking system, and wires itself up. If the A-Frame version changes how that internal system works, AR.js breaks, often silently with no useful error message.

The combination above (`aframe@1.0.4` + `AR.js master via raw.githack.com`) is the pairing used in the official AR.js documentation examples. It is the tested, known-working combination for marker-based AR with HTML button overlays.

> **Why raw.githack.com instead of jsdelivr?**
> `jsdelivr` caches CDN files aggressively and sometimes serves stale builds. `raw.githack.com` serves files directly from the GitHub repository with proper caching headers, which is more reliable for an actively developed library.

---

## The HTML Structure

```
<head>          ← imports, settings, styles
<body>
  <div>         ← the button bar (lives OUTSIDE a-scene)
  <a-scene>     ← the entire 3D + AR world
    <a-marker>  ← the Hiro marker anchor
      <a-box>   ← 3D shapes (pre-declared, all four of them)
      <a-sphere>
      <a-cylinder>
      <a-torus>
    </a-marker>
    <a-entity camera>  ← the AR camera
  </a-scene>
  <script>      ← the button logic
```

The most important structural rule: **the button bar is a plain HTML `<div>` that sits outside `<a-scene>`**. If you put buttons inside the scene, they either disappear or stop responding to taps. Browsers treat the WebGL canvas (the AR viewport) as one layer and regular HTML as another. The buttons need to live in the HTML layer, positioned on top using CSS.

---

## CSS: Positioning the Buttons

```css
body { margin: 0; overflow: hidden; }

#shape-menu {
  position: fixed;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 999;
  display: flex;
  gap: 8px;
}
```

Line by line:

| Property | What it does |
|---|---|
| `margin: 0` | Removes the default white border browsers add around pages |
| `overflow: hidden` | Prevents scrollbars appearing when the AR canvas fills the screen |
| `position: fixed` | Pins the div to the screen — it won't scroll away |
| `top: 10px` | 10 pixels from the top of the screen |
| `left: 50%` + `transform: translateX(-50%)` | A classic CSS trick to horizontally centre something of unknown width |
| `z-index: 999` | Stacking order — 999 puts this above the AR canvas layer |
| `display: flex; gap: 8px` | Lays children (the buttons) side by side with 8px spacing |

---

## The AR Scene Tag

```html
<a-scene embedded arjs="sourceType: webcam; debugUIEnabled: false;">
```

Two attributes:

- **`embedded`** — by default A-Frame tries to run in full VR mode, taking over the entire browser. `embedded` tells it to behave like a regular page element instead.
- **`arjs="..."`** — passes configuration to AR.js:
  - `sourceType: webcam` — use the device camera as the video feed
  - `debugUIEnabled: false` — hides the developer overlay (a black debug canvas AR.js shows by default)

> **What was tried and broke:** Adding extra renderer flags like `renderer="logarithmicDepthBuffer: true"` or extra detection mode flags caused the camera to zoom in incorrectly and the marker to stop being detected. The rule is: keep `<a-scene>` minimal unless you have a specific reason to change something.

---

## The Marker and Shapes

```html
<a-marker preset="hiro">
  <a-box     id="shape-box"       visible="true"  ... ></a-box>
  <a-sphere  id="shape-sphere"    visible="false" ... ></a-sphere>
  <a-cylinder id="shape-cylinder" visible="false" ... ></a-cylinder>
  <a-torus   id="shape-torus"     visible="false" ... ></a-torus>
</a-marker>
```

`<a-marker preset="hiro">` is AR.js's custom tag. It continuously watches the camera. When it detects the Hiro pattern, it makes its children visible and positions them in 3D space relative to the marker. When the marker leaves the frame, children disappear again.

### The Critical Architectural Decision: Pre-declare Everything

All four shapes are written in the HTML from the start — even the three that begin as `visible="false"`.

**This is the most important thing in the whole file.** Here is why:

When the page loads, AR.js reads the DOM (the HTML structure) and builds an internal 3D scene graph — a map of every object and how they relate to each other. Once this process completes, AR.js considers the scene "done."

If you try to add a new shape **after** this point (using JavaScript to create elements dynamically), AR.js often fails to attach the new element to the marker's internal tracking object. The shape exists in the HTML, but never in the 3D world — so it simply never appears. This was the root cause of the original broken version of this code.

Pre-declaring all shapes solves this entirely. All four are part of the scene graph from the start. Hiding and showing them later is just a visibility toggle — the 3D attachment has already been established.

### Shape Attributes Explained

```html
<a-box
  id="shape-box"
  position="0 0.25 0"
  scale="0.5 0.5 0.5"
  color="#2979ff"
  animation="property: rotation; to: 0 360 0; loop: true; dur: 4000; easing: linear"
  visible="true">
```

| Attribute | Meaning |
|---|---|
| `id` | A unique name so JavaScript can find this element |
| `position="0 0.25 0"` | X Y Z position relative to the marker centre. Y=0.25 lifts it slightly off the marker surface |
| `scale="0.5 0.5 0.5"` | Makes the box half-size in all three dimensions (default box is 1m × 1m × 1m) |
| `color="#2979ff"` | Hex colour, same format as CSS colours |
| `animation="..."` | A-Frame's built-in animation system. This one rotates the shape 360° around the Y axis, looping forever over 4 seconds |
| `visible="true/false"` | Whether this object is rendered |

> **Note on `<a-torus>`:** The torus has two radius values — `radius` (how big the ring is overall) and `radius-tubular` (how thick the tube of the ring is). A-Frame docs for all primitives: https://aframe.io/docs/1.0.0/primitives/a-torus.html

> **Note on cones:** A-Frame has no `<a-cone>` primitive. The standard approach is to use `<a-cylinder>` with `radius-top` set near zero, which creates a cone shape.

---

## The JavaScript Logic

```javascript
var shapes = ['box', 'sphere', 'cylinder', 'torus'];

function showShape(name) {
  shapes.forEach(function(id) {
    var el  = document.getElementById('shape-' + id);
    var btn = document.getElementById('btn-' + id);

    if (id === name) {
      el.setAttribute('visible', true);
      btn.classList.add('active');
    } else {
      el.setAttribute('visible', false);
      btn.classList.remove('active');
    }
  });
}
```

When a button is tapped, `showShape('sphere')` (for example) is called. The function loops through all four shape names. For each one, it finds the element in the HTML using `getElementById`, then either shows or hides it using `setAttribute`.

**Why `setAttribute` works here (but doesn't always):**
`setAttribute('visible', false)` is reliable *because* all shapes were pre-declared in the HTML. A-Frame's component system — the part that watches for attribute changes and responds to them — is only fully initialised for an element after A-Frame processes it at startup. Since all four shapes were in the DOM at startup, all four are fully initialised and ready to respond to attribute changes.

The button highlight is handled separately via `classList.add('active')` and `classList.remove('active')`, which adds or removes the CSS `.active` class (which changes the button to blue).

---

## What Went Wrong in Earlier Versions (and Why)

Understanding failures is often more useful than understanding successes:

| Mistake | What broke | Why |
|---|---|---|
| Used `aframe@1.4.0` with an unpinned AR.js CDN | Scene loaded but entity initialisation was unreliable | Version mismatch — the two libraries' internal APIs were out of sync |
| Added extra `renderer` and `detectionMode` flags to `<a-scene>` | Camera zoomed in too far, marker not detected | Those flags alter the camera and detection pipeline in ways that conflict with the defaults AR.js expects |
| Put UI buttons inside `<a-scene>` | Buttons weren't tappable | HTML input events don't propagate through the WebGL canvas |
| Created shapes dynamically via JavaScript at click time | Shapes never appeared | AR.js had already finished building its scene graph — late-added entities didn't get attached to the marker's 3D object |

---

## How to Go Further

If this project sparked your curiosity, here are natural next steps:

**Learn the tools used here:**
- A-Frame introduction: https://aframe.io/docs/1.0.0/introduction/
- AR.js marker tracking guide: https://ar-js-org.github.io/AR.js-Docs/marker-based/
- MDN HTML basics (2–3 hours): https://developer.mozilla.org/en-US/docs/Learn/HTML/Introduction_to_HTML
- MDN CSS basics: https://developer.mozilla.org/en-US/docs/Learn/CSS/First_steps

**Things to try next with this code:**
- Change the `color` attribute to any hex colour
- Change `dur: 4000` in the animation to make shapes spin faster or slower
- Add a 5th button for a cone — use `<a-cylinder radius-top="0.001" radius-bottom="0.28">`
- Try loading a 3D model (`.gltf` file) using `<a-gltf-model>` instead of a primitive shape — A-Frame docs: https://aframe.io/docs/1.0.0/primitives/a-gltf-model.html
- Create your own custom marker: https://ar-js-org.github.io/AR.js-Docs/marker-based/#pattern-markers

**The bigger picture — how WebAR works:**
The flow is: Camera feed → AR.js detects marker → calculates pose (position + orientation in 3D) → A-Frame positions the scene to match → three.js renders the result composited over the video. Every frame, this loop runs at up to 60 times per second.

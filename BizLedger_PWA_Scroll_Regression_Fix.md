# BizLedger — Fix PWA Scroll Regression After Pull-to-Refresh Fix

## Bug

After adding the pull-to-refresh prevention, normal mobile scrolling is broken.

Current behavior:

```
User reaches bottom of page
        ↓
Tries to swipe upward
        ↓
Page does not move toward the top
        ↓
Scrolling feels blocked / treated like refresh protection
```

This must be fixed without bringing back Chrome pull-to-refresh.

## Required Behavior

Normal scrolling must always work:

```
Top → Middle → Bottom
Bottom → Middle → Top
```

The user must be able to move freely in both directions.

Pull-to-refresh must remain disabled.

Only this gesture should be blocked:

```
At the very TOP
        +
Swipe DOWN
        ↓
Prevent browser pull-to-refresh
```

Do not block:

- At bottom + swipe UP
- At middle + swipe UP
- At middle + swipe DOWN
- Normal page scrolling

## 1. Remove the Overly Broad Touch Blocking

Do not use a global handler that prevents touchmove for all app scrolling.

Avoid:

```js
document.addEventListener('touchmove', function(e){
  e.preventDefault();
}, {passive:false});
```

Also do not use logic that identifies the wrong scroll container.

The previous pull-to-refresh JavaScript guard must be reviewed and removed/restricted if it interferes with normal scrolling.

## 2. Prefer CSS Overscroll Protection

Use:

```css
html {
  overscroll-behavior-y: none;
}

body {
  overscroll-behavior-y: none;
  overflow-x: hidden;
}
```

For the actual application scroll container, make sure it remains scrollable:

```css
#APP {
  overflow-y: auto;
  overscroll-behavior-y: contain;
  -webkit-overflow-scrolling: touch;
}
```

For bottom sheets / internal scroll areas:

```css
.shwrap {
  overflow-y: auto;
  overscroll-behavior-y: contain;
  -webkit-overflow-scrolling: touch;
}
```

Do not add `touch-action: none;` because it can disable normal touch scrolling.

If touch-action is required, use `touch-action: pan-y;`.

## 3. Important Scroll-Container Rule

The pull-to-refresh protection must detect the actual element that scrolls.

Do not assume `document.body.scrollTop` is the application's scroll position.

The implementation must identify whether BizLedger uses:

- window/document scrolling, or
- `#APP` scrolling, or
- `.page` scrolling,

and apply the protection to the correct container.

## 4. If JavaScript Protection Is Still Required

Only prevent the specific gesture:

```
scroll position == 0
AND
finger moves downward
```

Use a narrowly scoped guard:

```js
var ptrStartY = 0;

document.addEventListener('touchstart', function(e){
  if(!e.touches.length) return;
  ptrStartY = e.touches[0].clientY;
}, {passive:true});

document.addEventListener('touchmove', function(e){
  if(!e.touches.length) return;

  var dy = e.touches[0].clientY - ptrStartY;

  // Only protect downward movement.
  if(dy <= 0) return;

  var scroller = e.target.closest('#APP, .page, .shwrap');

  if(!scroller) return;

  // Only block when THIS scroll container is at its top.
  if(scroller.scrollTop <= 0){
    e.preventDefault();
  }
}, {passive:false});
```

However, test CSS-only protection first. If CSS completely prevents pull-to-refresh, the JavaScript guard should preferably be removed.

## 5. Acceptance Test

On a real Android PWA:

- Start at the top.
- Swipe down → no browser reload.
- Scroll downward normally → works.
- Reach the bottom.
- Swipe upward → must move toward the top.
- Continue swiping upward → reaches the top normally.
- From the middle, swipe down → normal scrolling works.
- From the middle, swipe up → normal scrolling works.
- Bottom sheets still scroll normally.
- Invoice forms still scroll normally.
- No accidental page reload.
- No Firebase/UI data loss.

## Definition of Done

Disable pull-to-refresh without disabling or trapping normal bidirectional PWA scrolling.

The user must be able to freely scroll from top → bottom and bottom → top, while only the browser's top-edge pull-down refresh gesture is prevented.
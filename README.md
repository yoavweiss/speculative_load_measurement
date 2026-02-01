[@yoavweiss](https://github.com/yoavweiss), [@tunetheweb](https://github.com/tunetheweb)

# Speculative load measurement API

## Introduction

Modern web applications can use speculative loading techniques to improve navigation performance.
However, developers currently lack visibility into whether these speculations were actually used, making it hard for them to deploy higher eagerness values than "conservative". Such values can result in wasted user bandwidth and server-side load, and currently developers have no way to weigh the trade-off between those and potential performance gains.

This proposal addresses this by exposing information about unused speculative loads on page dismissal, enabling developers to measure speculation effectiveness and optimize their speculative load strategies, to pick the trade-off that's right for them.

## Goals

1. Enable developers to measure the effectiveness of their speculation rules and preload strategies. 
1. Expose relevant metadata ([speculation rules tags](https://html.spec.whatwg.org/C#prefetch-record-tags), `as` attribute values) to make the above measurement actionable.
1. Only expose speculation data after the page starts [unloading](https://html.spec.whatwg.org/C#unload-a-document), to reduce potential confusion.
1. Only expose data on user actions in the current document (even for cross-origin speculative loads), without exposing cross-origin data.

## Non-Goals
1. Enable developers to detect and handle situations when speculations are unsuccessful.

## API Design

We will extend the event object that the `pagehide` event gets as input to include a `speculations` attribute, which will hold information about unused preloads, speculation rules navigational prefetches and speculation rules prerenders.

The information contained would be:
- Preload
  - URL
  - The [`as` attribute](https://html.spec.whatwg.org/#attr-link-as) value, as it can often lead to mismatches and unused preloads
  - The `crossorigin` attribute value, as it can similarly lead to mismatches
- Speculation rules navigational prefetch
  - URL
  - The relevant [tags](https://html.spec.whatwg.org/C#prefetch-record-tags).
- Speculation rules prerender
  - URL
  - The relevant [tags](https://html.spec.whatwg.org/C#prefetch-record-tags).

### Why `pagehide`?

Only on page unload is the final state of preload usage and/or speculations known. `pagehide` is the preferred event to handle over `beforeunload` (which can be cancelled) and `unload` (which has reliability issues for bfcache-eligible pages and [is being deprecated](https://developer.chrome.com/docs/web-platform/deprecating-unload)).

Note that `pagehide` does still not have reliability guarantees, particularly on mobile, if the page is dismissed without being unloaded (e.g. killed while backgrounded). However, the alternative `visibilitychange` event,  while being the last reliable event, does not indicate the page is being navigated away from, and may also be called multiple times as tabs are switched back and forth.

The intent of the API is to provide a means of reasonably measuring preload/speculation usage across a broad set of page loads rather than to provide comprehensive guarantees of reporting for every page load, so this reliability issue is a tradeof to reduce developer complexity of other alternatives.

### Example Usage

```javascript
window.addEventListener('pagehide', (event) => {
  const { unusedPreloads, unusedPrefetches, unusedPrerenders } = event.speculations;

  // Send unused speculation data to analytics endpoint
  fetch('/analytics/unused-speculations', {
    method: 'POST',
    keepalive: true,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      unusedPreloads,
      unusedPrefetches,
      unusedPrerenders
    })
  });
});
```

## Key Concepts

### "Unused" Definition

A speculative navigation is considered **unused** if it was initiated but the user navigated to a different URL than the speculation target.
The determination is made at pagehide time by comparing all tracked speculations against the URL of the navigation that triggered the page unload.
Whether or not a speculation was successful (e.g. didn't result in a 403) is not relevant. A failed speculation that the user ended up going to its target will be considered "used".

A preload is considered **unused** if no resource load during the page lifetime [consumed](https://html.spec.whatwg.org/multipage/links.html#consume-a-preloaded-resource) it from the [map of preloaded resources](https://html.spec.whatwg.org/multipage/links.html#map-of-preloaded-resources).

### bfcache handling

Uponk restore from bfcache, counts will not be reset and it's possible for unused preloads/speculations to reduce on second and subsequent navigations away from such pages.

## Security and Privacy Considerations

The above definition for "used" explicitly obfuscates whether a certain speculation was successful or not.
That is important when discussing cross-origin fetches (either prefetch or prerender), as otherwise it can expose cross-origin HTTP response status.

Beyond that, this proposal can potentially make it easier to access same-origin data (e.g. "user navigated to a certain link", "user had certain sections go into the viewport" or "user hovered over something"). This data is already something developers can gather today.

## Considered Alternatives

### PerformanceObserver API

### An imperitive JavaScript API (e.g. document.unusedPreloads)

### Reporting API


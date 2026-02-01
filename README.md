[@yoavweiss](https://github.com/yoavweiss), [@tunetheweb](https://github.com/tunetheweb)

# Speculative load measurement API

## Introduction

Modern web applications can use speculative loading techniques to improve navigation performance.
However, developers currently lack visibility into whether these speculations were actually used, making it hard for them to deploy higher eagerness values than "conservative". Such values can result in wasted user bandwidth and server-side load, and currently developers have no way to weigh the trade-off between those and potential performance gains.

This proposal addresses this by exposing information about unused speculative loads on page dismissal, enabling developers to measure speculation effectiveness and optimize their speculative load strategies, to pick the trade-off that's right for them.

## Goals

1. Enable developers to measure the effectiveness of their speculation rules and preload strategies. 
1. Expose relevant metadata (speculation rules [tags](https://html.spec.whatwg.org/C#prefetch-record-tags), "as" attribute values) to make the above measurement actionable.
1. Avoid exposing cross-origin data.
1. Only expose speculation data after the page starts [unloading](https://html.spec.whatwg.org/C#unload-a-document), to reduce potential confusion.

## Non-Goals


## API Design

We will extend the event object that "pagehide" gets as input to include a `speculations` attribute, which will hold information about unused preloads, prefetches and prerenders.

The information contained would be:
- Preload
  - URL
  - The [`as` attribute](https://html.spec.whatwg.org/#attr-link-as) value, as it can often lead to mismatches and unused preloads
  - The `crossorigin` attribute value, as it can similarly lead to mismatches
- Prefetch
  - URL
  - The relevant [tags](https://html.spec.whatwg.org/C#prefetch-record-tags).
- Prerender
  - URL
  - The relevant [tags](https://html.spec.whatwg.org/C#prefetch-record-tags).

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

A speculation is considered **unused** if it was initiated but the user navigated to a different URL than the speculation target.
The determination is made at pagehide time by comparing all tracked speculations against the URL of the navigation that triggered the page unload.
Whether or not a speculation was successful (e.g. didn't result in a 403) is not relevant. A failed speculation that the user ended up going to its target will be considered "used".

## Security and Privacy Considerations

The above definition for "used" explicitly obfuscates whether a certain speculation was successful or not.
That is important when discussing cross-origin fetches (either prefetch or prerender), as otherwise it can expose cross-origin HTTP response status.

Beyond that, this proposal can potentially make it easier to access same-origin data (e.g. "user navigated to a certain link", "user had certain sections go into the viewport" or "user hovered over something"). This data is already something developers can gather today.

## Considered Alternatives

### PerformanceObserver API

### Reporting API


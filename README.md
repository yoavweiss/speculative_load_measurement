[@yoavweiss](https://github.com/yoavweiss), [@tunetheweb](https://github.com/tunetheweb)

# Speculative load measurement API

## Introduction

Modern web applications can use speculative loading techniques to improve navigation performance.
However, developers currently lack visibility into whether these speculations were actually used, making it hard for them to deploy higher eagerness values than "conservative". Such values can result in wasted user bandwidth and server-side load, and currently developers have no way to weigh the trade-off between those and potential performance gains.

This proposal addresses this by exposing information about unused speculative loads on page dismissal, enabling developers to measure speculation effectiveness and optimize their speculative load strategies, to pick the trade-off that's right for them.

## Use-cases

When experimenting with speculative loads, developers need to be able to answer questions such as:
* Does preloading more/certain resources cause the site's performance metrics to get better or worse?
* How often do my preloaded resources go unused? Which preloaded resource should I stop preloading?
* Is more aggressive prefetching/prerendering worth the cost tradeoff? User performance improves by X%, but server-cost is increased by Y%.
* What links should I stop prefetching/prerendering, as they almost never get used?
* What's the benefit from moving preloads to use Early Hints?

This proposal aims to enable developers answering those questions.

## Goals

1. Enable developers to measure the effectiveness of their speculation rules and preload strategies. 
1. Expose relevant metadata ([speculation rules tags](https://html.spec.whatwg.org/C#prefetch-record-tags), `as` attribute values) to make the above measurement actionable.
1. Only expose speculation data after the page starts [unloading](https://html.spec.whatwg.org/C#unload-a-document), to reduce potential confusion.
1. Only expose data on user actions in the current document (even for cross-origin speculative loads), without exposing cross-origin data.

## Non-Goals
1. Enable developers to detect and handle situations when speculations are unsuccessful.
1. Measurement of prefetched resources (as opposed to prefetched navigations).
   - Speculative resource fetches for the current page are better served with preload, which is covered.
   - For speculative resource fetches for future pages, it's very hard to say with certainty that they were not used, as they may still be used in a future navigation.

## API Design

We will extend the event object that the `pagehide` event gets as input to include a `speculations` attribute, which will hold information about unused preloads, speculation rules navigational prefetches and speculation rules prerenders.

It will have the following shape:
```
{
  preloads: [
    { url: '...', as: '...', crossorigin: '...', earlyhint: true, used: true },
    { url: '...', as: '...', crossorigin: '...', earlyhint: true, used: false },
    // ...
  ],
  navigations: [
    { type: 'prefetch', url: '...', tags: '...', eagerness: '...', used: false },
    { type: 'prerender', url: '...', tags: '...', eagerness: '...', used: true },
    // ...
  ]
}
```

* `URL` - the URL of the resource.
* `as` - The reflected [`as` attribute](https://html.spec.whatwg.org/C#attr-link-as) value, as it can often lead to mismatches and unused preloads
* `crossorigin` - an enum representing the `crossorigin` attribute value, as it can similarly lead to mismatches
* `type` - the type of the speculative fetch. e.g. "prefetch", "prerender" or "prerender-until-script"
* `tags` - The relevant [tags](https://html.spec.whatwg.org/C#prefetch-record-tags).
* `eagerness` - The [eagerness](https://html.spec.whatwg.org/C#speculation-rule-eagerness) of the rule that lead to the speculative load.
* `earlyhint` - whether a preload was delivered using an early hint header.
* `used` - See definition below, under "key concepts".

### Why `pagehide`?

Only on page unload is the final state of preload usage and/or speculations known. `pagehide` is the preferred event to handle over `beforeunload` (which can be cancelled) and `unload` (which has reliability issues for bfcache-eligible pages and [is being deprecated](https://developer.chrome.com/docs/web-platform/deprecating-unload)).

Note that `pagehide` does still not have reliability guarantees, particularly on mobile, if the page is dismissed without being unloaded (e.g. killed while backgrounded). However, the alternative `visibilitychange` event,  while being the last reliable event, does not indicate the page is being navigated away from, and may also be called multiple times as tabs are switched back and forth.

The intent of the API is to provide a means of reasonably measuring preload/speculation usage across a broad set of page loads rather than to provide comprehensive guarantees of reporting for every page load, so this reliability issue is a tradeoff to reduce developer complexity of other alternatives.

### Example Usage

```javascript
window.addEventListener('pagehide', (event) => {
  const { preloads, navigations } = event.speculations;

  // Send unused speculation data to analytics endpoint
  fetch('/analytics/unused-speculations', {
    method: 'POST',
    keepalive: true,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      preloads, navigations
    })
  });
});
```

## Key Concepts

### "Used" Definition

A speculative navigation is considered **unused** if it was initiated but the user navigated to a different URL than the speculation target.
The determination is made at pagehide time by comparing all tracked speculations against the URL of the navigation that triggered the page unload.
Whether or not a speculation was successful (e.g. didn't result in a 403) is not relevant. A failed speculation that the user ended up going to its target will be considered "used".

A preload is considered **unused** if no resource load during the page lifetime [consumed](https://html.spec.whatwg.org/multipage/links.html#consume-a-preloaded-resource) it from the [map of preloaded resources](https://html.spec.whatwg.org/multipage/links.html#map-of-preloaded-resources).

### bfcache handling

Upon restore from bfcache, counts will not be reset and it's possible for unused preloads/speculations to reduce on second and subsequent navigations away from such pages.

Note: This can result in undercounting of used prefetches. We need to figure out what that means.

## Security and Privacy Considerations

The above definition for "used" explicitly obfuscates whether a certain speculation was successful or not.
That is important when discussing cross-origin fetches (either navigational prefetch or prerender), as otherwise it can expose cross-origin HTTP response status.

Beyond that, this proposal can potentially make it easier to access same-origin data (e.g. "user navigated to a certain link", "user had certain sections go into the viewport" or "user hovered over something"). This data is already something developers can gather today.

## Considered Alternatives

### PerformanceObserver API

PerformanceObserver is the natural choice for exposing performance-oriented data.
At the same time, there are a couple of issues with PerformanceObserver in this particular case:
* The information we're interested in here is only known when the page is being dismissed.
* There's no single point in time in which we know that a load went unused before the page dismissal events. So an observer feels like the wrong pattern.
* By default, observers are async. That makes it particularly hard to work with them inside dismissal events, after which the page unloads.

Beyond the above, there's also a concern with exposing some information about speculative loads before we know for sure that they will go unused. That feels like a footgun, and something developers may end up misusing.

### An imperative JavaScript API

An imperative JS API  (e.g. `document.speculations`) would suffer from exposing incomplete information to developers throughout the lifetime of the page. Additionally it is difficult to know when it will have been updated with the future navigation information so will likely only be read on page hide anyway, at which point that event seems a better place for the API.

Finally, one of [the non-goals](#non-goals) is to help react to unsuccessful speculations which may have other concerns if exposed earlier in the page lifecycle (e.g. override user preferences regarding speculations).

### Reporting API

While a Reporting API can be most accurate in terms of the point in time it can send the information (e.g. after the page was already dismissed, and potentially after eviction from the bfcache), the Reporting API presents some challenges to developers, who'd need to set up new collection backends and would need to join the data reported by the API, with other performance data collected through JS.

## Frequently Asked Questions

### Can't you get that information from server side logs?

It is tricky to correlate previous speculative requests with future page loads on the server.

Additionally, many resources or navigations may be served from caches (CDNs, browser caches...etc.) further making this difficult (impossible?) to measure server-side.

### What about `<link rel=prefetch>` and `<link rel=prerender>`?

As detailed in [the non-goals](#non-goals), resource prefetches (as opposed to navigation prefetches) are often for future page views so are explicitly excluded. `<link rel=preload>` should be used for this page view and so is included.

`<link rel=prerender>` is an older API which has been replaced with the Speculation Rules API, and may be deprecated in future, so is excluded. Usage of this API is also low, and likely will drop further once [`prerender_until_script` becomes available by default](https://developer.chrome.com/blog/prerender-until-script-origin-trial?hl=en).

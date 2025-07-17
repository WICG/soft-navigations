# Soft Navigations

## Overview

“Soft navigations” are JS-driven same-document navigations that are using the history API or the new Navigation API, triggered by a user gesture and modifies the DOM, modifying the previous content, as well as the URL displayed to the user.

[PerformanceTimeline#168](https://github.com/w3c/performance-timeline/issues/168) outlines the desire to be able to better report performance metrics on soft navigation. Heuristics for detecting soft navigations can ensure that developers can measure their SPA’s performance metrics, and optimize them to benefit their users.

## Motivation

Why would we want to web-expose soft navigation at all, you ask?

Well, a few reasons:

- Developers would like to attribute various performance entries to specific “soft navigation” URLs. For example, layout shifts caused in one URL can currently be attributed to the corresponding landing page, resulting in mis-attribution and trouble finding the real cause and fixing it.
- Developers would like to receive various “load” performance entries for soft navigations. Specifically, paint timing entries seem desired for such navigations.

From a user's perspective, while they don't necessarily care about the architecture of the site they're visiting, they likely care about it being fast. This specification would enable alignment of the measurements with the user experience, improving the chances of SPA sites being perceived as fast by their users.

## Goals

- Enable measurement of Single-Page app performance in the wild for today's apps.
- Enable such measurement at scale - allowing the team that owns the measurement to be decoupled from the team that owns the app's logic.
- Not rely on developers annotating soft navigations themselves.

## Heuristics

1. A User _initiates_ a soft navigation via an Interaction with the page.

   - For example: clicking on a `<a href>` link, button, submitting a `<form>`, clicking the browser back button (or using gestures), or using keyboard shortcuts.
   - Note: Navigations without Interactions are explicitly not considered.

1. That operation results in an event handler firing.

   - For example: `“click”`, `“navigate”`, `"keydown"` events, etc.

1. We establish an Interaction "Context", wrapping those event handlers, and which _persists across the scheduling of new tasks_.

   - For example: `setTimeout()` or `fetch()` calls, `await` keyword, etc.
   - See related proposal for explicit `AsyncContext` API.

1. We observe specific effects which modify the document's location or DOM structure.

   - For example, via `history.pushState()`.
   - For example, by adding new child elements, or updating specific attributes of existing elements (`<img src="...">`).

1. When such effects happen, if the actively running Task has an active Interaction "Context", we can _attibute_ the cause of the effect to it.

   - Each Context observes the set of effects it caused, looking to meet specific necessary criteria.

1. We _attribute_ the page updates with the related contentful paints that follow and report then to the performance timeline.

   - Interaction attributable paints are reporting via a new `PerformanceEntry` called `InteractionContentfulPaint` (ICP).
   - These are observed for any new Element paint that belongs to a Container (aka Component) that was modified by the Interaction.
   - See related proposal for explicit `ContainerTiming` API.
   - Note: `InteractionContentfulPaint` currently reports only new largest element paint candidates, similar to LCP, but it might also report all updated area, like `ContainerTiming`.

1. When all criteria are met: A real user Interaction leads to a document location change and a substantial contentful paint update that is directly attributable to that interaction: that is a Soft Navigation.

   - This will be reported via a new `PerformanceEntry` called `SoftNavigationEntry`.
   - Among other metadata, it will include a recomended new timeOrigin (via `startTime`), and provide a new `navigationId` value, which can be used to help group and measure other `PerformanceEntry` on the performance timeline.
   - The `SoftNavigation` entry also acts as the FCP measure for the navigation.
   - ICP entries act as LCP candidates for the navigation.

### [Task attribution](https://bit.ly/task-attribution)

The above heuristics rely on the ability to keep track of tasks and their provenance. We need to be able to tell that a certain task was posted by another, and be able to create a causality chain between DOM dirtying and URL modifications to the event handler that triggered the soft navigation.

## Proposed API shape

### SoftNavigationEntry

```
SoftNavigationEntry : PerformanceEntry {
}
```

The inheritance from `PerformanceEntry` means that the entry will have `startTime`, `name`, `entryType` and `duration`:

- `startTime` is defined as the time in which the user's interaction event processing ended, or the time in which the "navigate" event processing ended, whichever's first.
- `name` is the URL of the history entry representing the soft navigation.
- `entryType` is "soft-navigation".
- `duration` is the time difference between the point in which a soft navigation is detected and the `startTime`.

### InteractionContentfulPaint

```
InteractionContentfulPaint : PerformanceEntry {
    readonly attribute DOMHighResTimeStamp renderTime;
    readonly attribute DOMHighResTimeStamp loadTime;
    readonly attribute unsigned long long size;
    readonly attribute DOMString id;
    readonly attribute DOMString url;
    readonly attribute Element? element;
}
```

Note: This currently mirrors the shape of `LargestContenfulPaint` and matches the same semantics. This will likely change.

## Examples

### Observing Soft Navigations

To observe the stream of new soft-navigations, you can use a `PerformanceObserver`:

```js
new PerformanceObserver((list) => {
  for (let entry of list.getEntries()) {
    // ...
  }
}).observe({
  type: "soft-navigation",
  buffered: true, // Optional
});
```

To list all the existing (buffered) entries so far, you can use `getEntriesByType`:

```js
const soft_navs = performance.getEntriesByType("soft-navigation");
```

That would give them a list of past and future soft navigations they can send to their server for processing.

They would be able to also get soft navigations as they come (similar to other performance entries):

```javascript
const soft_navs = [];
new PerformanceObserver((list) => soft_navs.push(...list.getEntries())).observe(
  { type: "soft-navigation" }
);
```

Or to include past soft navigations:

```javascript
const soft_navs = [];
new PerformanceObserver((list) => soft_navs.push(...list.getEntries())).observe(
  { type: "soft-navigation", buffered: true }
);
```

### Correlating performance entries with a soft navigation

For that developers would need to collect `soft_navs` into an array as above.
Then they can, for each entry (which can be LCP, FCP, or any other entry type), find its corresponding duration as following:

```javascript
const icp_entries = [];
new PerformanceObserver((list) =>
  icp_entries.push(...list.getEntries())
).observe({
  type: "interaction-contentful-paint",
  includeSoftNavigationObservations: true,
});

for (icpEntry of icp_entries) {
  // Find the soft navigaton entry matching on `navigationId`:
  const navEntry = soft_navs.filter(
    (navEntry) => navEntry.navigationId == icpEntry.navigationId
  )[0];

  const lcp_candidate_duration = icpEntry.startTime - navEntry.startTime;
  // ...
}
```

## Required spec changes

- We need to add `PerformanceObserverInit` option named "includeSoftNavigationObservations", that will indicate that post-soft-navigation ICP entries should be observed.

Note: With the new ICP entry replacing LCP, this may change.

## Privacy and security considerations

This API exposes a few novel paint timestamps that are not available today, after a soft navigation is detected: the first paint, the first contentful paint and the largest contentful paint.
It is already possible to get some of that data through Element Timing and requestAnimationFrame, but this proposal will expose that data without the need to craft specific elements with the `elementtiming` attribute.

### Mitigations

- The LCP timestamp is subject to the same constraints as current LCP entries, and doesn't expose cross-origin rendering times without an explicit opt-in.
- The FCP timestamp doesn't necessary waits until a cross-origin image is fully loaded in order to fire, minimizing the cross-origin information exposed.
- Soft navigations detected are inherently user-driven, preventing programatic scanning of markup permutations and their impact on paint timestamps.
- Soft Navigation detection can be time limited, further limiting the scalability of information exposure.

Given the above mitigations, attacks such as [history sniffing attacks](https://krebsonsecurity.com/2010/12/what-you-should-know-about-history-sniffing/) are not feasible, given that `:visited` information is not exposed.
`:visited` only modifications are not counting as DOM modifications, so soft navigations are not detected. Whenever other DOM modifications are included alongside visited changes, the next paint would include both modifications, and hence won't expose visited state.

Furthermore, cross-origin imformation about images or font resources is not exposed by Soft Navigation LCP, similarly to regular LCP.

## Open Questions for future enhancements

- Do we need to define FCP/LCP as contentful paints that are the result of the soft navigation?
- Could we augment the heuristic to take both DOM additions and removals into account?
  - Currently, interactions such as Twitter/Gmail's "compose" button would be considered soft navigations, where one could argue they are really interactions.
  - A heuristic that requires either a modification of existing DOM nodes or addition _and_ removal of nodes may be able to catch that, without increasing the rate of false negatives.
- `<your questions here>`

## Considered alternatives

A few notes regarding heuristic alternatives:

- We considered using only semantic elements, but it seems to not match current real-world practices.
- We considered limiting DOM modifications to specific DOM elements or some other heuristic regarding "meaningful" DOM modifications. We haven't seen a necessity for this in practice.
- Finally, we could consider limiting the amount of soft navigation detected in a certain timeframe (e.g. X per Y seconds), if we'd see that some web applications detect an excessive amount of soft navigations that don't correspond to the user experience.

## I want to take this for a spin!!

I like how you're thinking!

You can do that by:

- Joining the [Origin Trial](https://developer.chrome.com/origintrials/#/trials/active), and [enabling it on your site](https://developer.chrome.com/en/docs/web-platform/origin-trials/).
- Alternatively, you can:
  - [enable "Experimental Web Platform features"](chrome://flags/#enable-experimental-web-platform-features) in Chrome
  - Browsing to the site you want to test!
  - Opening the devtools' console
  - Looking for "A soft navigation has been detected" in the console logs
  - Alternatively, running the example code above in your console to observe `SoftNavigationEntry` entries

The Chrome team [have published an article about its implementation](https://developer.chrome.com/blog/soft-navigations-experiment/), and how developers can use this to try out the proposed API to see how it fits your needs.

And remember, if you find bugs, https://crbug.com is the best way to get them fixed!

# FAQs

## Should this rely on the [Navigation API](https://html.spec.whatwg.org/multipage/nav-history-apis.html#navigation-api)?

If this effort were to rely on the Navigation API, that would mean that it can only cover future web apps, or require web apps to completely rewrite their routing libraries in order to take advantage of Soft Navigation measurement.
That would go against the goal of being able to measure such navigations at scale.

On top of that, the Navigation API does not make any distinction between "real" navigations and interactions, so even if we were to rely on the Navigation API, extra heuristics would still be needed.

With that said, this effort works great with the Navigation API, as well as with the older history API.

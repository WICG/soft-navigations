# Soft Navigations

## Overview

“Soft navigations” are user-initiated but JS-driven same-document navigations.

The Soft Navigation API considers a "soft navigation" when the following occurs:

- A user-based interaction occurs (URL updates without a user interaction don't count)
- … which results in a DOM modification and a paint
- … and a URL update occurs, which changes the history state

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

## Processing Model

1. A User _initiates_ a soft navigation via an Interaction with the page.

   - For example: clicking on a `<a href>` link, button, submitting a `<form>`, clicking the browser back button (or using gestures), or using keyboard shortcuts.
   - Note: Navigations without Interactions are explicitly not considered.

1. That operation results in an event handler firing.

   - For example: `“click”`, `“navigate”`, `"keydown"` events, etc.

1. We establish an Interaction "Context", wrapping those event handlers, and which also _persists across the scheduling of new tasks_.

   - For example: `setTimeout()` or `fetch()` calls, `await` keyword, etc.
   - See related proposal for explicit `AsyncContext` API.

1. We observe certain direct modifications to the document structure.

   - For example, via `history.pushState()`.
   - For example, via `element.appendChild()`, or changes to attributes i.e. `<img src="...">`.

1. When such effects happen, if the actively running Task was scheduled with an active Interaction "Context", we can _attribute effects_ to that context.

   - Thus, each Context is assigned the set of effects it caused. The "Soft navigation heuristics" is really just a small set of necessary effects that a single Context must have.

1. We also _attribute contentful paints_ for the parts of the dom that were directly modified with the context, and report them to the performance timeline.

   - `InteractionContentfulPaint` entries are reported, for any Element that was directly modified, or was _part of a Container_ that was directly modified by the Interaction.

1. When all criteria are met-- A real user Interaction leads to a document location change and a substantial contentful paint update-- that becomes a Soft Navigation.

   - `SoftNavigation` entries are reported.

###

### [Task attribution](https://bit.ly/task-attribution)

The above heuristics rely on the ability to keep track of tasks and their provenance. We need to be able to tell that a certain task was posted by another, and be able to create a causality chain between DOM dirtying and URL modifications to the event handler that triggered the soft navigation.

## Proposed API shape

### PerformanceEntry

```
PerformanceEntry {
    readonly attribute unsigned long long navigationId;
}
```

- Extend `PerformanceEntry` to add `navigationId`.
  - This is a pseudorandom number which increments with each new interaction. It is exposed to many different entry types and helps group (or slice) the single performance timeline by navigations.

### SoftNavigationEntry

```
SoftNavigationEntry : PerformanceEntry {
}

SoftNavigationEntry includes PaintTimingMixin;
```

The inheritance from `PerformanceEntry` means that the entry will have `startTime`, `name`, `entryType` and `duration`:

- `startTime` is a recomended new timeOrigin for this navigation. It is the earlier value of:
  - the user's interaction event's `processingEnd`, or
  - the URL was explicitly changed.
- `name` is the URL of the history entry representing the soft navigation.
- `entryType` is `"soft-navigation"`.
- `duration` is the time difference between the point in which a soft navigation is detected and the `startTime`.
- `SoftNavigationEntry` also includes `PaintTimingMixin` which exposed `paintTime` and `presentationTime` and which represent the First Contentful Paint (FCP) for this navigation.

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

- The `InteractionContentfulPaint` entries report any new Element paint that _belongs to a Container_ that was modified by the Interaction.
  - See related proposal for explicit `ContainerTiming` API.
- `InteractionContentfulPaint` (ICP) entries act as `LargestContenfulPaint` (LCP) candidates for the navigation.
  - Note: this entry currently perfectly mirrors the shape of `LargestContenfulPaint`, but might change to extend it.
  - For example: `InteractionContentfulPaint` currently reports only new largest element paint candidates, like LCP, but it might change to also report each updated paint area via `size`, like `PerformanceContainerTiming`.

### Required spec changes

- We need to add a `PerformanceObserverInit` option named `"includeSoftNavigationObservations"`.
  - This flag is used to mark that `PerformanceEntry` should be reported with NavigationId.
  - This is required for `InteractionContentfulPaint` entries.

```
dictionary PerformanceObserverInit {
  boolean includeSoftNavigationObservations;
};
```

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

## Should this rely on the Navigation API)?

The [Navigation API](https://html.spec.whatwg.org/multipage/nav-history-apis.html#navigation-api) consolidates the many methods to initiate or observe same document history navigations. It also provides a mechanism to `intercept()` semantic navigations (such as `<a href>` clicks or `<form action>` submits) and provide custom behaviours.

However, if this effort were to _require_ that websites opt to use this new `intercept()` feature in order to receive Soft Navigation measurement, that would mean that it would only cover future web apps, or require web apps to completely rewrite their routing libraries.

That would go against the goal of being able to measure such navigations at scale.

On top of that, the Navigation API does not make any distinction between "real" navigations and interactions. So although the Navigation API does provide useful mechanisms and simplications for observing same document history changes, extra heuristics are still needed.

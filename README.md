# Soft Navigations and Interaction Contentful Paint

## Authors:

- Michal Mocny (Google)
- Scott Haseley (Google)
- Yoav Weiss (Former Editor - Shopify)

## Participate
- [Issue tracker](https://github.com/WICG/soft-navigations/issues)
- [Specification](https://wicg.github.io/soft-navigations/)

## Introduction

Modern web applications often dynamically update content in response to user interactions, without performing a full cross-document navigation to do so.  The existing Web Performance Timeline APIs do not provide a mechanism to measure the performance of such user experiences.

This repository hosts a specification for two new `PerformanceEntry` types:

1. **`InteractionContentfulPaint`**: Reports on new contentful paints that are initiated by, and attributed to, user interactions.
    - This entry represents all contentful updates to the page, modeled on concepts from [Container Timing](https://github.com/WICG/container-timing).
    - This entry also reports a nested `LargestContentfulPaint` entry, representing the single largest element rendered as a result of that interaction.
    - Note: This measures all interactions, even if they do not also result in soft navigations, and so is useful for general UI responsiveness measures.

2. **`PerformanceSoftNavigation`**: Reports on user-initiated same-document navigations.
    - This entry primarily serves to help "slice" the performance timeline, group existing `PerformanceEntry` entries, and give them a URL to attribute to.
    - This entry reports the First Contentful Paint of the soft navigation (via `PaintTimingMixin`), and defines a new `timeOrigin` for subsequent entries (via its `startTime`).
    - Note: Unlike the Navigation API, this API is designed to associate the initial interaction with a same-document navigation and a first contentful paint. All pieces are required.

This specification also defines an extension to all existing `PerformanceEntry` types:
- Adds a `navigationId` attribute, which can be used to "slice" the performance timeline data into useful sub-parts.
- A `PerformanceSoftNavigation` becomes one mechanism for slicing, though other page lifecycle events (i.e., `pageshow` for bfcache restorations, etc.) are also common reasons.

Finally, this specification also proposes several modifications to existing specifications to support these new APIs.

## User-Facing Problem

The web [Performance Timeline](https://www.w3.org/TR/performance-timeline/) and [related specifications](https://www.w3.org/groups/wg/webperf/publications/) define a rich set of capabilities for measuring the performance of pages. These help developers monitor, understand, and improve user experience.

From these primitives, an interoperable set of metrics is defined, such as [First Contentful Paint (FCP)](https://developer.mozilla.org/en-US/docs/Glossary/First_contentful_paint), [Largest Contentful Paint (LCP)](https://developer.mozilla.org/en-US/docs/Glossary/Largest_contentful_paint), and [Interaction to Next Paint (INP)](https://developer.mozilla.org/en-US/docs/Glossary/Interaction_to_next_paint).

However, those specifications, and the metrics defined in terms of them, are currently tied to cross-document navigations, aka "hard" page loads.  I.e., you only get paint timings for the initial page load, and all other timings are reported with timestamps relative the original navigation start, and typically attributed to the initial document URL.

Yet, many modern web applications will not always choose to "hard" navigate between distinct pages on every interaction. Sites might instead only partially update existing page contents in response to user interactions. Some sites might even be designed as [Single Page Applications](https://developer.mozilla.org/en-US/docs/Glossary/SPA), though modern practice is to leverage a mixture of cross-document and same-document interactions/navigations.

**Problem:** Such sites currently do not fully benefit from the existing Performance Timeline APIs.

### Motivating Scenario

- A user clicks a product link.
- A `click` handler initiates a network `fetch()`.
- The fetch response triggers a callback that:
  - Dynamically injects new content into the DOM, and
  - Updates the URL using history APIs.

To the user, this feels exactly like a "navigation."
To the performance timeline, the new URL is irrelevant, and the eventual paint updates are unmeasured.

### Goals

1. **Attribute paints to user interactions**: Observe dynamic page updates, and attribute their contentful paints to initiating user interactions.
   - See: [Solving Goal 1: Observing interaction contentful paints](#solving-goal-1-observing-interaction-contentful-paints)
2. **Measure loading performance**: Measure loading performance (FCP, LCP) for same-document ("soft") navigations.
   - See: [Solving Goal 2: Observing soft navigations](#solving-goal-2-observing-soft-navigations)
3. **Observe and group timeline entries**: Group existing performance entries by same-document navigations for improved URL attribution.
   - See: [Solving Goal 3: Mapping existing performance timeline entries to navigations](#solving-goal-3-mapping-existing-performance-timeline-entries-to-navigations)
4. **Leverage existing APIs**: Leverage and extend existing Performance APIs and platform capabilities (e.g., Navigation API, Event Timing, AsyncContext, etc.) in an integrated way.

### Non-goals

- Fully replacing the standard Navigation API or custom application-level routers.

## User research

No formal user research has been conducted for this proposal yet.

Instead, we investigated existing techniques and best practices used by web frameworks and client side routers to measure and observe soft navigations. 

The proposed solution was evaluated, and evolved, through several rounds of Origin Trial feedback and developer testing in Chromium.

## Proposed Approach

The APIs proposed in the specifications contained within this repository create an elegant mechanism to address this existing gap:

- Same-document navigations are just interactions that:
  - Dynamically update the contents of the page, and
  - Update the application's same-document history entry.

By measuring the "loading performance" of all interactions, summarized into a single nested "LCP" for each interaction, and by observing same-document history changes initiated by those same interactions — we can define and measure soft navigations and their subsequent loading performance (i.e., "soft" LCP).

### Detailed Design

This specification mostly leverages and brings together several existing web platform capabilities, as well as a few new nascent feature incubations:

- **Event Timing**: User interactions (like clicks) are already identified and assigned an `interactionId`. We extend Event Timing to add support for `navigate`, `popstate`, and `hashchange` events.
- **AsyncContext**: Each observed interaction creates a unique `InteractionContext`, which is stored in an internal `AsyncContext.Variable`. This gets propagated through asynchronous operations (like `fetch()` or `setTimeout`), ensuring that the eventual effects of that interaction can be attributed back to the original user interaction.
- **HTML** and **DOM**: Whenever DOM modifications occur (e.g., `appendChild`, `innerHTML`, `style` or `src` attributes, etc.), and the modification is from a task that is associated with an `InteractionContext` (via `AsyncContext`), we "mark" that part of the DOM as being associated with that interaction.
- **Paint Timing** and **Largest Contentful Paint**: We extend these APIs to define how to "reset" paint timings (after DOM modifications) and how to map specific element paints to interactions.
- **Container Timing**: The concept of "marking" nodes in the DOM, then later mapping element paints to these, borrows from concepts that are part of the proposed Container Timing API. (Directly integrating with and exposing Container Timing IDL attributes on the `InteractionContentfulPaint` is an aspirational future goal).
- **Navigation API**: Although the use of the new Navigation API is not required by developers, it provides a more robust and consistent way to define same-document navigations, and their attributes.

### Dependencies on non-stable features

This proposal has a dependency on the following non-stable or proposed web platform features:

- **AsyncContext**: This proposal depends on `AsyncContext` for propagating the `InteractionContext` across asynchronous boundaries (such as network requests or timers).
  - Note: The current experimental Chromium implementation does not literally use the proposed public `AsyncContext` JavaScript API, but instead uses an internal Chromium mechanism known as "Task Attribution." Task Attribution is expected to power the public `AsyncContext` API when implemented, and the two systems are expected to stay aligned.
- **Container Timing**: The current specification does not directly depend on the Container Timing proposal. However, the proposed solution has overlapping concepts (specifically, the idea of "marking" DOM nodes as "containers" for aggregating together contentful paints to any elements which paint inside them), and we expect to integrate/share some of the paint timing attribution parts over time. The IDL structure does not yet expose any of Container Timing, but the current API has been designed so that future extensions to support it will be possible, if desired.

### Solving Goal 1: Observing interaction contentful paints

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const {
      startTime,
      duration,
      interactionId,
      largestContentfulPaint,
    } = entry;

    console.log(
      "[ICP] interactionId:", interactionId,
      "startTime:", startTime,
      "duration:", duration,
      "LCP element (so far):", largestContentfulPaint.element,
      "LCP size (so far):", largestContentfulPaint.size
    );
  }
}).observe({
  type: "interaction-contentful-paint",
  buffered: true // Optional
});
```

### Solving Goal 2: Observing soft navigations

To observe the stream of new soft navigations, you can either use a `PerformanceObserver`:

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const {
      startTime,
      duration,
      interactionId,
      navigationId,
    } = entry;
    const url = entry.name;

    // Optional: Retrieve the largest ICP for this soft navigation so far.
    // Note: This keeps updating as the page loads beyond FCP, so you can read the final value when you are ready to report/beacon.
    const icpEntry = entry.getLargestInteractionContentfulPaint();
    const lcpElement = icpEntry?.largestContentfulPaint?.element;

    console.log(
      "[SoftNav] interactionId:", interactionId,
      "startTime:", startTime,
      "url:", url,
      "fcp:", duration,
      "lcp element (so far):", lcpElement
    );
  }
}).observe({
  type: "soft-navigation",
  buffered: true, // Optional
});
```

Or, use `performance.getEntriesByType()`:

```js
const soft_navs = performance.getEntriesByType("soft-navigation");
```

Note: The latter is limited by the global buffer size for this entry type, so using a `PerformanceObserver` is recommended.

> [!NOTE]
> **Retrieving LCP from soft navigations (and its tradeoffs):**
> Once a soft navigation is detected and emitted as a `soft-navigation` PerformanceEntry, developers often want to report its final/largest contentful paint (LCP) value to their analytics beacon. To make this convenient, the `PerformanceSoftNavigation` entry provides a `getLargestInteractionContentfulPaint()` getter method.
> 
> This method returns the largest `InteractionContentfulPaint` observed during the soft navigation's interaction context. This allows developers to keep a reference to the soft navigation entry as it is emitted, wait for page unload or other beaconing criteria, and report the last value of LCP directly from this nested getter.
> 
> However, doing so has some tradeoffs: if you wait too long (e.g. at page unload), the nested LCP element reference (`largestContentfulPaint.element`) may have already been garbage-collected, removed from the DOM, or detached, returning `null`. For real-time tracking, element inspection, or robust bookkeeping, developers should instead subscribe to `interaction-contentful-paint` entries for real-time observation.

### Solving Goal 3: Mapping existing performance timeline entries to navigations

All `PerformanceEntry` types can be mapped to a navigation using a `navigationId` value.

From this, you can extract:
- a "timeOrigin", via `startTime` (or `activationStart`)
- the initial URL, via `name`

```js
function getNavigationEntry(navigationId) {
    const navs = [
        performance.getEntriesByType('navigation')[0],
        ...performance.getEntriesByType('soft-navigation'),
    ];
    return navs.find(entry => entry.navigationId === navigationId);
}
```

Note: This specification does not define it, but it would be a useful future extension to also add (e.g., bfcache restorations) to this list.

## Alternatives considered

### Alternative: Strong baked-in heuristics
The initial solution for detecting soft navigations relied on stronger, baked-in heuristics. For example:

- Detecting a "soft" navigation required a certain fraction of the overall page to be modified.
- A "soft" LCP entry was only reported if/after a soft navigation was emitted.
- Certain URL change patterns and restrictions were applied (i.e., push only, no support for `hashchange` only).

These heuristics were meant to approximate existing cross-document navigations, and support use cases that blended both hard and soft navigation data. There is also some value in having a standard set of criteria that are baked in and consistently applied across sites.

However, these self-imposed limitations (heuristics) also reduced the utility of the feature for many real-world use cases, and reduced the quality of the performance data for many sites. It also made the implementation more complex, rather than easier. Over time, the feedback from developers was to relax these heuristics and provide a simpler, more flexible solution.

The single biggest change was to decouple the task of reporting `InteractionContentfulPaint` from reporting `PerformanceSoftNavigation`. This simplifies implementation complexity, and it has proved useful as a general-purpose tool for measuring the performance of user interactions, even when those interactions don't result in a navigation of any kind.

Some fundamental requirements do remain:

- You must initiate the navigation via a trusted user-generated interaction (e.g., click, tap, etc.).
- At least some contentful update to the page must be observed.

But the remaining "heuristics" are left to the developer to enforce. For example, the `navigationType` and URL are exposed, so the developer may filter or group as desired.

### Alternative: Do not rely on AsyncContext
Another alternative explored was to observe effects such as interactions, same-document navigations, and paints, just as global effects, and then tie them together with a simple timer—i.e., via [Transient User Activation](https://developer.mozilla.org/en-US/docs/Glossary/Transient_activation).

However, this created a problem: although most interactions provide a fast response, performance data from the field is *most useful* for finding slow outliers. We know from aggregate field data that navigation loading surprisingly often takes between 4 and 10 seconds on slow devices. This suggests that any timer-based cut-off value should not be less than 10 seconds, and potentially much larger.

But users are typically interacting at least once every few seconds. Thus, at least after the initial interaction, a page would nearly always be in a state of having an "active" interaction.

### Alternative: Using only semantic elements
We considered observing only changes to specific semantic elements (i.e., `<main>` or `<article>` sections of the page), but this does not seem to match current real-world practices.

### Alternative: Rate limiting soft navigations
We could consider limiting the amount of soft navigations detected in a certain timeframe (e.g., X per Y seconds), if we'd see that some web applications detect an excessive amount of soft navigations that don't correspond to the user experience.

## Alternatives: Previous API shape

(Note: This section is incomplete.)

- A previous version tried to re-emit the `LargestContentfulPaint` entryType directly, with a "soft" mode filter.
- Then, we created a unique entry type, but copied all existing LCP attributes.
- Finally, we create a new unique entry type, modelled on "container timing" ideas, and embedded a nested `LargestContentfulPaint` inside. This is the shape of the API today.

## Accessibility, Internationalization, Privacy, and Security Considerations

Exposing these entries does not introduce significant novel privacy risks.

- Interactions are "observed" and reported using the same criteria and conditions as existing ones (via Event Timing).
- Interaction-attributed paint timings follow the same security constraints as existing Paint Timing and LCP (e.g., cross-origin image opt-in).
- Observation conditions are limited to trusted user interactions, preventing programmatic observation of document updates.

For a detailed analysis, see the W3C TAG [Security & Privacy Self-Review Questionnaire (SP-questions.md)](SP-questions.md) and the [Security & Privacy section](https://wicg.github.io/soft-navigations/#priv-sec) of the specification.

## Stakeholder Feedback / Opposition

- **Chromium**: Positive (implementing experimental support)
- **Gecko**: No public signals yet
- **WebKit**: No public signals yet

## References & acknowledgements

Many thanks for valuable feedback and advice from the members of the [W3C Web Performance Working Group](https://www.w3.org/groups/wg/webperf) and all of the [contributors to this repository](https://github.com/WICG/soft-navigations/graphs/contributors).

Thanks to the following proposals, projects, and specifications for their work on related problems that influenced this proposal:
- [Event Timing API](https://w3c.github.io/event-timing/)
- [Paint Timing API](https://w3c.github.io/paint-timing/)
- [Largest Contentful Paint API](https://w3c.github.io/largest-contentful-paint/)
- [Navigation API](https://wicg.github.io/navigation-api/)
- [AsyncContext TC39 Proposal](https://github.com/tc39/proposal-async-context)
- [Container Timing Proposal](https://github.com/WICG/container-timing)

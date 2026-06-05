# Soft Navigations and Interaction Contentful Paint

## Spec link

https://wicg.github.io/soft-navigations/

## Overview

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

- Add a `navigationId` attribute, which can be used to "slice" the performance timeline data into useful sub-parts.
- A `PerformanceSoftNavigation` would become one mechanism for slicing, though other page lifecycle events (i.e., `pageshow` for bfcache restorations, etc.) are also common reasons.

Finally, this specification also proposes several modifications to existing specifications to support these new APIs.

## Motivation

The web [Performance Timeline](https://www.w3.org/TR/performance-timeline/) and [related specifications](https://www.w3.org/groups/wg/webperf/publications/) define a rich set of capabilities for measuring the performance of pages. These help developers monitor, understand, and improve user experience.

From these primitives, an interoperable set of metrics is defined, such as [First Contentful Paint (FCP)](https://developer.mozilla.org/en-US/docs/Glossary/First_contentful_paint), [Largest Contentful Paint (LCP)](https://developer.mozilla.org/en-US/docs/Glossary/Largest_contentful_paint), and [Interaction to Next Paint (INP)](https://developer.mozilla.org/en-US/docs/Glossary/Interaction_to_next_paint).

However, those specifications, and the metrics defined in terms of them, are currently defined in terms of cross-document navigations, aka "hard" page loads.

Yet, many modern web applications will not always choose to "hard" navigate between distinct pages on every interaction. Sites might instead only partially update existing page contents in response to user interactions. Some sites might even be designed as [Single Page Applications](https://developer.mozilla.org/en-US/docs/Glossary/SPA), though modern practice is to leverage a mixture of cross-document and same-document interactions/navigations.

**Problem:** Such sites currently do not fully benefit from the existing Performance Timeline APIs.

**Example Scenario:**
- A user clicks a product link.
- A `click` handler initiates a network `fetch()`.
- The fetch response triggers a callback that:
  - Dynamically injects new content into the DOM, and
  - Updates the URL using history APIs.

To the user, this feels exactly like a "navigation."

To the performance timeline, the new URL is irrelevant, and the eventual paint updates are unmeasured.

## Proposed Solution

The APIs proposed in the specifications contained within the repository create an elegant mechanism to address this existing gap:

- Same-document navigations are just interactions that:
  - Dynamically update the contents of the page, and
  - Update the application's same-document history entry.

By measuring the "loading performance" of all interactions, summarized into a single nested "LCP" for each interaction, and by observing same-document history changes initiated by those same interactions — we can define and measure soft navigations and their subsequent loading performance (i.e., "soft" LCP).

### How it Works

This specification mostly leverages and brings together several existing web platform capabilities, as well as a few new nascent feature incubations:

- **Event Timing**: User interactions (like clicks) are already identified and assigned an `interactionId`. We extend Event Timing to add support for `navigate`, `popstate`, and `hashchange` events.
- **AsyncContext**: Each observed interaction creates a unique `InteractionContext`, which is stored in an internal `AsyncContext.Variable`. This gets propagated through asynchronous operations (like `fetch()` or `setTimeout`), ensuring that the eventual effects of that interaction can be attributed back to the original user interaction.
- **HTML** and **DOM**: Whenever DOM modifications occur (e.g., `appendChild`, `innerHTML`, `style` or `src` attributes, etc.), and the modification is from a task that is associated with an `InteractionContext` (via `AsyncContext`), we "mark" that part of the DOM as being associated with that interaction.
- **Paint Timing** and **Largest Contentful Paint**: We extend these APIs to define how to "reset" paint timings (after DOM modifications) and how to map specific element paints to interactions.
- **Container Timing**: The concept of "marking" nodes in the DOM, then later mapping element paints to these, borrows from concepts that are part of the proposed Container Timing API. (Directly integrating with and exposing Container Timing IDL attributes on the `InteractionContentfulPaint` is an aspirational future goal).
- **Navigation API**: Although the use of the new Navigation API is not required by developers, it provides a more robust and consistent way to define same-document navigations, and their attributes.


## Examples

### Observing `PerformanceSoftNavigation`

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

### Observing `InteractionContentfulPaint`

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

### Mapping existing entries to navigations

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

### Putting it all together and making it pretty

Try [this example JS snippet](https://github.com/mmocny/mmocny.github.io/blob/main/snippets/InteractionsAndNavigations.js).

## Privacy and Security

Exposing these entries does not introduce significant novel privacy risks.

- Interactions are "observed" and reported using the same criteria and conditions as existing ones (via Event Timing).
- Interaction-attributed paint timings follow the same security constraints as existing Paint Timing and LCP (e.g., cross-origin image opt-in).
- Observation conditions are limited to trusted user interactions, preventing programmatic observation of document updates.

For a detailed analysis, see the [Security & Privacy section](https://wicg.github.io/soft-navigations/#priv-sec) of the specification.

## Considered alternatives

(Note: This section is incomplete. Leaving the original text from an initial explainer, below; however, many alternatives were explored over the life of the feature development.)

- We considered using only semantic elements, but it seems to not match current real-world practices.
- We considered limiting DOM modifications to specific DOM elements or some other criteria regarding "meaningful" DOM modifications. We haven't seen a necessity for this in practice.
- Finally, we could consider limiting the amount of soft navigations detected in a certain timeframe (e.g. X per Y seconds), if we'd see that some web applications detect an excessive amount of soft navigations that don't correspond to the user experience.

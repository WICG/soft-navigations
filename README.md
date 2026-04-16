# Soft Navigations and Interaction Contentful Paint

## Overview

The web performance timeline has an existing rich set of capabilities for measuring the performance of page loads that help developers monitor, understand, and improve user experience.  For example: First Contenful Paint (FCP) and Largest Contentful Paint (LCP) are an interoperable set of metrics for reporting on these experiences across browsers.

However, modern web applications often dynamically update contents of the document in response to user interactions without performing a full cross-document navigation-- no new page load.  Thus, they do not benefit from many of the existing web performance timeline features, which only report performance of cross-document page loads.

This is especially true for apps built using JavaScript driven, client-side component rendering frameworks.

This repository hosts a specification for two new PerformanceEntry types:

1. **`InteractionContentfulPaint`**: Reports contentful paint updates within the same document that are initiated by user interactions (similar to LCP).

2. **`SoftNavigationEntry`**: Reports user-initiated same-document navigations (an alternative to the Navigation Timing API).

...as well as modifications to existing specifications to support these use cases, such as adding a `navigationId` to all PerformanceEntry types.

## Motivation

Existing web performance metrics like **Largest Contentful Paint (LCP)**, **Interaction to Next Paint (INP)** and **Cumulative Layout Shift (CLS)** leave a gap in measuring dynamic page updates:

- **LCP** only measures the initial page load. Subsequent "soft" navigations in an SPA do not currently trigger new LCP entries.
- **CLS** measures layout instability across the entire page lifespan. However, without soft navigation boundaries, it is difficult to attribute layout shifts to specific user journeys based on specific interactions and resulting URL changes.
- **INP** measures the latency of the immediate visual feedback of user interactions, but does not capture any asynchrnously scheduled subsequent rendering of rich content updates (e.g., a product page loading after network response), which is an equally important part of user experience.  Similar to CLS, INP also measures across the entire page lifespan, but does not typically attribute interactions to specific URLs.

**Example Scenario:** A user clicks a product link in a Single Page Application (SPA). A `click` handler initiates a network `fetch()`. When the response arrives, a callback dynamically injects the new content into the DOM and updates the URL.
- To the user, this is a navigation.
- To existing metrics, the "paint" happens long after the interaction is "over."
- To the performance timeline, the new URL is irrelevant, and many RUM products continue to beacont to the initial page.

The InteractionContentfulPaint specification bridges that gap by attributing the late-arriving paint back to the initiating click, and the SoftNavigation specification assists group existing performance entries for improved URL attribution.

## How it Works

This specification brings together several web platform capabilities to measure dynamic page updates:

- **Event Timing**: User interactions (like clicks) are identified and assigned an `interactionId`.  We also extend Event Timing to add support for `navigate`, `popstate`, and `hashchange` events.
- **AsyncContext**: Each interaction is assigned a new `InteractionContext`. This context is automatically propagated through asynchronous operations (like `fetch()` or `setTimeout`), ensuring that the eventual effects can be attributed back to the original user interaction.
- **Container Timing**: This specification leverages some parts of the experimental Container Timing specification to help track contentful paints within the DOM subtrees that are marked as "container roots" and are attributed to an interaction context.
- **Navigation API**: Although the use of the new Navigation API is not required by developers, it provides a more robust and consistent way to define same document navigations, and their attributes.

Leveraging these primitives:

- Whenever Event Timing observes a new Event dispatch that is considered an Interaction, we create an `InteractionContext`.
- We save this `InteractionContext` into an internal `AsyncContext.Variable` which propogates across asynchronous task scheduling.
- We observe structural modifications to the current document (such as adding new child nodes to an existing node, or updating existing node content or attributes).
  - If an `InteractionContext` is available for the current task when this happens, we mark the modified DOM subtree as belonging to a "container" which is specific to this `InteractionContext`.
  - We do so as sparsely and lazily as possible, such that only new unique DOM tree roots are marked, and only visible parts of the DOM tree are processed.
  - Later, contentful paints for elements inside these container trees will be observed (using Container Timing semantics), and will become candidates for emitting new `InteractionContentfulPaint` entries (using LCP semantics).
- We also observe all same document navigations (such as `pushState` or `navigate` event interceptions).
  - If an `InteractionContext` is available for the current task, we store a link between this same document navigation and the `InteractionContext`.
  - If this is the first same document navigation for this interaction, and all other criteria are met, we emit a `SoftNavigationEntry`.
- For these new entries, we expose the `interactionId` of the interaction that initiated it, and for all existing performance entries, we expose a `navigationId` to allow developers to group them by navigation.

## Examples

### Observing Soft Navigations

To observe the stream of new soft navigations, you can use a `PerformanceObserver`:

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const {
      startTime,
      renderTime,
      duration,
      interactionId,
      navigationId,
    } = entry;
    const url = entry.name;

    console.log(
      "[SoftNav] interactionId:", interactionId,
      "startTime:", startTime,
      "duration:", duration,
      "url:", url
    );
  }
}).observe({
  type: "soft-navigation",
  buffered: true, // Optional
});
```

Or, to list all the existing (buffered) entries so far, you can use `getEntriesByType`:

```js
const soft_navs = performance.getEntriesByType("soft-navigation");
```

Together with navigation timing (for the initial page load) you can map any Performance Entry to a navigation:

```js
function getNav(navigationId) {
    const navs = [
        performance.getEntriesByType('navigation')[0],
        ...performance.getEntriesByType('soft-navigation'),
    ];
    return navs.find(entry => entry.navigationId == navigationId);
}
```

### Observing InteractionContentfulPaint

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const {
      startTime,
      renderTime,
      duration,
      interactionId,
      element,
    } = entry;

    console.log(
      "[ICP] interactionId:", interactionId,
      "startTime:", startTime,
      "duration:", duration,
      "element:", element
    );
  }
}).observe({
  type: "interaction-contentful-paint",
  buffered: true // Optional
});
```

### Observing PerformanceEntrys sliced by Soft Navigations

```js
let currentNav = performance.getEntriesByType('navigation')[0];

const entryTypes = [
  "soft-navigation",
  "interaction-contentful-paint",
  // ... consider adding "event", "layout-shift", "resource", etc
];

function getEntriesByNavigation(entries, navigationId) {
  return entries.filter(
    entry => entry.navigationId === navigationId
  );
}

const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();

  for (const nav of [currentNav, ...list.getEntriesByType('soft-navigation')]) {
    currentNav = nav;
    const entriesForNav = getEntriesByNavigation(entries, nav.navigationId).filter(entry => entry.entryType !== "soft-navigation");
    if (!entriesForNav.length) continue;

    console.group(nav.navigationId, nav.name);
    for (const entry of entriesForNav) {
      console.log(entry.entryType, entry);
    }
    console.groupEnd();
  }
});

entryTypes.forEach(type => {
  observer.observe({ type, durationThreshold: 0, buffered: true });
});
```

## Privacy and Security

Exposing these entries does not introduce significant novel privacy risks.
- InteractionContentfulPaint timings follow the same security constraints as Container Timing and/or LCP (e.g., cross-origin image opt-in).
- Detection conditions are limited to trusted user interactions, preventing programmatic scanning of document updates.
- The use of `AsyncContext` ensures that attribution is strictly causal.

For a detailed analysis, see the [Security & Privacy section](https://wicg.github.io/soft-navigations/#priv-sec) of the specification.

## Considered alternatives

A few notes regarding alternative approaches:

- We considered using only semantic elements, but it seems to not match current real-world practices.
- We considered limiting DOM modifications to specific DOM elements or some other criteria regarding "meaningful" DOM modifications. We haven't seen a necessity for this in practice.
- Finally, we could consider limiting the amount of soft navigations detected in a certain timeframe (e.g. X per Y seconds), if we'd see that some web applications detect an excessive amount of soft navigations that don't correspond to the user experience.

# Design Doc for Soft-navs v2

This design doc aims to define the desired functionality and operation of the Soft Navigations performance timeline API, and serve as a template for a specification, rather than just detailing its exact current implementation in Chromium.

Some notes will be added to this draft design to document deviations from the current Chromium implementation.

## Proposed API shape

### `SoftNavigationEntry`

```
interface SoftNavigationEntry : PerformanceEntry {
}

SoftNavigationEntry includes PaintTimingMixin;
```

Note: The inheritance from `PerformanceEntry` means that the entry will have `startTime`, `name`, `entryType`, `duration`, and `navigationId`.

- `startTime` is a recomended new timeOrigin for this navigation. It is the earlier value of:
  - the user's interaction event's `processingEnd`, or
  - the URL was explicitly changed.
- `name` is the URL of the history entry representing the soft navigation.
- `entryType` is `"soft-navigation"`.
- `duration` is the time difference between the point in which a soft navigation is detected and the `startTime`.
- `navigationId` is a new pseudo-random number identifying this navigation.
- `SoftNavigationEntry` also includes `PaintTimingMixin` which exposed `paintTime` and `presentationTime` and which represent the First Contentful Paint (FCP) for this navigation.

### `InteractionContentfulPaint`

```
interface InteractionContentfulPaint : PerformanceEntry {
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

## Required spec changes

### `PerformanceEntry`

```
interface PerformanceEntry {
    readonly attribute unsigned long long navigationId;
}
```

- Extend `PerformanceEntry` to add `navigationId`.
  - This is a pseudorandom number which increments with each new navigation.
  - It is exposed to many different entry types and helps group (or slice) the single performance timeline by navigations.

### `PerformanceObserverInit` options

```
dictionary PerformanceObserverInit {
  boolean includeSoftNavigationObservations;
};
```

- We need to add a `PerformanceObserverInit` option named `"includeSoftNavigationObservations"`.
  - This flag is used to mark that `PerformanceEntry` should be reported with a `navigationId`.
  - This is required for `InteractionContentfulPaint` entries.

## NavigationId

- Update document to have a current **navigationId value**.

### Emitting InteractionContentfulPaint (ICP) Entry

- Define an **InteractionContext** struct
  - **created timestamp**
  - **most recent input or scroll timestamp**
  - **total size painted**
  - **largest paint candidate element timing**
  - _Note: Should slo add metadata describing / pointing back to the Interaction itself._
- Extend the Event Timing API to add support for more event types:
  - `popstate` and `navigate`.
- Hook into Event Timing API [initialize event timing](https://www.w3.org/TR/event-timing/#initialize-event-timing) "processingStart":
  - If this Event creates a new interaction (new interactionId), or may do so in the future:
    - _Create_ a new **InteractionContext**, and _Set_ its **created timestamp** to Now
    - _Save_ it into a map InteractionId→**InteractionContext**
    - _Note: We already have a mapping for active interactions, currently via pointer_id or key_code → Interaction data, in Event Timing. Could store it there, or even just invert the relationship._
  - Else, If this Event merges an existing interaction (same interactionId):
    - _Get_ its saved **InteractionContext**
  - Else, this is not an interaction, return
  - _Set_ the **active interaction context** for the **current Task** to this **InteractionContext** (via "AsyncContext")
    - _Note: This mechanism is needed for persistence across future task Scheduling._
- Observe Dom modifications:
  - E.g. `element.appendChild()` (and many variants), setting `<img src=...>`, etc.
  - _Note: The full list of apis that should qualify as dom modifications is evolving._
  - If there is no **active interaction context** (via “AsyncContext”), return
  - _Get_ the **most recent modifier** of this node, and if the **active interaction context** is already this value, return
  - _Assign_ the **active interaction context** as the **most recent modifier** of this Node.
- Observe Element paints:
  - _Get_ the **most recent modifier** of this Node to use as the **InteractionContext**, and if there is none, return
  - If the **InteractionContext** has a **most recent input or scroll timestamp**, return
  - _Record_ the paint for this Element as a **new attributable paint** for this context
    - …via Container Timing semantics where possible?
    - i.e. update **total size painted**, based on intersection area
      - One simple option is to just add all element areas
      - A better option is to maintain a Region and add the element rect to the region, in order to not double count overlapping areas.
    - i.e. update **largest paint candidate element**, if needed
  - If this updates the **total size painted**, or the **largest paint candidate element timing**:
    - _Measure_ **Paint Timing** (via **PaintTimingMixin**)
    - _Emit_ an **InteractionContentfulPaint** entry.
      - _Note: Mostly matches LCP entry behaviour, and may merge with ContainerTimingEntry._
      - _Note: All PerformanceEntry are assigned the current document **navigationId value** for their **navigationId** attribute._
- Observe Input or Scroll
  - If the Input or Scroll event is not a **trusted user input**, return
  - For each **InteractionContext,** _Set_ the **most recent input or scroll timestamp** to _Now_.
- Algo: Set/_Get_ the **active interaction context** for Task
  - _Note: …perhaps via hidden **AsyncContext.Variable**._
- Algo: _Assign_ the **most recent modifier** for Node to **InteractionContext**
  - Store a pointer from Node → **InteractionContext**
  - _Note: alternatively can use Context → Node, perhaps via set/vector stored in Interaction Context directly. This may be easier to write, but would require scanning/searching through n Contexts potentially with k Node’s, though in practice n and k are small and can be constrained / bounded._
- Algo: _Get_ the **most recent modifier** for Node
  - If Node, or one of its container nodes, was modified by an **InteractionContext**, return that context.

## Emitting SoftNavigation (SN) Entry

- Extend **InteractionContext** to add:
  - **most recent event processing end timestamp**
  - **first URL value**
  - **most recent URL value**
  - **first URL value update timestamp**
  - **first contentful paint timing info**
- Hook into Event Timing [finalize paint timing](https://www.w3.org/TR/event-timing/#sec-fin-event-timing) "processingEnd":
  - If we do not have an **active interaction context**, return
  - Update the **most recent event processing end timestamp** to _Now_.
- Observe History modifications:
  - If we do not have an **active interaction context**, return
  - If unset, _Set_ the **first URL value**
  - If unset, _Set_ the **first URL value timestamp** to _Now_
  - _Set_ the **most recent URL** value
  - _Call_ **Check if all conditions are met**
- Hook into **InteractionContext** _Record_ **new attributable paints**:
  - If this is the first observed element paint:
    - _Mark_ its **PaintTimingInfo** as the **first contentful paint timing info** for this **InteractionContext**
  - _Call_ **Check if all conditions are met**
- Algo: **Check if all conditions are met** to _Report_ **SoftNavigation** Entry for a **InteractionContext**
  - If this **InteractionContext** has already reported a **SoftNavigation** Entry, return
  - Check that **InteractionContext** has all of following, otherwise return:
    - **most recent event processing end timestamp**
    - **first URL value update timestamp**
    - **first URL value**
    - **most recent URL value**
    - **first contentful paint timing info**
  - If this **InteractionContext’s most recent URL** is not the **current document URL**, return
  - If total size painted is less than _Get_ Required Threshold Paint Area, return
  - _Update_ the current document **navigationId value** and _Increment_ **soft navigation count**
  - _Emit_ a **SoftNavigationEntry**
    - For **startTime** assign the smaller of:
      - **most recent event processing end timestamp**
      - **first URL value timestamp**
- Algo: _Get_ **Required Threshold Paint Area**
  - return 2% of viewport size

# Differences in Chromium (as of m139, July 2025)

- Currently, InteractionContext creation, and the InteractionContentfulPaint Entry, are tightly coupled to SoftNavigation Heuristics.
  - SoftNavigations should really just extend a generic InteractionContext tracking system, but is currently the manager of it.
- Currently, Event Timing "Interactions" are not aligned with InteractionContentfulPaint "Interactions".
- For example `pointerdown` and `pointerup` are Interactions (e.g. for INP) but not for ICP or SoftNavigation. Keyboard events are supported, but with a more limited capacity.
- SoftNavigations also requires observing some events that are not measured by Event Timings (such as `popstate` or `navigate`).
- There has been a recent feature request to expand the Event Timing concept of InteractionId to more events in general, and potentially we can unify around "uses same InteractionContext" for this also.
- Chromium does not currently fully implement the `AsyncContext` API proposal, nor does it expose that API to developers. There is effort underway to specify, and prototype, that feature, likely leveraging some of the work done here for Task Attribution. Part of that work includes an attempt to unify this work as much as possible.

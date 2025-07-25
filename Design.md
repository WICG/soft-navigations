# Design Doc for Soft-navs v2

This design doc aims to define the desired functionality and operation of the Soft Navigations performance timeline API, and serve as a template for a specification, rather than just detailing its exact current implementation in Chromium.

Some notes will be added to this draft design to document deviations from the current Chromium implementation.

## Proposed API shape

### `SoftNavigationEntry`

```idl
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

```idl
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

```idl
interface PerformanceEntry {
    readonly attribute unsigned long long navigationId;
}
```

- Extend `PerformanceEntry` to add `navigationId`.
  - This is a pseudorandom number which increments with each new navigation.
  - It is exposed to many different entry types and helps group (or slice) the single performance timeline by navigations.

### `PerformanceObserverInit` options

```idl
dictionary PerformanceObserverInit {
  boolean includeSoftNavigationObservations;
};
```

- We need to add a `PerformanceObserverInit` option named `"includeSoftNavigationObservations"`.
  - This flag is used to mark that `PerformanceEntry` should be reported with a `navigationId`.
  - This is required for `InteractionContentfulPaint` entries.

### NavigationId

- Update document to have a current **navigationId value**.

## Emitting InteractionContentfulPaint (ICP) Entry

### InteractionContext Struct

Define an **InteractionContext** struct with the following fields:

- **created timestamp**
- **most recent input or scroll timestamp**
- **total size painted**
- **largest paint candidate element timing**
- _Note: Should also add metadata describing / pointing back to the Interaction itself._

### Event Timing Integration

Extend the Event Timing API to add support for more event types: `popstate` and `navigate`.

When hooking into the Event Timing API's "initialize event timing" `processingStart` step:

1. If this Event creates a new interaction (new `interactionId`), or may do so in the future:
   1. _Create_ a new **InteractionContext**, and _Set_ its **created timestamp** to Now.
   2. _Save_ it into a map from `interactionId` to **InteractionContext**.
   - _Note: We already have a mapping for active interactions, currently via `pointer_id` or `key_code` → Interaction data, in Event Timing. Could store it there, or even just invert the relationship._
2. Else, If this Event merges an existing interaction (same `interactionId`):
   1. _Get_ its saved **InteractionContext**.
3. Else, this is not an interaction, return.
4. _Set_ the **active interaction context** for the **current Task** to this **InteractionContext** (via `AsyncContext`).
   - _Note: This mechanism is needed for persistence across future task Scheduling._

### Observing DOM Modifications

When observing DOM modifications (e.g. `element.appendChild()` or setting `<img src="...">`):

1. If there is no **active interaction context** (via `AsyncContext`), return.
2. _Get_ the **most recent modifier** of this node. If the **active interaction context** is already this value, return.
3. _Assign_ the **active interaction context** as the **most recent modifier** of this Node.

### Observing Element Paints

When observing element paints:

1. _Get_ the **most recent modifier** of this Node to use as the **InteractionContext**. If there is none, return.
2. If the **InteractionContext** has a **most recent input or scroll timestamp**, return.
3. _Record_ the paint for this Element as a **new attributable paint** for this context, for example by:
   - Updating **total size painted**, based on intersection area.
     - One simple option is to just add all element areas.
     - A better option is to maintain a Region and add the element rect to the region, in order to not double count overlapping areas.
   - Updating **largest paint candidate element**, if needed.
4. If this updates the **total size painted**, or the **largest paint candidate element timing**:
   1. _Measure_ **Paint Timing** (via **PaintTimingMixin**).
   2. _Emit_ an **InteractionContentfulPaint** entry.
   - _Note: This mostly matches LCP entry behaviour, and may merge with `ContainerTimingEntry`._
   - _Note: All `PerformanceEntry` objects are assigned the current document's **navigationId value** for their `navigationId` attribute._

### Observing Input or Scroll

When observing an input or scroll event:

1. If the Input or Scroll event is not a **trusted user input**, return.
2. For each **InteractionContext,** _Set_ the **most recent input or scroll timestamp** to _Now_.

### Abstract Operations

To _set/get the active interaction context_ for a Task:

- _Note: …perhaps via a hidden `AsyncContext.Variable`._

To _assign the most recent modifier_ for a Node to an **InteractionContext**:

1. Store a pointer from Node → **InteractionContext**.

To _get the most recent modifier_ for a Node:

1. If the Node, or one of its container nodes, was modified by an **InteractionContext**, return that context.

## Emitting SoftNavigation (SN) Entry

### Extending InteractionContext

Extend the **InteractionContext** struct to add the following fields:

- **most recent event processing end timestamp**
- **first URL value**
- **most recent URL value**
- **first URL value update timestamp**
- **first contentful paint timing info**

### Finalizing Event Timing

When hooking into Event Timing's "finalize paint timing" `processingEnd` step:

1. If there is no **active interaction context**, return.
2. Update the **most recent event processing end timestamp** to _Now_.

#### Observing History Modifications

When observing History modifications:

1. If there is no **active interaction context**, return.
2. If unset, _Set_ the **first URL value**.
3. If unset, _Set_ the **first URL value timestamp** to _Now_.
4. _Set_ the **most recent URL** value.
5. _Call_ to _check if all conditions are met_.

### Recording Attributable Paints

When hooking into an **InteractionContext** to _record new attributable paints_:

1. If this is the first observed element paint:
   1. _Mark_ its **PaintTimingInfo** as the **first contentful paint timing info** for this **InteractionContext**.
2. _Call_ to _check if all conditions are met_.

### Abstract Operations

To _check if all conditions are met_ to report a `SoftNavigationEntry` for an **InteractionContext context**:

1. If this **InteractionContext context** has already reported a `SoftNavigationEntry`, return.
2. Check that **InteractionContext context** has all of the following, otherwise return:
   - **most recent event processing end timestamp**
   - **first URL value update timestamp**
   - **first URL value**
   - **most recent URL value**
   - **first contentful paint timing info**
3. If the **most recent URL** of this **InteractionContext context** is not the current `Document`'s URL, return.
4. If **total size painted** is less than the value returned by running _get the Required Threshold Paint Area_, return.
5. _Update_ the current document **navigationId value** and _Increment_ the **soft navigation count**.
6. _Emit_ a `SoftNavigationEntry`.
   - For its `startTime`, assign the smaller of:
     - **most recent event processing end timestamp**
     - **first URL value timestamp**

To _get the Required Threshold Paint Area_:

1. Return 2% of the viewport size.

## Differences in Chromium (as of m139, July 2025)

- `InteractionContext` creation and `InteractionContentfulPaint` entries are tightly coupled to SoftNavigation Heuristics.
  - SoftNavigations should really just extend a generic InteractionContext tracking system, but it is currently the manager of it.
  - For example, `pointerdown` and `pointerup` are Interactions (e.g., for INP) but not for ICP or SoftNavigation. Keyboard events are supported, but with a more limited capacity.
- SoftNavigations also requires observing some events that are not measured by Event Timings (such as `popstate` or `navigate`).
- There has been a recent feature request to expand the Event Timing concept of `interactionId` to more events in general. Unifying around "uses same InteractionContext" may be possible.
- Chromium does not currently fully implement the `AsyncContext` API proposal, nor does it expose that API to developers.
  - There is effort underway to specify and prototype that feature, likely leveraging some of the work done here for Task Attribution. Part of that work includes an attempt to unify this work as much as possible.

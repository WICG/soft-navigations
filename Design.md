# Design Doc for Soft-navs v2

This design doc aims to define the desired functionality and operation of the Soft Navigations performance timeline API, and serve as a template for a specification, rather than just detailing its exact current implementation in Chromium.

Some notes will be added to this draft design to document deviations from the current Chromium implementation.

## InteractionContentfulPaint Entry

- Define an **InteractionContext** struct
  - **created timestamp**
  - **most recent input or scroll timestamp**
  - **total size painted**
  - **largest paint candidate element timing**
- Hook into Event Timing [initialize event timing](https://www.w3.org/TR/event-timing/#initialize-event-timing) (processingStart):
  - If this Event creates a new interaction (new interactionId), or may do so in the future:
    - _Create_ a new **InteractionContext**, and _Set_ its **created timestamp** to Now
    - _Save_ it into a map InteractionId→**InteractionContext**
    - _Note: We already have a mapping for active interactions, currently via pointer_id or key_code → Interaction data, in Event Timing. Could store it there, or even just invert the relationship._
  - Else, If this Event merges an existing interaction (same interactionId):
    - _Get_ its saved **InteractionContext**
  - Else, this is not an interaction, return
  - _Set_ the **active interaction context** for the **current Task** to this **InteractionContext** (for persistence across future task Scheduling)
  - _Note: ICP/SN doesn’t currently don’t observe all interaction event types, just a few, and constrained_
  - _Note: Event Timing doesn’t yet measure certain events needed for SN, such as popstate/navigate. This seems within scope to add._
- Observe Dom modifications:
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
    - _Emit_ an **InteractionContentfulPaint** entry
- Observe Input or Scroll
  - If the Input or Scroll event is not a **trusted user input**, return
  - For each **InteractionContext,** _Set_ the **most recent input or scroll timestamp** to _Now_.
- Algo: Set/_Get_ the **active interaction context** for Task
  - …perhaps via hidden **AsyncContext.Variable**, if possible?
- Algo: _Assign_ the **most recent modifier** for Node to **InteractionContext**
  - Store a pointer from Node → **InteractionContext**
  - _Note: alternatively can use Context → Node, perhaps via set/vector stored in Interaction Context directly. This may be easier to write, but would require scanning/searching through n Contexts potentially with k Node’s, though in practice n and k are small and can be constrained / bounded._
- Algo: _Get_ the **most recent modifier** for Node
  - If Node, or one of its container nodes, was modified by an **InteractionContext**, return that context.

## SoftNavigation Entry

- Extend **InteractionContext** to add**:**
  - **most recent event processing end timestamp**
  - **first URL value**
  - **most recent URL value**
  - **first URL value update timestamp**
  - **first contentful paint timing info**
- Hook into Event Timing [finalize paint timing](https://www.w3.org/TR/event-timing/#sec-fin-event-timing) (processingEnd):
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

# Differences in Chromium

- Currently, InteractionContext creation, and the InteractionContentfulPaint Entry, are tightly coupled to SoftNavigation Heuristics.
  - SoftNavigations should really just extend a generic InteractionContext tracking system, but is currently the manager of it.
- Currently, Event Timing "Interactions" are not aligned with InteractionContentfulPaint "Interactions".
- For example `pointerdown` and `pointerup` are Interactions (e.g. for INP) but not for ICP or SoftNavigation. Keyboard events are supported, but with a more limited capacity.
- SoftNavigations also requires observing some events that are not measured by Event Timings (such as `popstate` or `navigate`).
- There has been a recent feature request to expand the Event Timing concept of InteractionId to more events in general, and potentially we can unify around "uses same InteractionContext" for this also.

```

```

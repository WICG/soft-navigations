# Soft Navigations

## Overview

“Soft navigations” are JS-driven same-document navigations that are using the history API or the new Navigation API, triggered by a user gesture and modifies the DOM, modifying the previous content, as well as the URL displayed to the user.

[PerformanceTimeline#168](https://github.com/w3c/performance-timeline/issues/168) outlines the desire to be able to better report performance metrics on soft navigation. Heuristics for detecting soft navigations can ensure that developers that follow a well-lit path can ensure their SPA’s performance metric scores properly represent soft navigations.

## Motivation
Why would we want to web-expose soft navigation at all, you ask?

Well, a few reasons:
* Developers would like to attribute various performance entries to specific “soft navigation” URLs. For example, layout shifts caused in one URL can currently be attributed to the corresponding landing page, resulting in mis-attribution and trouble finding the real cause and fixing it.
* Developers would like to receive various “load” performance entries for soft navigations. Specifically, paint timing entries seem desired for such navigations.


## Proposed Heuristics
* The user initiated a soft navigation, by clicking on a DOM element.
  - We considered using only semantic elements, but it seems to not match current real-world practices.
* That operation resulted in an event handler firing (either a “click” event or a “navigate” event)
* We then follow the tasks triggered by the event handler:
  - If it’s a “navigate” event, those tasks are part of the Promise passed to traverseTo()
  - If it’s a “click” event, those tasks are spawned by the event handler itself
* In case of a “click” event, the handler triggered tasks that included History.pushState() or History.replaceState() calls, or a change to the document’s location
* The tasks modify DOM elements.
  - We may try to limit that to specific DOM elements or some other heuristic regarding "meaningful" DOM modifications, in case we'd see the heuristic is too broad and captures modifications which should not be reasonably be considered 
navigations".
* The next paint that contains a contentful element will be considered the soft navigation’s FCP.
* The next largest contentnful element will trigger LCP entries.
* Finally, we should consider limiting the amount of soft navigation detected in a certain timeframe (e.g. 1 per X seconds).

### [Task attribution](https://bit.ly/task-attribution)
The above heuristics rely on the ability to keep track of tasks and their provenance. We need to be able to tell that a certain task was posted by another, and be able to create a causality chain between DOM dirtying and URL modifications to the event handler that triggered the soft navigation.

Note: We would need to specify TaskAttribution as part of the event loop's processing in order to properly specify the heuristics above.

## Proposed API shape
```
SoftNavigationEntry : PerformanceEntry {
   unsigned long NavigationId;
}
```
[NavigationID](https://pr-preview.s3.amazonaws.com/w3c/performance-timeline/192/ca6936d...clelland:6e5497e.html#dom-performanceentry-navigationid) will be defined in Performance Timeline.

The inheritance from `PerformanceEntry` means that the entry will have `startTime`, `name`, `entryType` and `duration`:
* `startTime` would be defined as the time in which the user's click was received. See [discussion](https://bugs.chromium.org/p/chromium/issues/detail?id=1369680).
* `name` would be the URL of the history entry representing the soft navigation.
* `entryType` would be "soft-navigation".
* `duration` would be the time from `startTime` until the point in which all the tasks spawned by the user click were finished.

Note: `duration` would require modifying the current TaskAttribution infrastructure implemented in Chromium.

## Examples
That's all neat, but how would developers use the above? Great question!

### Reporting a soft navigation

If developers want to augment their current reporting with soft navigations, they'd need to do something like the following:
```javascript
const soft_navs = await new Promise(resolve => {
  (new PerformanceObserver( list => resolve(list.getEntries()))).observe(
    {type: 'soft-navigation', buffered: true});
  });
```

That would give them a list of past soft navigations they can send to their server for processing.

They would be able to also get soft navigations as they come (similar to other performance entries):
```javascript
const soft_navs = [];
(new PerformanceObserver( list => soft_navs.push(...list.getEntries()))).observe(
    {type: 'soft-navigation'});
```

### Correlating paints with a soft navigation

For that developers would need to collect `soft_navs` into an array as above.
```javascript
const lcp_entries = [];
(new PerformanceObserver( list => lcp_entries.push(...list.getEntries()))).observe(
    {type: 'largest-contentful-paint'});

for (entry of lcp_entries) {
  const id = entry.navigationId;
  const nav = soft_navs.filter(entry => entry.navigationId == id)[0];
  const lcp_duration = entry.startTime - nav.startTime;
}
```

## Required spec changes
* This relies on performance timeline's navigationID
* We'd need to specify Task Attribution
* We would need to modify PaintTiming nd LCP to restart their reporting once a soft navigation was encountered.

## Open Questions
* Do we need to define FCP/LCP as contentful paints that are the result of the soft navigation?
* Could we augment the heuristic to take both DOM additions and removals into account?
  - Currently, interactions such as Twitter/Gmail's "compose" button would be considered soft navigations, where one could argue they are really interactions.
  - A heuristic that requires either a modification of existing DOM nodes or addition *and* removal of nodes may be able to catch that, without increasing the rate of false negatives.
* `<your questions here>`

## I want to take this for a spin!!

I like how you're thinking!

Here's how:
* Install Chrome Canary, if you haven't already (or build tip-of-tree Chromium, if that's your thing)
* Enable "experimental web platform features"
* Browse to the site you want to test!
* Open the devtools' console
* Look for "A soft navigation has been detected" in the console logs
* Alternatively, run the example code above in your console to observe `SoftNavigationEntry` entries

And remember, if you found bugs, https://crbug.com is the best way to get them fixed!

This is based on the TAG's [security and privacy questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/).

# Questions to consider

## 01. What information does this feature expose, and for what purposes?

This feature adds `soft-navigation` and `interaction-contentful-paint` {{PerformanceEntry}} types to the web performance timeline to track interaction-driven page performance.

More specifically, it exposes:
1. **`interaction-contentful-paint`**: Reports on new contentful paints within parts of the page modified by a user interaction, helping developers understand interaction loading latency. This includes tracking timing for asynchronous work like fetch requests or image source updates associated with the interaction.
2. **`soft-navigation`**: Reports same-document history state changes initiated by interactions, establishing a new time origin to correctly attribute subsequent performance data to the active route rather than the initial document URL.

This exposure is necessary to allow developers to accurately slice performance timelines and attribute dynamic rendering updates (such as paints) to the user interactions that triggered them.

For soft navigations, exposing the timing doesn't reveal unexposed information about the user, as the same information could in theory be observed using code instrumentation.

Regarding paints following interactions (reported via ICP entries), because developers can associate user interactions with paint timing, this could expose (at a low, user-controlled rate) arbitrary paint operations on the document. This carries a risk of leaking history information if the `:visited` link cache has not been [partitioned](https://github.com/explainers-by-googlers/Partitioning-visited-links-history). Additionally, this timing could theoretically be used to observe paint updates when spelling or grammar error decorations (`::spelling-error` / `::grammar-error`) are applied, which could allow a site to probe the user's dictionary. The former risk is mitigated by requiring [visited link partitioning](https://github.com/explainers-by-googlers/Partitioning-visited-links-history), and the latter is mitigated at the source by limiting spelling/grammar highlight updates to at most once per interaction (see [user dictionary leaks explainer](https://explainers-by-googlers.github.io/user-dictionary-leaks/)).

## 02. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes. Exposing the timing of soft navigations, user interactions (via their IDs), and their associated paint entries is the minimum information necessary to enable accurate attribution of rendering performance to user actions.

## 03. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No. The feature is not related to, and does not expose, any PII or information derived from it.

## 04. How do the features in your specification deal with sensitive information?

The feature does not deal with sensitive information directly. There is a potential indirect leak of history info (via `:visited` links) or user dictionary contents (via spellcheck highlight paints), but these are mitigated by requiring [visited link partitioning](https://github.com/explainers-by-googlers/Partitioning-visited-links-history) and limiting spellcheck/grammar highlight updates to at most once per interaction (see [user dictionary leaks explainer](https://explainers-by-googlers.github.io/user-dictionary-leaks/)).

## 05. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No. The timing data and interaction associations are already observable by the origin through event listeners and custom instrumentation. The API standardizes and simplifies these measurements.

## 06. Do the features in your specification introduce state that persists across browsing sessions?

No.

## 07. Do the features in your specification expose information about the underlying platform to origins?

No.

## 08. Does this specification allow an origin to send data to the underlying platform?

No.

## 09. Do features in this specification enable access to device sensors?

No.

## 10. Do features in this specification enable new script execution/loading mechanisms?

No.

## 11. Do features in this specification allow an origin to access other devices?

No.

## 12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

No.

## 13. What temporary identifiers do the features in this specification create or expose to the web?

The specification exposes a `navigationId` attribute on `PerformanceEntry` objects and an `interactionId` on `InteractionContentfulPaint`.

`navigationId` is a 64-bit integer initialized to a random value per-Window and incremented by a random/small value chosen by the user agent, ensuring it cannot be used as a stable tracking identifier across different Windows or sessions.

`interactionId` is an existing attribute already exposed on Event Timing entries (used to group related events initiated by the same user interaction).

## 14. How does this specification distinguish between behavior in first-party and third-party contexts?

Soft navigations and interaction timing/ICP entries are queued on their respective Document's performance timeline. For documents in third-party contexts (such as cross-origin iframes), these entries are restricted to that iframe's timeline and are not accessible to the parent or other frames unless they are same-origin. There is no cross-origin leakage of this performance data.

## 15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Similarly to how it works outside of these modes.

## 16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

It has a single Security and Privacy considerations section.

## 17. Do features in your specification enable origins to downgrade default security protections?

No.

## 18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

When a document is restored from the BFCache, its script state and Performance Timeline are preserved. A BFCache restoration itself is a browser-initiated history navigation and does not trigger a soft navigation detection (which requires a user interaction same-document navigation). However, the specification does not currently explicitly define whether the `navigationId` is incremented or reset upon BFCache restoration.

We expect future extensions may do so. See: https://github.com/w3c/performance-timeline/issues/182.

## 19. What happens when a document that uses your feature gets disconnected?

Soft navigation and interaction tracking are only active for fully active documents. If a document gets disconnected or is no longer fully active, tracking is suspended or discarded.

## 20. Does your spec define when and how new kinds of errors should be raised?

No. The specification is a reporting-only API that populates the Performance Timeline. It does not define or raise any new exceptions or error types.

## 21. Does your feature allow sites to learn about the user's use of assistive technology?

No. Tracking is based on standard, generic user interaction events (such as clicks or key presses) dispatched by the browser. The API does not expose or distinguish whether these events were generated via assistive technologies or conventional input devices.

## 22. What should this questionnaire have asked?

None.

This is based on the TAG's [security and privacy questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/).

# Questions to consider

## What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature exposes information to web sites about soft navigations that happened and paints that follow them.
More specifically, they expose when a URL change and a DOM modifications were triggered as a result of a user interaction. That in itself doesn't reveal unexposed information about the user, and could in theory be achieved using code instrumentation.

Regarding paints that follow soft navigations, because developers can "translate" any user interaction into a soft navigation, this feature can expose (in a low, user-controlled rate) arbitrary paint operations on the document. This can be risky for certain paint metrics in cases where the `:visited` link cache has not been [partitioned](https://github.com/kyraseevers/Partitioning-visited-links-history).

Therefore this feature should not be enabled without `:visited` link partitioning.

## Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes. Exposing soft navigations and their related paint entries is the core functionality of the feature.

## How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

The feature is not related to any PII.

## How do the features in your specification deal with sensitive information?

The feature is not related to any sensitive information, once we assume `:visited` links are partitioned.

## Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

## Do the features in your specification expose information about the underlying platform to origins?

No.

## Does this specification allow an origin to send data to the underlying platform?

No.

## Do features in this specification enable access to device sensors?

No.

## Do features in this specification enable new script execution/loading mechanisms?

No.

## Do features in this specification allow an origin to access other devices?

No.

## Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

## What temporary identifiers do the features in this specification create or expose to the web?

None.

## How does this specification distinguish between behavior in first-party and third-party contexts?

This specification only operates at top-level documents, so only in first-party contexts.

## How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Similarly to how it works outside of these modes.

## Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

It has a single Security and Privacy considerations section.

## Do features in your specification enable origins to downgrade default security protections?

No.

## How does your feature handle non-"fully active" documents?

Soft navigations events will not be emitted to non-fully-active documents.


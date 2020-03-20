# Security and privacy

In general this specification allows web pages to opt in to better security and privacy for their users (specifically against side-channel protection), in the cases where browsers are able to process-isolate them.

## Questionnaire answers

The following are the answers to the W3C TAG's [security and privacy self-review questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/).

**Does this specification deal with personally-identifiable information?**

No.

**Does this specification deal with high-value data?**

No.

**Does this specification introduce new state for an origin that persists across browsing sessions?**

No.

**Does this specification expose persistent, cross-origin state to the web?**

No.

**Does this specification expose any other data to an origin that it doesn’t currently have access to?**

No. In fact, it allows an origin to reduce the data it has access to.

**Does this specification enable new script execution/loading mechanisms?**

No.

**Does this specification allow an origin access to a user’s location?**

No.

**Does this specification allow an origin access to sensors on a user’s device?**

No.

**Does this specification allow an origin access to aspects of a user’s local computing environment?**

No.

**Does this specification allow an origin access to other devices?**

No.

**Does this specification allow an origin some measure of control over a user agent’s native UI?**

No.

**Does this specification expose temporary identifiers to the web?**

No.

**Does this specification distinguish between behavior in first-party and third-party contexts?**

Yes. The [rules involved in choosing an agent cluster key](/README.md#specification-plan) will vary depending on what else is in the browsing context group, so e.g. a top-level page with no iframes can differ in behavior from a top-level page with many iframes, or from a page embedded as an iframe.

**How should this specification work in the context of a user agent’s "incognito" mode?**

There should be no difference.

**Does this specification persist data to a user’s local device?**

No.

**Does this specification have a "Security Considerations" and "Privacy Considerations" section?**

Not in itself. Since, as a specification, this will end up just being a patch to HTML, it will rely on that specification's sections.

**Does this specification allow downgrading default security characteristics?**

No.

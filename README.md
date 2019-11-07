# Origin Isolation explainer

This proposal provides a way for web applications to achieve **origin isolation**, segregating cross-origin documents into separate agent clusters, and thus allowing browsers to segregate them into different processes.

## Example

The delivery mechanism for a web application to achieve origin isolation is [origin policy](https://github.com/WICG/origin-policy). The application would have a JSON file at `/.well-known/origin-policy` looking something like:

```json
{
  "id": "my-policy",
  "origin_isolated": "best-effort"
}
```

and would add a HTTP header

```
Origin-Policy: allowed=(my-policy)
```

to their responses.

This indicates that any realms (documents or workers) derived from that origin should be separated from other cross-origin realms—even if those realms are [same site](https://github.com/whatwg/url/pull/449). In terms of observable, specified consequences for web developers, this means:

* Attempts to set `document.domain` will fail, so cross-origin realms will not be able to synchronously script each other, even if they are same site.
* Attempts to share `SharedArrayBuffer`s to cross-origins via `postMessage()` will fail, even if those realms are same site.

In terms of behind-the-scenes implementation, disallowing cross-origin synchronous scripting and shared memory allows implementations to, if they choose, put cross-origin realms in different processes. When implementations do this, web developers gain the following additional advantages:

* Protection against performance interference from cross-origin iframes or popups. Or, more positively, this allows the browser to spread out work from cross-origin iframes/popups across multiple cores.
* Protection against side channel attacks, such as [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)), even from same-site attackers.
* Memory metrics ([proposal 1](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md), [proposal 2](https://github.com/WICG/performance-memory/blob/master/explainer.md)) can become origin-focused instead of site-focused, which makes them more useful.

In other words, this feature allows web developers to voluntarily remove the ability to do certain (rather esoteric) cross-origin operations, in exchange for allowing the browser to create better isolation between origins.

## Objectives

Goals:

* Allow web applications to isolate themselves from cross-origin memory sharing and synchronous scripting (in a guaranteed way).
* Allow user agents to process-isolate realms when circumstances allow, and the developer has opted in to origin isolation, thus potentially achieving benefits for performance, security, and memory metrics.
* Ensure consistent web-developer-observable behavior (modulo side channels) regardless of implementation process allocation choices.
* Allow web applications to easily configure origin isolation for all URLs on their origin.

Non-goals:

* Mandate process isolation or other implementation choices for user agents.
* Give web developers visibility into process allocation.

## How it works

### Background: agent clusters

An important concept in the browser processing model is the [agent cluster](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Agent clusters are how the specification expresses which realms (windows/workers/etc.) are guaranteed to be able to share memory with each other using `SharedArrayBuffer`. They also constrain which pages can potentially synchronously script each other, e.g. via `someIframe.contentWindow.someFunction()`.

The HTML Standard currently [keys](https://html.spec.whatwg.org/multipage/webappapis.html#agent-cluster-key) agent clusters on scheme-and-registrable-domain ("site"), which roughly means that any documents within the same [browsing context group](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context-group) that are same-site can share memory with each other or synchronously script each other. Examples:

* `https://sub1.example.org/` can share memory with a `https://sub2.example.org/` iframe. It cannot share with a `http://sub1.example.org/` iframe (mismatched scheme).
* `https://example.com/` can share memory with a `https://example.com/` popup that it opens. It cannot share with a completely separate `https://example.com/` tab, nor with a popup opened with `noopener` (mismatched browsing context group).
* `https://whatwg.github.io/` cannot share memory with a `https://jsdom.github.io/`, because `github.io` is on the Public Suffix List.

Moving from the spec to the implementation level, the agent cluster boundary generally constrains how an implementation performs _process_ separation. That is, since the specification mandates certain documents be able to share memory with each other, implementations are constrained to put them in the same process, if they want to follow the specifications.

This means that the current state of the art in terms of segregating web content into different processes is [_site isolation_](https://www.chromium.org/Home/chromium-security/site-isolation), which segregates documents into separate processes according to the agent cluster rules. This can be done with no observable effects for JavaScript code.

Moving to origin isolation requires some observable changes, and thus the opt-in we propose here.

### Specification plan

The specification for this feature in itself is fairly small. Most of the work is done by the foundational mechanisms of [origin policy](https://github.com/WICG/origin-policy) and the HTML Standard's [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map).

In particular, when creating a document or worker, we use the following algorithm:

1. If an agent cluster keyed by the origin in question exists, use that agent cluster.
    * Example: `https://example.com/` was loaded with origin isolation, and embeds an iframe for `https://example.com/` which loads with origin isolation.
    * Example: `https://example.com/` was loaded with origin isolation, but embeds an iframe for `https://example.com/` which loads without origin isolation. In this case the iframe will still be origin-isolated; once a given browsing context group has `https://example.com/` in its list of isolated origins (i.e., its agent cluster map keys), there is no going back for that browsing context group.
1. Otherwise, if an agent cluster keyed by the site in question exists, use that agent cluster.
    * Example: `https://example.com/` was loaded without origin isolation, and embeds an iframe for `https://example.com/` which also loads without origin isolation.
    * Example: `https://example.com/` was loaded without origin isolation, but embeds an iframe for `https://example.com/` which loads with origin isolation. In this case the iframe will not be origin-isolated, even though it was requested, because we don't want to separate it from the existing site-keyed documents.
1. If no such agent clusters exist, and the origin policy contains an `"origin_isolated"` member not set to `"none"`, then create a new agent cluster keyed by origin, and use it.
    * Example: a new tab is created which loads `https://example.com/`, whose origin policy specifies origin isolation.
1. Otherwise, create a new agent cluster keyed by site.
    * Example: a new tab is created which loads `https://example.com/`, whose origin policy does not specify origin isolation.

This will automatically cause `SharedArrayBuffer`s to no longer be shareable cross-origin, because of the agent cluster check in [StructuredDeserialize](https://html.spec.whatwg.org/multipage/structured-data.html#structureddeserialize). We'd also need to add a check to the [`document.domain` setter](https://html.spec.whatwg.org/multipage/origin.html#dom-document-domain) to make it throw. (Ideally we'd find some way of restructuring the existing checks in `document.domain` so that they are also tied to agent cluster boundaries.)

The criteria of looking for `"origin_isolated"` not set to `"none"`, instead of specifically looking for `"best-effort"`, is done to increase future compatibility. See more on this [below](#the-best-effort-string). In the meantime, user agents should warn for any unknown values, explaining to the web developer that the unknown value is being treated as `"best-effort"`.

### More complicated example

Consider the following scenario:

* `https://example.org/` embeds an iframe for `https://a.example.com/` which embeds an iframe for `https://b.example.com/1`
* We open tab #1 to `https://example.org/`. Both `https://a.example.com/` and `https://b.example.com/1` responses point to origin policies with `"origin_isolated": "best-effort"` set.

(To simplify this example, assume at all times that we use "blocking" origin policies, i.e. no [async updates](https://github.com/WICG/origin-policy/issues/10) are involved.)

Now, things get fun:

* While we have tab #1 open, the server operator updates both `https://a.example.com/.well-known/origin-policy` and `https://b.example.com/.well-known/origin-policy` to set `"origin_isolated": "none"`.
* Then, in tab #1, `https://a.example.com/` inserts a new iframe, pointing to `https://b.example.com/2`. Since the `https://b.example.com/` policy on the server has been updated, the response used for creating this second child iframe is no longer requesting origin isolation.
* This `https://b.example.com/2` iframe inserts an iframe for `https://c.example.com/`, which has no origin policy. Then `https://b.example.com/2` tries to `postMessage()` a `SharedArrayBuffer` to `https://c.example.com/`.

What happens?

The answer that the above specification plan gives is that the `postMessage()` fails. Within the browsing context group (i.e. tab #1), `https://b.example.com/` is in the list of isolated origins, so even though the `https://b.example.com/2` iframe was loaded with an origin policy saying `"origin_isolated": "none"`, it still gets origin-isolated.

OK, let's go further.

* Now we open up a new tab to `https://example.org/`, tab #2. Because of the server update, the origin policies corresponding to the nested iframes for `https://a.example.com/` and `https://b.example.com/1` have `"origin_isolated": "none"` set.
* The `https://a.example.com/` iframe tries to `postMessage()` a `SharedArrayBuffer` to the `https://b.example.com/1` iframe.

What happens this time?

This time, we are in a new browsing context group, with a new agent cluster map. So this time, the iframes are not origin-isolated, and the sharing succeeds.

This means that, if you want to "un-isolate" your origin, you'll need to do so via a new, disconnected browsing context group, separate from the isolated one.

## Design choices and alternatives considered

### Origin policy

Origin policy is a natural fit for this feature, as isolation causes origin-wide effects. Compared to a per-document mechanism, such as a HTTP header, this better ensures that all documents on the origin are aligned behind the decision to be isolated.

### The browsing context group scope of isolation

The [more complicated example](#more-complicated-example) above explains how the current specification plan works in tricky scenarios involving origin policies changing between loads of documents from that origin. In short, origins get isolated per browsing context group.

We considered two other possible models:

* Per-document: in this model, both `postMessage()`s would succeed, as even in the tab #1 browsing context group, we would put `https://b.example.com/2` and `https://c.example.com/` in the same agent cluster.
* Per-user agent: in this model, both `postMessage()`s would fail, because the user agent keeps a global list of isolated origins, that does not get cleared until it restarts.

The downside of the per-document model is that it leads to strange scenarios like `https://b.example.com/1` and `https://b.example.com/2` being in different agent clusters. That is, in the example above, the per-document model would put `https://b.example.com/1` in an agent clustered keyed by the `(https, b.example.com, 443)` origin, while it puts `https://b.example.com/2` in an agent cluster keyed by the `(https, example.com)` site. This is not desirable, as same-origin pages should not be isolated from each other by the origin isolation feature.

The downside of the per-user agent model is that it makes it very hard for web application developers to roll back origin isolation. They essentially have to communicate to the user to restart their browser.

### Not using feature/document policy

Given the choice of origin policy to deliver this, why did we choose a new top-level key (`"origin_isolated"`) instead of using [feature policy](https://github.com/w3c/webappsec-feature-policy) or [document policy](https://github.com/w3c/webappsec-feature-policy/blob/master/document-policy-explainer.md)? Both of those are envisioned to be configurable on an origin-wide basis using origin policy, after all.

It turns out, both of these mechanisms are designed for dealing with embedded content, and as such have inheritance models that don't make much sense for origin isolation. If you want to isolate your own origin, that doesn't necessarily mean anything about how your subframes should behave—especially considering that the effects of feature/document policy are not limited to same-origin, or even same-site subframes.

### String-based enumeration

We originally considered `"origin_isolated": true` for configuring origin isolation. However, a string-based enumeration is more extensible in case we want to add more modes in the future. Some ideas that have been thrown around:

* `"required"` / `"for-security"` / `"process-isolated"` / `"no-spectres"`, which will fail to load documents on that origin if the user agent cannot guarantee process isolation. This would be used for sites that demand protection from side channel attacks like Spectre.

* `"for-performance"` / `"event-loop-isolated"`, which would tell the user agent that the main goal of isolation is performance isolation, and thus it could consider or prefer different strategies like using separate threads instead of separate processes.

* `"non-scriptable"`, which would tell the user agent that no degree of performance or process isolation is required, but the web application still wants to prevent synchronous cross-realm scripting. This could be useful to increase encapsulation.

The proposal's processing model is designed to be forward-compatible in the sense that if user agents add new values like these in the future, and web applications start using them, user agents that don't support the new values will instead treat them as `"best-effort"`. This is better than the alternative, of treating unknown values as `"none"`, which would fail to achieve any origin isolation in those user agents.

### Opt-in, instead of implicit, origin isolation

Another potential road we explored was to try to find scenarios where a process-per-origin could be done transparently, without an explicit opt-in. This would be possible if:

* We [restricted `SharedArrayBuffer` to same-origin usage](https://github.com/whatwg/html/issues/4920) at all times. (It's currently unknown if doing this would be web-compatible.)
* Pages used the [`"document-domain"` feature policy](https://html.spec.whatwg.org/multipage/infrastructure.html#document-domain-feature) to turn off their own ability to use `document.domain`.

Under these conditions, origin-based process isolation could be done by user agents unobservably, similar to how site-based process isolation is done today.

We avoided this approach because of the complexities involved in the use of the `"document-domain"` feature policy here. In particular, it can only allow isolation if all documents in a browsing context group impose the policy upon themselves. Since feature policy is a per-document, not per-origin or per-browsing context group configuration, this is hard to guarantee. What if a new page joins the browsing context group, which does not impose the feature policy? There are workarounds, e.g. changing the semantics of `"document-domain"` so that its default allowlist depends on other pages in the browsing context group, but these end up making the `"document-domain"` feature policy no longer mean "disable `document.domain`", but instead have a bunch of other side effects and action at a distance.

# Origin Isolation explainer

**Origin isolation** refers to segregating cross-origin documents into separate [agent clusters](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Translated into developer-observable effects, this means:

* preventing the [`document.domain`](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction) setter from relaxing the same-origin policy; and
* preventing `SharedArrayBuffer`s from being shared with cross-origin (but same-site) documents.

If a developer chooses to give up these capabilities, then the browser has more flexibility in how it allocates resources like processes, threads, and event loops.

This proposal allows web applications to give up these capabilities for their origin, and also hint at why they are doing so, to better guide the browser in its resource allocation decisions.

_Note: this proposal does not fully "isolate" the origin in every sense of the word. It focuses on agent cluster allocation issue. See [below](#potential-future-work-on-further-isolation) for potential future work on further isolation._

_Note: a [previous version](https://github.com/WICG/origin-isolation/tree/6c35a736792526877b97ecb2250d017a790a2980) of this proposal was based on [origin policy](https://wicg.github.io/origin-policy/), but we have now decoupled them. See [below](#origin-policy) for more on this subject._

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of contents

- [Example](#example)
- [Objectives](#objectives)
- [Proposed hints](#proposed-hints)
- [Implementation strategies](#implementation-strategies)
- [How it works](#how-it-works)
  - [Background: agent clusters](#background-agent-clusters)
  - [Specification plan](#specification-plan)
  - [More detail on workers](#more-detail-on-workers)
- [Design choices and alternatives considered](#design-choices-and-alternatives-considered)
  - [Using a per-document header](#using-a-per-document-header)
  - [The browsing context group scope of isolation](#the-browsing-context-group-scope-of-isolation)
  - [Not using feature/document policy](#not-using-featuredocument-policy)
  - [Usage hints](#usage-hints)
  - [Opt-in, instead of implicit, origin isolation](#opt-in-instead-of-implicit-origin-isolation)
  - [No-op `document.domain` setter instead of throwing](#no-op-documentdomain-setter-instead-of-throwing)
- [Adjacent work](#adjacent-work)
  - [`COOP` + `COEP`](#coop--coep)
- [Potential future work](#potential-future-work)
  - [Origin policy](#origin-policy)
  - [Further isolation](#further-isolation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Example

The delivery mechanism for a web application to achieve origin isolation is a new `Origin-Isolation` response header:

```
HTTP/1.1 200 OK
Content-Type: text/html
...
Origin-Isolation: isolated-event-loop, isolated-memory
...
```

The presence of the header indicates that any documents derived from that origin should be separated from other cross-origin documents—even if those documents are [same site](https://html.spec.whatwg.org/multipage/origin.html#same-site). In terms of observable, specified consequences for web developers, this means:

* Attempts to set `document.domain` will do nothing, so cross-origin documents will not be able to synchronously script each other, even if they are same site.
* Attempts to share `SharedArrayBuffer`s to cross-origin documents via `postMessage()` will fail, even if those documents are same site.

(Note that these are only observable consequences in document contexts; for workers, [origin isolation has no effect](#more-detail-on-workers).)

The value of the header consists of a series of tokens indicating to the browser that the reason why the origin is giving up these capabilities is in the hopes that the browser can provide better isolation of the origin's event loop and memory. See below for [the full list of possible hints](#proposed-hints).

The browser can then use these hints to choose a behind-the-scenes implementation strategy such as isolating the origin into its own process, or its own thread, or memory heap, or any other such implementation construct. These are just hints; ultimately the browser remains in charge of such implementation decisions.

The developer can also supply the header with no specific hints, just the [structured headers](https://httpwg.org/http-extensions/draft-ietf-httpbis-header-structure.html) boolean "true":

```
Origin-Isolation: ?1
```

This indicates that the page wants isolation solely for increased encapsulation, i.e. for the direct effects of preventing `document.domain` usage and `SharedArrayBuffer` memory sharing.

_[#18](https://github.com/WICG/origin-isolation/issues/18) is an issue for discussing the header value syntax._

## Objectives

Goals:

* Allow web applications to isolate themselves from cross-origin memory sharing and synchronous scripting (in a guaranteed way).
* Allow user agents to use various implementation strategies to isolate origins when circumstances allow, and the developer has opted in to origin isolation, thus potentially achieving benefits for performance, security, and memory metrics.
* Allow web applications to provide information to the browser about why they want isolation, in order to better guide the browser's implementation strategies.
* Ensure consistent web-developer-observable behavior regardless of implementation choices (such as process allocation) or the sub-preferences expressed by an origin. (Modulo side channels.)
* Avoid situations where two same-origin pages with different isolation preferences are treated as cross-origin by the browser. ([See below](#the-browsing-context-group-scope-of-isolation).)

Non-goals:

* Mandate process isolation or other implementation choices for user agents.
* Give web developers visibility into process allocation.

Non-goals for now:

* "Fully" isolate an origin from all other origins, e.g. by preventing sharing of data via site-scoped storage or navigation. See [below](#potential-future-work-on-further-isolation) for potential future work in that direction.
* Allow web applications to easily configure origin isolation for all URLs on their origin. See [below](#origin-policy) for potential future work in that direction.

## Proposed hints

The following tokens are proposed as hints:

* `isolated-event-loop`: indicates that one reason the origin is opting in to isolation is in the hope that its event loop will be isolated from those of other origins. Origins might want this in order to get more consistent and predictable performance, isolated from whatever is happening in cross-origin iframes or popups.

* `isolated-memory`: indicates that one reason the origin is opting in to isolation is in the hope of having an as-large-as-possible amount of memory available, such that large allocations from cross-origin pages should not cause the origin's allocations to start failing. Origins that plan on using lots of memory might want this. (This is intended to be a standards-track alternative to Mozilla's non-standardized [`Large-Allocation`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Large-Allocation) header.)

* `side-channel-protection`: indicates that one reason the origin is opting in to isolation is in the hopes of preventing side-channel attacks such as [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)). Origins might want this if they deal with sensitive data, which would be problematic if exfiltrated.

* `memory-measurement`: indicates that one reason the origin is opting in to isolation is in the hopes of enabling accurate memory measurement via the [proposed `performance.measureMemory()` API extensions](https://github.com/ulan/javascript-agent-memory/blob/master/explainer.md#future-api-extensions). This API is proposed to reject if the browser cannot restrict its measurements to only same-origin resources (or cross-origin ones which have opted in via `Cross-Origin-Resource-Policy`). Thus, by indicating that the origin plans to use this API, the browser can take steps to ensure the measurements are restricted.

If none of these hints are present, i.e. the header just contains the boolean `?1` indicator, then it is assumed that the origin wants isolation solely for increased encapsulation, i.e. for the direct effects of preventing `document.domain` usage and `SharedArrayBuffer` memory sharing.

## Implementation strategies

Currently, the state of the art in isolation is process-based. That is, current browser implementation strategies would likely place each origin-isolated origin in its own process.

The nuances of this proposal come into play under the following scenarios:

* When allocating a process-per-origin is infeasible, e.g. because you are operating on a low-memory device or have already allocated a ton of processes. In such scenarios the browser can use the different hints to prioritize how it allocates processes, e.g. prioritizing protection from side-channel attacks above allowing memory measurement.

* When alternate isolation techniques are available, which give some but not all of the benefits as proccess isolation. For example, Blink is exploring a ["multiple Blink isolates/threads"](https://docs.google.com/document/d/12wEWJsZmxVnNwVGuxuEJF4922OWUr4fCs1xKHi9mTiI/edit#heading=h.g6fq85as9ptv) project, which would provide isolated event loops and heaps, but not full side-channel protection. A site which expressed a preference only for isolated event loops could thus be given a new thread, instead of a new process, saving resources.

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

In particular, when creating a document, we use the following in place of the current algorithm for [obtain an agent cluster key](https://html.spec.whatwg.org/#obtain-agent-cluster-key). It takes as input an _origin_ of the realm to be created, _requestsIsolation_ (a boolean, derived from the presence of a valid `Origin-Isolation` header), and a browsing context group _group_. It outputs the [agent cluster key](https://html.spec.whatwg.org/#agent-cluster-key) to be used, which will be either a site or an origin.

1. If _origin_ is an [opaque origin](https://html.spec.whatwg.org/#concept-origin-opaque), then return _origin_.
1. If _origin_'s [host](https://html.spec.whatwg.org/#concept-origin-host)'s [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain) is null, then return _origin_.
1. If _group_'s [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map)\[_origin_] exists, then return _origin_.
1. Let _site_ be (_origin_'s [scheme](https://html.spec.whatwg.org/#concept-origin-scheme), _origin_'s [host](https://html.spec.whatwg.org/#concept-origin-host)'s [registrable domain](https://url.spec.whatwg.org/#host-registrable-domain)).
1. If _requestsIsolation_ is true:
    1. If _group_'s [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map)\[_site_] exists, and _group_'s [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map)\[_site_] contains any agents which contain any realms whose [settings object](https://html.spec.whatwg.org/#concept-realm-settings-object)'s [origin](https://html.spec.whatwg.org/#concept-settings-object-origin) are [same-origin](https://html.spec.whatwg.org/#same-origin) with _origin_, then return _site_.
    1. Return _origin_.
1. Return _site_.

This algorithm has two interesting features:

* Even if origin isolation is not requested for a particular document creation, if origin isolation has previously created an origin-keyed agent cluster, then we put the new document in that origin-keyed agent cluster.
* Even when origin isolation is requested, if there is a site-keyed agent cluster with same-origin documents, then we put the new document in that site-keyed agent cluster, ignoring the isolation request.

Both of these are consequences of a desire to ensure that same-origin sites do not end up isolated from each other.

You can see a more full analysis of what results this algorithm produces in our [scenarios document](./scenarios.md).

As far as the web-developer–observable consequences of using origins for agent cluster keys, this will automatically cause `SharedArrayBuffer`s to no longer be shareable cross-origin, because of the agent cluster check in [StructuredDeserialize](https://html.spec.whatwg.org/multipage/structured-data.html#structureddeserialize). We'd also need to add a check to the [`document.domain` setter](https://html.spec.whatwg.org/multipage/origin.html#dom-document-domain) to make it no-op if the current agent's agent cluster is origin-keyed.

### More detail on workers

We mentioned above that origin isolation has no effect on workers. Here we explain why exactly that is.

First note that shared and service workers are already isolated into their own agent clusters. The only other things that exist in those agent clusters are potential nested dedicated workers, which are already restricted to being same-origin.

What remains to consider is dedicated workers which are spawned directly by documents. Those end up in the agent cluster of their owner document.

Does the owner document's `Origin-Isolation` header impact any code inside such a decicated worker? The answer is no. All access to other realms is asynchronous inside a dedicated worker, so `document.domain` does not apply. And there's no way for dedicated worker code to send a `SharedArrayBuffer` to a same-site cross-origin destination directly, because the only thing it can communicate with is its parent document, or further nested dedicated workers, all of which are same-origin. Note that even `BroadcastChannel`, which bypasses the need to have a direct reference to the destination, is restricted to same-origin communications.

A worker could send a `SharedArrayBuffer` to a same-site cross-origin destination by first passing it (or a `MessageChannel` through which the `SharedArrayBuffer` gets transmitted) to a same-origin document, and letting the document try to go beyond the origin boundary. This would no longer be possible with this proposal, but only because of the proposal's effect on the document, not because of any effect it had on the worker.

(Note: this lack of impact is a bit hard to see with the current specification for how worker agents and agent clusters are allocated, which is declarative. [whatwg/html#5210](https://github.com/whatwg/html/issues/5210) tracks improving that specification; with such improvements, some of the above would become clearer.)

## Design choices and alternatives considered

### Using a per-document header

This proposal follows in the tradition of other security primitives like CSP, COEP, and COOP, by using a header to change the behavior of the document.

A previous iteration of this proposal used [origin policy](https://wicg.github.io/origin-policy/), in an attempt to ensure that all documents on the origin are aligned behind the decision to be isolated. We believe this is still a desirable direction, both for the `Origin-Isolation` header and for other security headers. But it's worth noting that the advantages are mainly in easing server configuration. Because different page loads on an origin could recieve different origin policies anyway (due to, e.g., server-side updates), there is no technical simplification gained by using origin policy. That is, the browser and site developer need to deal with the potential for varying origin isolation hints across the site in both delivery mechanisms.

See [below](#origin-policy) for how this proposal might be upgraded to use origin policy in the future.

### The browsing context group scope of isolation

The [scenarios document](./scenarios.md) explains how the current specification plan works in tricky scenarios involving different `Origin-Isolation` headers between loads of realms from a given origin. In short, origins get isolated per browsing context group.

We considered two other possible models:

* Per-realm: in this model, we would not check for the existence of other site-keyed realms within the browsing context group, and would always origin-isolate realms which request isolation, and never isolate ones that do not.
* Per-user agent: in this model, we would keep a global list of all origins which have ever requested isolation, and consult this first whenever creating a new realm.

The downside of the per-realm model is that it leads to strange scenarios like `https://b.example.com/1` and `https://b.example.com/2` being in different agent clusters: one of them site-keyed, and the other origin-keyed. This is not desirable, as same-origin pages should not be isolated from each other by the origin isolation feature.

The downside of the per-user agent model is that it makes it very hard for web application developers to roll back origin isolation. They essentially have to communicate to the user to restart their browser.

### Not using feature/document policy

Why did we choose to invent a new header for origin isolation, instead of using [feature policy](https://github.com/w3c/webappsec-feature-policy) or [document policy](https://github.com/w3c/webappsec-feature-policy/blob/master/document-policy-explainer.md)?

It turns out, both of these mechanisms are designed for dealing with embedded content, and as such have inheritance models that don't make much sense for origin isolation. If you want to isolate your own origin, that doesn't necessarily mean anything about how your subframes should behave—especially considering that the effects of feature/document policy are not limited to same-origin, or even same-site, subframes.

### Usage hints

We've considered several previous designs for origin isolation, such as just `Origin-Isolation: ?1` to indicate isolation with no additional information, or something like `Origin-Isolation-Implementation-Preference: process` / `thread` / `separate-heap` to directly indicate the preferred implementation strategy. See some discussion in [#1](https://github.com/domenic/origin-isolation/issues/1).

A simple boolean gives the user agent too little information to make the right choices. Faced with multiple possible implementation strategies and constrained resources, how can it best decide what strategy to use? Currently Chrome uses heuristics, like "a page containing a password input probably needs side-channel protection". Making this more explicit can only benefit pages. In the end, user agents which don't find the extra information useful can ignore it.

On the other end of the spectrum, directly indicating implementation choices to the browser is overstepping a bit. It's unlikely a site author has enough information to confidently declare that they need a process; instead, they know things about their performance, memory, or security needs, which the browser can take into account.

### Opt-in, instead of implicit, origin isolation

Another potential road we explored was to try to find scenarios where a process-per-origin could be done transparently, without an explicit opt-in. This would be possible if:

* We [restricted `SharedArrayBuffer` to same-origin usage](https://github.com/whatwg/html/issues/4920) at all times. (It's currently unknown if doing this would be web-compatible.)
* Pages used the [`"document-domain"` feature policy](https://html.spec.whatwg.org/multipage/infrastructure.html#document-domain-feature) to turn off their own ability to use `document.domain`.

Under these conditions, origin-based process isolation could be done by user agents unobservably, similar to how site-based process isolation is done today.

In addition to the same concerns as stated above about how this doesn't give useful hints to the user agent, we avoided this approach because of the complexities involved in the use of the `"document-domain"` feature policy. In particular, it can only allow isolation if all documents in a browsing context group impose the policy upon themselves. Since feature policy is a per-document, not per-origin or per-browsing context group configuration, this is hard to guarantee. What if a new page joins the browsing context group, which does not impose the feature policy? There are workarounds, e.g. changing the semantics of `"document-domain"` so that its default allowlist depends on other pages in the browsing context group, but these end up making the `"document-domain"` feature policy no longer mean "disable `document.domain`", but instead have a bunch of other side effects and action at a distance.

### No-op `document.domain` setter instead of throwing

The proposal here is to make the `document.domain` setter do nothing (perhaps emitting a console warning) when used. Throwing an exception would be more semantically correct: it is unable to do the operation you are requesting.

However, throwing would pose a potential migration burden for sites. Measurements from Chrome show that the `document.domain` setter is widely used ([~12% of page views](https://chromestatus.com/metrics/feature/timeline/popularity/739)), but only has an effect a smaller percentage of the time ([~0.28%](https://chromestatus.com/metrics/feature/timeline/popularity/2543) + [~0.52%](https://chromestatus.com/metrics/feature/timeline/popularity/2544) of page views). If we made it throw, then all of that 12% would need to update their code before they could take advantage of origin isolation. By making it do nothing, we instead confine the migration cost to the <1%.

## Adjacent work

### `COOP` + `COEP`

The [`Cross-Origin-Opener-Policy`](https://gist.github.com/annevk/6f2dd8c79c77123f39797f6bdac43f3e) and [`Cross-Origin-Embedder-Policy`](https://mikewest.github.io/corpp/) headers, when combined, also give origin isolation. That is, documents which include both headers will automatically change their agent cluster key to use the origin, instead of the site. As discussed previously, this has observable effects on `document.domain` and `SharedArrayBuffer`. Note that, in contrast to the problems explained above with using feature/document policy, this model works, since `COEP` is required to match for all descendant browsing contexts anyway, and `COOP` is consulted via the top-level browsing context.

We are optimistic about this approach for getting the observable effects of origin isolation. In particular, preventing the detrimental effects of `document.domain` is a long-time goal of the web standards and browser community, and tying that to headers like `COOP` + `COEP` and the various carrots they unlock seems like a great strategy.

However, we think that there remains room for this separate origin isolation proposal. In particular:

* Turning on `COOP` + `COEP` does not provide a clear signal that the origin desires origin isolation, or why it would desire origin isolation. As discussed at length above, these signals can be useful for the browser in making decisions on implementation strategies. For example, consider a site with 30 iframes containing ads, which wishes to achieve side-channel protection. With an explicit dedicated isolation signal, the top-level site could explain its desire for side-channel protection, and recieve a dedicated process, while letting the ad frames get coalesced into a single process. Whereas if `COOP` + `COEP` by itself automatically created a process-isolation signal, then the browser could only use heuristics to guess whether to allocate 2 processes, or 31 processes, or something in between.

* Deploying `COOP` + `COEP` can be a significant burden for sites, e.g. in terms of how it requires all of their embedded content to be updated with `CORP` headers. Some sites may not want to use any of the features enabled by `COOP` + `COEP` (e.g. `SharedArrayBuffer` or `process.measureMemory()`), and so be unmotivated to take on this burden. But they still might want origin isolation, e.g. of the event loop or heap isolation variety. In such cases, this separate header would be much easier to deploy than `COOP` + `COEP`, since it does not require any embeddee cooperation.

So to conclude, the separate `Origin-Isolation` header can be used both by sites who are not yet using `COOP` + `COEP`, to get origin isolation without taking on the burden of cross-origin isolation, and it can be used by sites that are using `COOP` + `COEP`, to provide the browser with specific hints that guide its implementation choices for converting origin isolation into process isolation.

## Potential future work

### Origin policy

A [previous version](https://github.com/WICG/origin-isolation/tree/6c35a736792526877b97ecb2250d017a790a2980) of this proposal relied on origin isolation. At this time, however, there remain a number of [open design questions](https://github.com/WICG/origin-policy/issues?q=is%3Aissue+is%3Aopen+label%3A%22open+design+question%22) about origin policy, some with no clear path toward a solution. So the plan is to decouple origin isolation from origin policy for now, instead using the `Origin-Isolation` header design seen throughout the rest of this document.

That said, we believe that origin policy would still be an optimal deployment vehicle for many things currently delivered as headers. This includes not only `Origin-Isolation`, but also `Content-Security-Policy`, `Feature-Policy`, `Document-Policy`, `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy`, and more. The same [arguments](https://github.com/WICG/origin-policy#the-problems-and-goals) as to why these would be best configured on an origin-wide basis apply to all of these headers. So for now, the plan is to develop origin isolation in the same way as the others, but also to add it to the list of candidates for integration into origin policy, once origin policy becomes ready.

For reference, a sample origin policy manifest expressing similar values to the header would be something like

```json
{
  "ids": ["my-policy"],
  "isolation": {
    "isolated_event_loop": true,
    "isolated_memory": true
  }
}
```

### Further isolation

As noted earlier in this document, this proposal is focused narrowly on the agent cluster allocation issue, and its implications for synchronous cross-origin access which has various security and performance effects.

One might envision others was of isolating an origin that go beyond this. Many such concepts have been discussed in a previous ["Isolation Explainer"](https://wicg.github.io/isolation/explainer.html) from 2016. Some, like `window.opener` restriction or preventing cross-origin fetches, have since found solutions. (Respectively, [`Cross-Origin-Opener-Policy`](https://gist.github.com/annevk/6f2dd8c79c77123f39797f6bdac43f3e) and [`Cross-Origin-Resource-Policy`](https://fetch.spec.whatwg.org/#cross-origin-resource-policy-header)/[`Sec-Fetch-Site`](https://www.w3.org/TR/fetch-metadata/#sec-fetch-site-header).) The ones that remain appear to be:

* Preventing the origin from sharing storage (of [various types](https://storage.spec.whatwg.org/#infrastructure)) with different origins, even if those origins are same-site. This would include things like cookies or [Web Authentication](https://w3c.github.io/webauthn/) entries.
* Double-keying cookies, so that when an isolated origin makes a request to `https://third-party.example/`, this uses a separate cookie jar from when any other origin makes a request to `https://third-party.example/`.
* Preventing navigations from other origins to this origin (or perhaps allowing only specific entry points as navigation targets).

We believe that, after tackling agent cluster allocation, these may be worthwhile further efforts to pursue. If so, building on top of the `Origin-Isolation` header seems very reasonable. One could imagine something like

```
Origin-Isolation: side-channel-protection, isolated-storage
```

as an extension of this proposal.

# Origin Isolation explainer

**Origin isolation** refers to segregating cross-origin documents into separate [agent clusters](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Translated into developer-observable effects, this means:

* preventing the [`document.domain`](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction) setter from relaxing the same-origin policy; and
* preventing `WebAssembly.Module`s from being shared with cross-origin (but same-site) documents.

If a developer chooses to give up these capabilities, then the browser has more flexibility in how it allocates resources like processes, threads, and event loops.

_Note: this proposal does not fully "isolate" the origin in every sense of the word. It focuses on agent cluster allocation issue. See [below](#further-isolation) for potential future work on further isolation._

_Note: a [previous version](https://github.com/WICG/origin-isolation/tree/6c35a736792526877b97ecb2250d017a790a2980) of this proposal was based on [origin policy](https://wicg.github.io/origin-policy/), but we have now decoupled them. See [below](#origin-policy) for more on this subject._

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of contents

- [Example](#example)
- [Objectives](#objectives)
- [Implementation strategies](#implementation-strategies)
- [How it works](#how-it-works)
  - [Background: agent clusters](#background-agent-clusters)
  - [Specification plan](#specification-plan)
  - [Parallelism](#parallelism)
  - [`SharedArrayBuffer`, `WebAssembly.Memory`, and `WebAssembly.Module`](#sharedarraybuffer-webassemblymemory-and-webassemblymodule)
  - [More detail on workers](#more-detail-on-workers)
- [Design choices and alternatives considered](#design-choices-and-alternatives-considered)
  - [Using a per-document header](#using-a-per-document-header)
  - [The browsing context group scope of isolation](#the-browsing-context-group-scope-of-isolation)
  - [Not using feature/document policy](#not-using-featuredocument-policy)
  - [Opt-in, instead of implicit, origin isolation](#opt-in-instead-of-implicit-origin-isolation)
  - [No-op `document.domain` setter instead of throwing](#no-op-documentdomain-setter-instead-of-throwing)
- [Adjacent work](#adjacent-work)
  - [`COOP` + `COEP`](#coop--coep)
- [Potential future work](#potential-future-work)
  - [Hints](#hints)
  - [Origin policy](#origin-policy)
  - [Further isolation](#further-isolation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Example

The delivery mechanism for a web application to achieve origin isolation is a new `Origin-Isolation` response header:

```
HTTP/1.1 200 OK
Content-Type: text/html
...
Origin-Isolation: ?1
...
```

The presence of the header, with the [structured headers](https://httpwg.org/http-extensions/draft-ietf-httpbis-header-structure.html) boolean "true" value `?1`, indicates that any documents derived from that origin should be separated from other cross-origin documents—even if those documents are [same site](https://html.spec.whatwg.org/multipage/origin.html#same-site). In terms of observable, specified consequences for web developers, this means:

* Attempts to set `document.domain` will do nothing, so cross-origin documents will not be able to synchronously script each other, even if they are same site.
* Attempts to share `WebAssembly.Module`s to cross-origin documents via `postMessage()` will fail, even if those documents are same site.

(Note that these are only observable consequences in document contexts; for workers, [origin isolation has no effect](#more-detail-on-workers).)

## Objectives

Goals:

* Allow web applications to isolate themselves from cross-origin synchronous scripting (in a guaranteed way).
* Allow user agents to use various implementation strategies to isolate origins when circumstances allow, and the developer has opted in to origin isolation, thus potentially achieving benefits for performance, security, and memory metrics.
* Ensure consistent web-developer-observable behavior regardless of implementation choices (such as process allocation) or the hints expressed by an origin. (Modulo side channels.)
* Avoid situations where two same-origin pages with different isolation preferences are treated as cross-origin by the browser. ([See below](#the-browsing-context-group-scope-of-isolation).)

Non-goals:

* Mandate process isolation or other implementation choices for user agents.
* Give web developers visibility into process allocation.

Non-goals for now:

* "Fully" isolate an origin from all other origins, e.g. by preventing sharing of data via site-scoped storage or navigation. See [below](#further-isolation) for potential future work in that direction.
* Allow web applications to easily configure origin isolation for all URLs on their origin. See [below](#origin-policy) for potential future work in that direction.
* Allow web applications to provide information to the browser about why they want isolation, in order to better guide the browser's implementation strategies. See [below](#hints) for potential future work in that direction.

## Implementation strategies

Currently, the state of the art in isolation is process-based. That is, current browser implementation strategies would likely place each origin-isolated origin in its own process. But it's important for applications to recognize that this header does not directly control process allocation, or other behind-the-scenes implementation strategies.

For example, in some cases allocating a process-per-isolated-origin is infeasible, e.g. because you are operating on a low-memory device, or have already allocated a ton of processes, or because the browser does not implement the ability for iframes to be placed in a separate process. In such scenarios the JavaScript-observable effects of isolation will still be present, giving applications an increase in encapsulation. But there wouldn't be any performance or security benefits.

Alternate isolation techniques are possible,  which give some but not all of the benefits as process isolation. For example, Blink is exploring a ["multiple Blink isolates/threads"](https://docs.google.com/document/d/12wEWJsZmxVnNwVGuxuEJF4922OWUr4fCs1xKHi9mTiI/edit#heading=h.g6fq85as9ptv) project, which would provide parallelism and isolated heaps, but not full side-channel protection. But, these remain speculative as of the time of this writing.

A [future extension](#hints) might be able to help the browser better prioritize when to allocate processes, or choose between process isolation and other techniques.

## How it works

### Background: agent clusters

An important concept in the browser processing model is the [agent cluster](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Agent clusters are how the specification expresses which realms (windows/workers/etc.) are able to share `SharedArrayBuffer`, `WebAssembly.Memory`, and `WebAssembly.Module` instances. They also constrain which pages can potentially synchronously script each other, e.g. via `someIframe.contentWindow.someFunction()`.

The HTML Standard currently [keys](https://html.spec.whatwg.org/multipage/webappapis.html#agent-cluster-key) agent clusters on scheme-and-registrable-domain ("site"), which roughly means that any documents within the same [browsing context group](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context-group) that are same-site can potentially synchronously script each other. Examples:

* `https://sub1.example.org/` could script an `https://sub2.example.org/` iframe. It could not script  a `http://sub1.example.org/` iframe (mismatched scheme).
* `https://example.com/` could script an `https://example.com/` popup that it opens. It could not script  a completely separate `https://example.com/` tab, nor a popup opened with `noopener` (mismatched browsing context group).
* `https://whatwg.github.io/` could not script `https://jsdom.github.io/`, because `github.io` is on the Public Suffix List.

Moving from the spec to the implementation level, the agent cluster boundary generally constrains how an implementation performs _process_ separation. That is, since the specification mandates certain documents be able to access each other synchronously, implementations are constrained to put them in the same process. (Cross-origin same-site pages need to both set `document.domain` in order to allow such synchronous scripting, but the very fact that they could do so at any time constrains implementation choices.)

This means that the current state of the art in terms of segregating web content into different processes is [_site isolation_](https://www.chromium.org/Home/chromium-security/site-isolation), which segregates documents into separate processes according to the agent cluster rules. This can be done with no observable effects for JavaScript code.

Moving to origin isolation requires some observable changes, and thus the opt-in we propose here.

### Specification plan

The specification for this feature in itself consists of two parts. One part is header parsing, which relies on the [Structured Headers](https://httpwg.org/http-extensions/draft-ietf-httpbis-header-structure.html) specification to do most of the work. The other is agent cluster keying, which is done by modifying the HTML Standard's [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map).

In particular, when creating a document, we modify the algorithm to [obtain an agent cluster key](https://html.spec.whatwg.org/#obtain-agent-cluster-key). It takes as input an _origin_ of the realm to be created, _requestsIsolation_ (a boolean, derived from the presence of a valid `Origin-Isolation` header), and a browsing context group _group_. It outputs the [agent cluster key](https://html.spec.whatwg.org/#agent-cluster-key) to be used, which will be either a site or an origin.

The new algorithm will check if _origin_ has previously been placed in a site- or origin-keyed agent cluster within _group_, and if so, it will always return that same key, to stay consistent. Otherwise, it uses the _requestsIsolation_ boolean to determine whether to return _origin_, or the derived site.

The consistency part of this algorithm has two interesting features:

* Even if origin isolation is not requested for a particular document creation, if origin isolation has previously created an origin-keyed agent cluster, then we put the new document in that origin-keyed agent cluster.
* Even when origin isolation is requested, if there has previously been a site-keyed agent cluster with same-origin documents in the browsing context group, then we put the new document in that site-keyed agent cluster, ignoring the isolation request.

Both of these are consequences of a desire to ensure that same-origin sites do not end up isolated from each other, even in scenarios involving navigating back and forward.

You can see a more full analysis of what results this algorithm produces in our [scenarios document](./scenarios.md).

As far as the web-developer–observable consequences of using origins for agent cluster keys, this will automatically cause `WebAssembly.Module` instances to no longer be shareable cross-origin, because of the agent cluster check in [StructuredDeserialize](https://html.spec.whatwg.org/multipage/structured-data.html#structureddeserialize). We'd also need to add a check to the [`document.domain` setter](https://html.spec.whatwg.org/multipage/origin.html#dom-document-domain) to make it no-op if the surrounding agent's agent cluster is origin-keyed.

### Parallelism

Will origin isolation necessarily increase parallelism? The answer is no; parallelism remains an implementation choice, and is not forced by changing the agent cluster allocation. This is because the relevant specifications are structured to allow non-parallelized implementations, even across totally separate agents. The JavaScript specification [says](https://tc39.es/ecma262/#sec-agents)

> an executing thread may be used as the executing thread by multiple agents

and gives the example of a web browser sharing a single thread across multiple unrelated tabs in a browser window.

In general, the only thing in the specification ecosystem that forces parallelism is workers, via their dedicated executing thread and the requirements around [forward progress](https://tc39.es/ecma262/#sec-forward-progress) and the atomics APIs. So, although origin isolation could incidentally result in better parallelism as part of how implementations respond behind the scenes, parallelism is not automatically forced.

### `SharedArrayBuffer`, `WebAssembly.Memory`, and `WebAssembly.Module`

Does this proposal have any effect on sharing of `SharedArrayBuffer` or `WebAssembly.Memory`? The answer is no, but the reasoning is somewhat subtle.

The [current specification consensus](https://github.com/whatwg/html/pull/4734) is that `SharedArrayBuffer` or `WebAssembly.Memory` will only be structured-serializable when the surrounding agent's agent cluster is _cross-origin isolated_, which means that `COOP` + `COEP` headers have been set. But cross-origin isolation automatically implies origin isolation; that is, cross origin isolation means that `SharedArrayBuffer` and `WebAssembly.Memory` will not be able to be sent cross-origin anyway. See [below](#coop--coep) for more discussion on those two headers, and on the relationship between cross-origin isolation and origin isolation.

(Note that currently in Chromium on desktop platforms, this specification consensus is not yet followed, and `SharedArrayBuffer` and `WebAssembly.Memory` can be sent to cross-origin same-site destinations within the same agent cluster. The plan is for origin isolation to restrict this. But this is a temporary, desktop-Chromium-specific situation, and does not impact the specification proposal.)

Finally, we turn our attention to `WebAssembly.Module`. Structured serialization of `WebAssembly.Module` is restricted to agent clusters, but not for security reasons: the restriction is in place because the point of structured serializing a `WebAssembly.Module` is to avoid recompilation of the module, which is only possible if its destination is within the same process. As such, sending `WebAssembly.Module` around is not guarded by `COOP` + `COEP`, and so origin isolation will observably change the boundaries within which it can be sent, by changing the scope of agent clusters to be origin-keyed instead of site-keyed. This is why, in addition to the synchronous scripting restrictions, we mention the changes to `WebAssembly.Module` serialization as an observable consequence of origin isolation.

### More detail on workers

We mentioned above that origin isolation has no effect on workers. Here we explain why exactly that is.

First note that shared and service workers are already isolated into their own agent clusters. The only other things that exist in those agent clusters are potential nested dedicated workers, which are already restricted to being same-origin.

What remains to consider is dedicated workers which are spawned directly by documents. Those end up in the agent cluster of their owner document.

Does the owner document's `Origin-Isolation` header impact any code inside such a decicated worker? The answer is no. All access to other realms is asynchronous inside a dedicated worker, so `document.domain` does not apply. And there's no way for dedicated worker code to send a `WebAssembly.Module` to a same-site cross-origin destination directly, because the only thing it can communicate with is its parent document, or further nested dedicated workers, all of which are same-origin. Note that even `BroadcastChannel`, which bypasses the need to have a direct reference to the destination, is restricted to same-origin communications.

A worker could send a `WebAssembly.Module` to a same-site cross-origin destination by first passing it (or a `MessageChannel` through which the `WebAssembly.Module` gets transmitted) to a same-origin document, and letting the document try to go beyond the origin boundary. This would no longer be possible with this proposal, but only because of the proposal's effect on the document, not because of any effect it had on the worker.

## Design choices and alternatives considered

### Using a per-document header

This proposal follows in the tradition of other security primitives like CSP, COEP, and COOP, by using a header to change the behavior of the document.

A previous iteration of this proposal used [origin policy](https://wicg.github.io/origin-policy/), in an attempt to ensure that all documents on the origin are aligned behind the decision to be isolated. We believe this is still a desirable direction, both for the `Origin-Isolation` header and for other security headers. But it's worth noting that the advantages are mainly in easing server configuration. Because different page loads on an origin could recieve different origin policies anyway (due to, e.g., server-side updates), there is no technical simplification gained by using origin policy. That is, the browser and site developer need to deal with the potential for varying origin isolation preferences across the site in both delivery mechanisms.

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

### Opt-in, instead of implicit, origin isolation

Another potential road we explored was to try to find scenarios where a process-per-origin could be done transparently, without an explicit opt-in. This would be possible if:

* We [restricted `WebAssembly.Module` to same-origin usage](https://github.com/WebAssembly/spec/issues/1081) at all times. (It's currently unknown if doing this would be web-compatible.)
* Pages used the [`"document-domain"` feature policy](https://html.spec.whatwg.org/multipage/infrastructure.html#document-domain-feature) to turn off their own ability to use `document.domain`.

Under these conditions, origin-based process isolation could be done by user agents unobservably, similar to how site-based process isolation is done today.

We avoided this approach because of the complexities involved in the use of the `"document-domain"` feature policy. In particular, it can only allow isolation if all documents in a browsing context group impose the policy upon themselves. Since feature policy is a per-document, not per-origin or per-browsing context group configuration, this is hard to guarantee. What if a new page joins the browsing context group, which does not impose the feature policy? There are workarounds, e.g. changing the semantics of `"document-domain"` so that its default allowlist depends on other pages in the browsing context group, but these end up making the `"document-domain"` feature policy no longer mean "disable `document.domain`", but instead have a bunch of other side effects and action at a distance.

### No-op `document.domain` setter instead of throwing

The proposal here is to make the `document.domain` setter do nothing (perhaps emitting a console warning) when used. Throwing an exception would be more semantically correct: it is unable to do the operation you are requesting.

However, throwing would pose a potential migration burden for sites. Measurements from Chrome show that the `document.domain` setter is widely used ([~12% of page views](https://chromestatus.com/metrics/feature/timeline/popularity/739)), but only has an effect a smaller percentage of the time ([~0.28%](https://chromestatus.com/metrics/feature/timeline/popularity/2543) + [~0.52%](https://chromestatus.com/metrics/feature/timeline/popularity/2544) of page views). If we made it throw, then all of that 12% would need to update their code before they could take advantage of origin isolation. By making it do nothing, we instead confine the migration cost to the <1%.

## Adjacent work

### `COOP` + `COEP`

The [`Cross-Origin-Opener-Policy`](https://gist.github.com/annevk/6f2dd8c79c77123f39797f6bdac43f3e) and [`Cross-Origin-Embedder-Policy`](https://mikewest.github.io/corpp/) headers, when combined, also give origin isolation. That is, documents which include both headers will automatically change their agent cluster key to use the origin, instead of the site. As discussed previously, this has observable effects on `document.domain` and `WebAssembly.Module`. Note that, in contrast to the problems explained above with using feature/document policy, this model works, since `COEP` is required to match for all descendant browsing contexts anyway, and `COOP` is consulted via the top-level browsing context.

We are optimistic about this approach for getting the observable effects of origin isolation. In particular, preventing the detrimental effects of `document.domain` is a long-time goal of the web standards and browser community, and tying that to headers like `COOP` + `COEP` and the various carrots they unlock seems like a great strategy.

However, we think that there remains room for this separate origin isolation proposal. In particular:

* Deploying `COOP` + `COEP` can be a significant burden for sites, e.g. in terms of how it requires all of their embedded content to be updated with `CORP`, `COEP`, and `COOP` headers. Some sites may not want to use any of the features enabled by `COOP` + `COEP` (such as `SharedArrayBuffer` or `process.measureMemory()`), and so be unmotivated to take on this burden. But they still might want origin isolation, either for increased encapsulation, or hoping for performance and security benefits. In such cases, this separate header would be much easier to deploy than `COOP` + `COEP`, since it does not require any embeddee cooperation.

* Turning on `COOP` + `COEP` does not provide a clear signal that the origin desires origin isolation; it's instead a technique for ensuring that everything in your process opts in to being there. As such, the presence of a more explicit signal could help the browser decide whether or not to consolidate processes. For example, consider a `COOP`+`COEP`-using site with 30 ad-containing iframes, also using `COOP`+`COEP`. By default, browsers are likely to place all 31 origins into the same process, since they have all appropriately opted in. But if the top-level site had an `Origin-Isolation` header, then the browser could take this as a hint to separate the top-level site into one process, and the ad iframes into another.

So to conclude, the separate `Origin-Isolation` header can be used both by sites who are not yet using `COOP` + `COEP`, to get origin isolation without taking on the burden of cross-origin isolation, and it can be used by sites that are using `COOP` + `COEP`, to provide the browser with a more explicit signal.

## Potential future work

### Hints

A [previous version](https://github.com/WICG/origin-isolation/tree/9d9156e1e5d355cd6156959247ea09eaabd64426) of this proposal included the ability for the web application to specify hints as to why it was requesting origin isolation, to better guide the browser in its resource allocation decisions. For example, a site could use

```
Origin-Isolation: paralllelism
```

to indicate that it was mostly interested in isolating itself to gain better parallelism with same-site cross-origin pages that it would otherwise share resources with. Or a page could use

```
Origin-Isolation: memory-measurement
```

to indicate that the reason it was isolating itself was to get tighter-scoped results from the [`performance.measureMemory()` API](https://github.com/WICG/performance-measure-memory).

We are setting aside this ability for now due to an initial lack of implementer interest. You can see the deliberation process for the Chromium browser in [this document](https://docs.google.com/document/d/1nXxJKuMDNF44vr4TQ219tAz64vPkynGHBDj2MPmpmik/edit#). However, we are happy to add this feature back if any implementation has concrete plans for how they would use the hints. For example, in Chromium we will be closely monitoring whether excessive use of the `Origin-Isolation` header causes us to want hints to help prioritize our process allocation decisions. And we will reevaluate the role of hints when the multiple Blink isolates/multiple Blink threads projects get closer to shipping.

### Origin policy

A [previous version](https://github.com/WICG/origin-isolation/tree/6c35a736792526877b97ecb2250d017a790a2980) of this proposal relied on [origin policy](https://wicg.github.io/origin-policy/). At this time, however, there remain a number of [open design questions](https://github.com/WICG/origin-policy/issues?q=is%3Aissue+is%3Aopen+label%3A%22open+design+question%22) about origin policy, some with no clear path toward a solution. So the plan is to decouple origin isolation from origin policy for now, instead using the `Origin-Isolation` header design seen throughout the rest of this document.

That said, we believe that origin policy would still be an optimal deployment vehicle for many things currently delivered as headers. This includes not only `Origin-Isolation`, but also `Content-Security-Policy`, `Feature-Policy`, `Document-Policy`, `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy`, and more. The same [arguments](https://github.com/WICG/origin-policy#the-problems-and-goals) as to why these would be best configured on an origin-wide basis apply to all of these headers. So for now, the plan is to develop origin isolation in the same way as the others, but also to add it to the list of candidates for integration into origin policy, once origin policy becomes ready.

For reference, a sample origin policy manifest expressing similar values to the header would be something like

```json
{
  "ids": ["my-policy"],
  "isolation": true
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

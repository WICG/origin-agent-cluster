# Origin-keyed agent clusters explainer

**Origin-keyed agent clusters** refers to segregating [cross-origin](https://html.spec.whatwg.org/multipage/origin.html#same-origin) documents into separate [agent clusters](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-cluster-formalism). Translated into developer-observable effects, this means:

* preventing the [`document.domain`](https://html.spec.whatwg.org/multipage/origin.html#relaxing-the-same-origin-restriction) setter from relaxing the same-origin policy; and
* preventing `WebAssembly.Module`s from being shared with cross-origin (but [same-site](https://html.spec.whatwg.org/multipage/origin.html#same-site)) documents.

If a developer chooses to give up these capabilities, then the browser has more flexibility in how it allocates resources like processes, threads, and event loops.

Origin-keyed agent clusters only work in a [secure context](https://w3c.github.io/webappsec-secure-contexts/).

**This proposal was merged into the HTML Standard in [whatwg/html#5545](https://github.com/whatwg/html/pull/5545) under the original name "origin isolation", and renamed in [whatwg/html#6214](https://github.com/whatwg/html/pull/6214). The source of truth for its specification is now [in the HTML Standard](https://html.spec.whatwg.org/multipage/origin.html#origin-keyed-agent-clusters). This repository remains, in archived form, to provide the full explainer and supplementary documents. Issues can be filed [on whatwg/html](https://github.com/whatwg/html/issues).**

_Note: a [previous version](https://github.com/WICG/origin-agent-cluster/tree/6c35a736792526877b97ecb2250d017a790a2980) of this proposal was based on [origin policy](https://wicg.github.io/origin-policy/), but we have now decoupled them. See [below](#origin-policy) for more on this subject._

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
  - [The name of the featurea and header](#the-name-of-the-featurea-and-header)
  - [Using a per-document header](#using-a-per-document-header)
  - [The browsing context group scope of keying](#the-browsing-context-group-scope-of-keying)
  - [Not using feature/document policy](#not-using-featuredocument-policy)
  - [Opt-in, instead of implicit, origin-keying](#opt-in-instead-of-implicit-origin-keying)
  - [No-op `document.domain` setter instead of throwing](#no-op-documentdomain-setter-instead-of-throwing)
  - [`window.originAgentCluster`](#windoworiginagentcluster)
- [Adjacent work](#adjacent-work)
  - [`COOP` + `COEP`](#coop--coep)
- [Potential future work](#potential-future-work)
  - [Hints](#hints)
  - [Origin policy](#origin-policy)
  - [Further isolation](#further-isolation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Example

The delivery mechanism for a web application to achieve origin-keyed agent clusters is a new `Origin-Agent-Cluster` response header:

```
HTTP/1.1 200 OK
Content-Type: text/html
...
Origin-Agent-Cluster: ?1
...
```

The presence of the header, with the [structured headers](https://httpwg.org/http-extensions/draft-ietf-httpbis-header-structure.html) boolean "true" value `?1`, indicates that any documents derived from that origin should be separated from other cross-origin documents—even if those documents are [same site](https://html.spec.whatwg.org/multipage/origin.html#same-site). In terms of observable, specified consequences for web developers, this means:

* Attempts to set `document.domain` will do nothing, so cross-origin documents will not be able to synchronously script each other, even if they are same site.
* Attempts to share `WebAssembly.Module`s to cross-origin documents via `postMessage()` will fail, even if those documents are same site.
* The boolean `window.originAgentCluster` will be true, to provide an easy way of detecting these restrictions without having to try them.

For example, if `https://example.com/` embedded `https://sub.example.com/` as an iframe or opened it as a popup, these two documents would normally be able to synchronously script each other, after both ran the JavaScript code `document.domain = "example.com"`. With origin-keyed agent clusters enabled, setting `document.domain` would instead be a no-op.

Similarly, normally if `https://example.com/` embedded `https://sub.example.com/` as an iframe or opened it as a popup, they would be able to send each other `WebAssembly.Module` instances using `postMessage()`. With origin keying enabled, the `postMessage()` call would instead throw an exception.

(Note that these are only observable consequences in document contexts; for workers, [origin-keying of agent clusters has no effect](#more-detail-on-workers).)

## Objectives

Goals:

* Allow web applications to prevent cross-origin synchronous scripting via `document.domain` (in a guaranteed way).
* Allow user agents to use various implementation strategies to segregate origins when circumstances allow, and the developer has opted in to origin-keyed agent clusters, thus potentially achieving benefits for performance, security, and memory metrics.
* Ensure consistent web-developer-observable behavior regardless of implementation choices such as process allocation. (Modulo side channels.)
* Avoid situations where two same-origin pages with different agent cluster keying preferences are treated as cross-origin by the browser. ([See below](#the-browsing-context-group-scope-of-isolation).)
* Allow web developers to introspect, using client-side scripting, whether their request for origin keying has been honored. ([See below](#windoworiginisolationrestricted).)

Non-goals:

* Mandate process isolation or other implementation choices for user agents.
* Give web developers visibility into process allocation.
* Give guaranteed security benefits, of the sort that can enable powerful features. [`COOP` + `COEP`](#coop--coep) is more appropriate for that.

Non-goals for now:

* "Fully" isolate an origin from all other origins, e.g. by preventing sharing of data via site-scoped storage or navigation. See [below](#further-isolation) for potential future work in that direction.
* Allow web applications to easily configure origin-keying for all URLs on their origin. See [below](#origin-policy) for potential future work in that direction.
* Allow web applications to provide information to the browser about why they want origin-keying, in order to better guide the browser's implementation strategies. See [below](#hints) for potential future work in that direction.

## Implementation strategies

Currently, the state of the art in isolation is process-based. That is, current browser implementation strategies would likely place each origin-keyed origin in its own process. But it's important for applications to recognize that this header does not directly control process allocation, or other behind-the-scenes implementation strategies.

For example, in some cases allocating a process-per-origin-keyed-agent-cluster is infeasible, e.g. because you are operating on a low-memory device, or have already allocated a ton of processes, or because the browser does not implement the ability for iframes to be placed in a separate process. In such scenarios the JavaScript-observable effects of origin-keying will still be present, giving applications an increase in encapsulation. But there wouldn't be any performance or security benefits.

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

If an implementation wants to segregate an _origin_ into its own process, this requires some observable changes. Thus, the opt-in we propose here.

### Specification plan

The specification for this feature in itself consists of three parts. One part is header parsing, which relies on the [Structured Headers](https://httpwg.org/http-extensions/draft-ietf-httpbis-header-structure.html) specification to do most of the work. The other is agent cluster keying, which is done by modifying the HTML Standard's [agent cluster map](https://html.spec.whatwg.org/#agent-cluster-map). And the final part is the `window.originAgentCluster` boolean.

In particular, when creating a document, we modify the algorithm to [obtain a similar-origin window agent](https://html.spec.whatwg.org/#obtain-similar-origin-window-agent). It takes as input an _origin_ of the realm to be created, a browsing context group _group_, and _requestsOAC_ (a boolean, derived from the presence of a valid `Origin-Agent-Cluster` header on a [secure context](https://w3c.github.io/webappsec-secure-contexts/)). It either returns a new or existing agent cluster, depending on these inputs.

The new algorithm will check if _origin_ has previously been placed in a site- or origin-keyed agent cluster within _group_, and if so, it will always return that same agent cluster, to stay consistent. Otherwise, it uses the _requestsOAC_ boolean to determine whether to return an _origin_-keyed agent cluster, or one keyed by the derived site.

The consistency part of this algorithm has two interesting features:

* Even if origin-keying is not requested for a particular document creation, if the user agent has previously created an origin-keyed agent cluster in the browsing context group, then we put the new document in that origin-keyed agent cluster.
* Even when origin-keying is requested, if there has previously been a site-keyed agent cluster with same-origin documents in the browsing context group, then we put the new document in that site-keyed agent cluster, ignoring the origin-keying request.

Both of these are consequences of a desire to ensure that same-origin sites do not end up segregated from each other, even in scenarios involving navigating back and forward.

Because this means that it's possible for the `Origin-Agent-Cluster` header to be present, but origin-keying restrictions to not apply, `window.originAgentCluster` exists to allow web developers to detect these mismatches (which are probably the result of server misconfiguration). That getter will only return `true` if the agent cluster is actually origin-keyed. See [more below](#windoworiginagentcluster).

You can see a more full analysis of what results this algorithm produces in our [scenarios document](./scenarios.md), or you can view the [HTML Standard pull request](https://github.com/whatwg/html/pull/5545).

As far as the web-developer–observable consequences of using origins for agent cluster keys, this will automatically cause `WebAssembly.Module` instances to no longer be shareable cross-origin, because of the agent cluster check in [StructuredDeserialize](https://html.spec.whatwg.org/multipage/structured-data.html#structureddeserialize). We'd also need to add a check to the [`document.domain` setter](https://html.spec.whatwg.org/multipage/origin.html#dom-document-domain) to make it no-op if the surrounding agent's agent cluster is origin-keyed.

### Parallelism

Will origin-keying necessarily increase parallelism? The answer is no; parallelism remains an implementation choice, and is not forced by changing the agent cluster allocation. This is because the relevant specifications are structured to allow non-parallelized implementations, even across totally separate agents. The JavaScript specification [says](https://tc39.es/ecma262/#sec-agents)

> an executing thread may be used as the executing thread by multiple agents

and gives the example of a web browser sharing a single thread across multiple unrelated tabs in a browser window.

In general, the only thing in the specification ecosystem that forces parallelism is workers, via their dedicated executing thread and the requirements around [forward progress](https://tc39.es/ecma262/#sec-forward-progress) and the atomics APIs. So, although origin-keying could incidentally result in better parallelism as part of how implementations respond behind the scenes, parallelism is not automatically forced.

### `SharedArrayBuffer`, `WebAssembly.Memory`, and `WebAssembly.Module`

Does this proposal have any effect on sharing of `SharedArrayBuffer` or `WebAssembly.Memory`? The answer is no, but the reasoning is somewhat subtle.

The [current specification consensus](https://github.com/whatwg/html/pull/4734) is that `SharedArrayBuffer` or `WebAssembly.Memory` will only be structured-serializable when the surrounding agent's agent cluster is _cross-origin isolated_, which means that `COOP` + `COEP` headers have been set. But cross-origin isolation automatically implies origin keying; that is, cross-origin isolation means that `SharedArrayBuffer` and `WebAssembly.Memory` will not be able to be sent cross-origin anyway. See [below](#coop--coep) for more discussion on those two headers, and on the relationship between cross-origin isolation and origin-keyed agent clusters.

(Note that currently in Chromium on desktop platforms, this specification consensus is not yet followed, and `SharedArrayBuffer` and `WebAssembly.Memory` can be sent to cross-origin same-site destinations within the same agent cluster. The plan is for origin-keyed agent clusters to restrict this. But this is a temporary, desktop-Chromium-specific situation, and does not impact the specification proposal.)

Finally, we turn our attention to `WebAssembly.Module`. Structured serialization of `WebAssembly.Module` is restricted to agent clusters, but not for security reasons: the restriction is in place because the point of structured serializing a `WebAssembly.Module` is to avoid recompilation of the module, which is only possible if its destination is within the same process. As such, sending `WebAssembly.Module` around is not guarded by `COOP` + `COEP`, and so origin-keying will observably change the boundaries within which it can be sent, by changing the scope of agent clusters to be origin-keyed instead of site-keyed. This is why, in addition to the synchronous scripting restrictions, we mention the changes to `WebAssembly.Module` serialization as an observable consequence of origin-keyed agent clusters.

### More detail on workers

We mentioned above that origin keying has no effect on workers. Here we explain why exactly that is.

First note that shared and service workers are already segregated into their own agent clusters. The only other things that exist in those agent clusters are potential nested dedicated workers, which are already restricted to being same-origin.

What remains to consider is dedicated workers which are spawned directly by documents. Those end up in the agent cluster of their owner document.

Does the owner document's `Origin-Agent-Cluster` header impact any code inside such a decicated worker? The answer is no. All access to other realms is asynchronous inside a dedicated worker, so `document.domain` does not apply. And there's no way for dedicated worker code to send a `WebAssembly.Module` to a same-site cross-origin destination directly, because the only thing it can communicate with is its parent document, or further nested dedicated workers, all of which are same-origin. Note that even `BroadcastChannel`, which bypasses the need to have a direct reference to the destination, is restricted to same-origin communications.

A worker could send a `WebAssembly.Module` to a same-site cross-origin destination by first passing it (or a `MessageChannel` through which the `WebAssembly.Module` gets transmitted) to a same-origin document, and letting the document try to go beyond the origin boundary. This would no longer be possible with this proposal, but only because of the proposal's effect on the document, not because of any effect it had on the worker.

## Design choices and alternatives considered

### The name of the featurea and header

"Origin-keyed agent clusters" sure is a mouthful, with lots of jargon that web developers are not usually exposed to. However, this is actually desirable.

A previous revision of this proposal used the name "origin isolation", with a header named `Origin-Isolation`. Unfortunately, we found that in practice this gave the wrong impression: people thought that the header gave guaranteed process isolation (similar to Chrome's site isolation feature), or gave security benefits (similar to the cross-origin isolation feature discussed [below](#coop--coep)). You can read more about this in [whatwg/html#6192](https://github.com/whatwg/html/issues/6192).

By using a jargon-full name, we are communicating that this feature is subtle in its effects. It also makes it clear that we are isolating in very specific ways, that do not touch on the other ideas discussed [below](#further-isolation).

### Using a per-document header

This proposal follows in the tradition of other security primitives like CSP, COEP, and COOP, by using a header to change the behavior of the document.

A previous iteration of this proposal used [origin policy](https://wicg.github.io/origin-policy/), in an attempt to ensure that all documents on the origin are aligned behind the decision to be origin-keyed. We believe this is still a desirable direction, both for the `Origin-Agent-Cluster` header and for other security headers. But it's worth noting that the advantages are mainly in easing server configuration. Because different page loads on an origin could recieve different origin policies anyway (due to, e.g., server-side updates), there is no technical simplification gained by using origin policy. That is, the browser and site developer need to deal with the potential for varying origin-keying preferences across the site in both delivery mechanisms.

See [below](#origin-policy) for how this proposal might be upgraded to use origin policy in the future.

### The browsing context group scope of keying

The [scenarios document](./scenarios.md) explains how the current specification plan works in tricky scenarios involving different `Origin-Agent-Cluster` headers between loads of realms from a given origin. In short, the keying is determined on a per-browsing context group basis.

We considered two other possible models:

* Per-realm: in this model, we would not check for the existence of other site-keyed realms within the browsing context group, and would always origin-key realms which request it, and never origin-key ones that do not.
* Per-user agent: in this model, we would keep a global list of all origins which have ever requested origin-keying, and consult this first whenever creating a new realm.

The downside of the per-realm model is that it leads to strange scenarios like `https://b.example.com/1` and `https://b.example.com/2` being in different agent clusters: one of them site-keyed, and the other origin-keyed. This is not desirable, as same-origin pages should not be segregated from each other by this feature.

The downside of the per-user agent model is that it makes it very hard for web application developers to roll back origin-keying. They essentially have to communicate to the user to restart their browser.

### Not using feature/document policy

Why did we choose to invent a new header for origin-keyed agent clusters, instead of using [feature policy](https://github.com/w3c/webappsec-feature-policy) or [document policy](https://github.com/w3c/webappsec-feature-policy/blob/master/document-policy-explainer.md)?

It turns out, both of these mechanisms are designed for dealing with embedded content, and as such have inheritance models that don't make much sense for origin keying. If you want to origin-key your own origin, that doesn't necessarily mean anything about how your subframes should behave—especially considering that the effects of feature/document policy are not limited to same-origin, or even same-site, subframes.

### Opt-in, instead of implicit, origin-keying

Another potential road we explored was to try to find scenarios where a process-per-origin could be done transparently, without an explicit opt-in. This would be possible if:

* We [restricted `WebAssembly.Module` to same-origin usage](https://github.com/WebAssembly/spec/issues/1081) at all times. (It's currently unknown if doing this would be web-compatible.)
* Pages used the [`"document-domain"` feature policy](https://html.spec.whatwg.org/multipage/infrastructure.html#document-domain-feature) to turn off their own ability to use `document.domain`.

Under these conditions, origin-based process isolation could be done by user agents unobservably, similar to how site-based process isolation is done today.

We avoided this approach because of the complexities involved in the use of the `"document-domain"` feature policy. In particular, it can only allow origin keying if all documents in a browsing context group impose the policy upon themselves. Since feature policy is a per-document, not per-origin or per-browsing context group configuration, this is hard to guarantee. What if a new page joins the browsing context group, which does not impose the feature policy? There are workarounds, e.g. changing the semantics of `"document-domain"` so that its default allowlist depends on other pages in the browsing context group, but these end up making the `"document-domain"` feature policy no longer mean "disable `document.domain`", but instead have a bunch of other side effects and action at a distance.

### No-op `document.domain` setter instead of throwing

The proposal here is to make the `document.domain` setter do nothing (perhaps emitting a console warning) when used. Throwing an exception would be more semantically correct: it is unable to do the operation you are requesting.

However, throwing would pose a potential migration burden for sites. Measurements from Chrome show that the `document.domain` setter is widely used ([~12% of page views](https://chromestatus.com/metrics/feature/timeline/popularity/739)), but only has an effect a smaller percentage of the time ([~0.28%](https://chromestatus.com/metrics/feature/timeline/popularity/2543) + [~0.52%](https://chromestatus.com/metrics/feature/timeline/popularity/2544) of page views). If we made it throw, then all of that 12% would need to update their code before they could take advantage of origin-keyed agent clusters. By making it do nothing, we instead confine the migration cost to the <1%.

### `window.originAgentCluster`

The `window.originAgentCluster` getter can feel a bit redundant. After all, the web developer should know whether origin keying applies: they set the header themselves!

However, as explained [above](#specification-plan), the `Origin-Agent-Cluster` header cannot always be respected. If a web developer has configured their server inconsistently, or if they are in the middle of changing their server configuration, other same-origin documents in the browsing context group might have different values for the header, and the overriding consistency requirement would then take over. Thus, `window.originAgentCluster` gives runtime insight into whether the origin-keying request succeeded, which can be useful for reporting, or for detecting misconfigurations.

There are some subtleties around what this getter should return in various edge cases. As indicated by the name, it's currently designed to return true whenever origin keying is in play. This means that it always returns `true` for  pages with opaque origins, because for those cases [the page's site is the same as its origin](https://html.spec.whatwg.org/#obtain-a-site). I.e., those cases can be considered automatically origin-keyed, regardless of the header. This was discussed in more detail in [#24](https://github.com/WICG/origin-agent-cluster/issues/24) and [#31](https://github.com/WICG/origin-agent-cluster/issues/31), which contemplated other alternatives that give different answers in these edge cases. See also [whatwg/html#5940](https://github.com/whatwg/html/issues/5940).

Note that `window.originAgentCluster` is present even in non-secure contexts. This maintains consistency with `window.crossOriginIsolated`; both are useful for detecting what effects headers have on a page, even if the corresponding headers are only usable in secure contexts. (Also, note that in the edge cases mentioned above, the getter can return `true` even in a non-secure context.)

Finally, there's the question as to whether the getter should be exposed in workers. Currently it is not, since [as noted above](#more-detail-on-workers), origin keying has no direct effects on workers, so we don't anticipate this being very useful. However, it is conceptually coherent to expose it; it would just always return true for shared/service workers, and would return the same value as the owner `Window` for dedicated workers, indicating the indirect effect that the owner `Window` has. We're open to doing so in the future if any use cases are provided.

## Adjacent work

### `COOP` + `COEP`

The [`Cross-Origin-Opener-Policy`](https://gist.github.com/annevk/6f2dd8c79c77123f39797f6bdac43f3e) and [`Cross-Origin-Embedder-Policy`](https://mikewest.github.io/corpp/) headers, when combined, also give origin-keyed agent clusters. That is, documents which include both headers will automatically change their agent cluster key to use the origin, instead of the site. As discussed previously, this has observable effects on `document.domain` and `WebAssembly.Module`. Note that, in contrast to the problems explained above with using feature/document policy, this model works, since `COEP` is required to match for all descendant browsing contexts anyway, and `COOP` is consulted via the top-level browsing context.

We are optimistic about this approach for getting the observable effects of origin keying. In particular, preventing the detrimental effects of `document.domain` is a long-time goal of the web standards and browser community, and tying that to headers like `COOP` + `COEP` and the various carrots they unlock seems like a great strategy.

However, we think that there remains room for this separate `Origin-Agent-Cluster` proposal. In particular:

* Deploying `COOP` + `COEP` can be a significant burden for sites, e.g. in terms of how it requires all of their embedded content to be updated with `CORP`, `COEP`, and `COOP` headers. Some sites may not want to use any of the features enabled by `COOP` + `COEP` (such as `SharedArrayBuffer` or `process.measureMemory()`), and so be unmotivated to take on this burden. But they still might want origin-keyed agent clusters, either for increased encapsulation, or hoping for performance and security benefits. In such cases, this separate header would be much easier to deploy than `COOP` + `COEP`, since it does not require any embeddee cooperation.

* Turning on `COOP` + `COEP` does not provide a clear signal that the origin desires origin keying; it's instead a technique for ensuring that everything in your process opts in to being there. As such, the presence of a more explicit signal could help the browser decide whether or not to consolidate processes. For example, consider a `COOP`+`COEP`-using site with 30 ad-containing iframes, also using `COOP`+`COEP`. By default, browsers are likely to place all 31 origins into the same process, since they have all appropriately opted in. But if the top-level site had an `Origin-Agent-Cluster` header, then the browser could take this as a hint to separate the top-level site into one process, and the ad iframes into another.

So to conclude, the separate `Origin-agent-Cluster` header can be used both by sites who are not yet using `COOP` + `COEP`, to get origin keying without taking on the burden of cross-origin isolation, and it can be used by sites that are using `COOP` + `COEP`, to provide the browser with a more explicit signal.

## Potential future work

### Hints

A [previous version](https://github.com/WICG/origin-agent-cluster/tree/9d9156e1e5d355cd6156959247ea09eaabd64426) of this proposal included the ability for the web application to specify hints as to why it was requesting origin keying, to better guide the browser in its resource allocation decisions. For example, a site could use

```
Origin-Agent-Cluster: ?1;why=parallelism
```

to indicate that it was mostly interested in origin-keying itself to gain better parallelism with same-site cross-origin pages that it would otherwise share resources with. Or a page could use

```
Origin-Agent-Cluster: ?1;why=memory-measurement
```

to indicate that the reason it was origin-keying itself was to get tighter-scoped results from the [`performance.measureMemory()` API](https://github.com/WICG/performance-measure-memory).

We are setting aside this ability for now due to an initial lack of implementer interest. You can see the deliberation process for the Chromium browser in [this document](https://docs.google.com/document/d/1nXxJKuMDNF44vr4TQ219tAz64vPkynGHBDj2MPmpmik/edit#). However, we are happy to add this feature back if any implementation has concrete plans for how they would use the hints. For example, in Chromium we will be closely monitoring whether excessive use of the `Origin-Agent-Cluster` header causes us to want hints to help prioritize our process allocation decisions. And we will reevaluate the role of hints when the multiple Blink isolates/multiple Blink threads projects get closer to shipping.

### Origin policy

A [previous version](https://github.com/WICG/origin-agent-cluster/tree/6c35a736792526877b97ecb2250d017a790a2980) of this proposal relied on [origin policy](https://wicg.github.io/origin-policy/). At this time, however, there remain a number of [open design questions](https://github.com/WICG/origin-policy/issues?q=is%3Aissue+is%3Aopen+label%3A%22open+design+question%22) about origin policy, some with no clear path toward a solution. So the plan is to decouple origin-keyed agent clusters from origin policy for now, instead using the `Origin-Agent-Cluster` header design seen throughout the rest of this document.

That said, we believe that origin policy would still be an optimal deployment vehicle for many things currently delivered as headers. This includes not only `Origin-Agent-Cluster`, but also `Content-Security-Policy`, `Feature-Policy`, `Document-Policy`, `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy`, and more. The same [arguments](https://github.com/WICG/origin-policy#the-problems-and-goals) as to why these would be best configured on an origin-wide basis apply to all of these headers. So for now, the plan is to develop origin-keyed agent clusters in the same way as the others, but also to add it to the list of candidates for integration into origin policy, once origin policy becomes ready.

For reference, a sample origin policy manifest expressing similar values to the header would be something like

```json
{
  "ids": ["my-policy"],
  "origin_keyed_agent_clusters": true
}
```

### Further isolation

This proposal is focused narrowly on the agent cluster allocation issue, and its implications for synchronous cross-origin access which has various security and performance effects.

One might envision others was of isolating an origin that go beyond this. Many such concepts have been discussed in a previous ["Isolation Explainer"](https://wicg.github.io/isolation/explainer.html) from 2016. Some, like `window.opener` restriction or preventing cross-origin fetches, have since found solutions. (Respectively, [`Cross-Origin-Opener-Policy`](https://gist.github.com/annevk/6f2dd8c79c77123f39797f6bdac43f3e) and [`Cross-Origin-Resource-Policy`](https://fetch.spec.whatwg.org/#cross-origin-resource-policy-header)/[`Sec-Fetch-Site`](https://www.w3.org/TR/fetch-metadata/#sec-fetch-site-header).) The ones that remain appear to be:

* Preventing the origin from sharing storage (of [various types](https://storage.spec.whatwg.org/#infrastructure)) with different origins, even if those origins are same-site. This would include things like cookies or [Web Authentication](https://w3c.github.io/webauthn/) entries.
* Double-keying cookies, so that when an isolated origin makes a request to `https://third-party.example/`, this uses a separate cookie jar from when any other origin makes a request to `https://third-party.example/`.
* Preventing navigations from other origins to this origin (or perhaps allowing only specific entry points as navigation targets).

We believe that, after tackling agent cluster allocation, these may be worthwhile further efforts to pursue. They would probably use separate headers, however.

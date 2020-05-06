# Origin isolation scenarios

This document considers various combinations of documents being loaded, with different relationships to each other and different origin isolation signals, to illustrate the consequences of the [proposed algorithm](./README.md#specification-plan).

Notes:

* We use `https://e.com` as our example domain for brevity. "e" stands for "example" but is short enough to fit into tables.
* All of these considerations apply for popups, not just frames, in the same ways.
* All of these considerations apply for workers, not just documents, in the same ways.
* We use the notation `Origin{}` and `Site{}` to denote the type of agent cluster key.

## Two-document scenarios

| Main frame              | Subframe                 | Result                                                               |
| ------------------------|--------------------------|----------------------------------------------------------------------|
| `https://e.com` w/ OI   | `https://e.com` w/ OI    | Both in `Origin{https://e.com}`                                      |
| `https://e.com` w/ OI   | `https://e.com` w/o OI   | Both in `Origin{https://e.com}`                                      |
| `https://e.com` w/o OI  | `https://e.com` w/o OI   | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://e.com` w/ OI    | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://x.e.com` w/o OI | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/ OI   | `https://x.e.com` w/o OI | Main in `Origin{https://e.com}` <br>Sub in `Site{https://e.com}`     |
| `https://e.com` w/ OI   | `https://x.e.com` w/ OI  | Main in `Origin{https://e.com}` <br>Sub in `Origin{https://x.e.com}` |

## Three-document scenarios

| Main frame                 | Subframe A                  | Subframe B                 | Result                                                               |
| ---------------------------|-----------------------------|----------------------------|----------------------------------------------------------|
| `https://e.com`<br>w/ OI   | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/o OI  | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Site{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/ OI   | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |
| `https://e.com`<br>w/o OI  | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main and A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |


## Session history scenarios

The previous scenarios illustrate some cases where the proposed algorithm ignores the origin isolation request because a pre-existing site-keyed agent cluster exists for a same-origin URL in the current frame tree. The following scenario illustrates a case where the session history contributes to this calculation.

### Navigating a subframe

* Main frame: `https://e.com` w/ OI
* Subframe: `https://x.e.com` w/o OI
* Outcome (per above): Main in `Origin{https://e.com}` / Sub in `Site{https://e.com}`
* Historical map of agent cluster keys for this browsing context group: `(https, e.com, 443) => Origin{https://e.com}`, `(https, x.e.com, 443) => Site{https://e.com}`

The user clicks a link in the `https://x.e.com` subframe which takes them to `https://e.org`.

The user then clicks a button in the main frame which inserts a second subframe, pointing at `https://x.e.com`. But in the meantime the server operator has deployed origin isolation on `https://x.e.com`. Thus the scenario is now:

* Main frame: `https://e.com` w/ OI
* Subframe 1: `https://e.org` (with `https://x.e.com` in the session history)
* Subframe 2: `https://x.e.com` w/ OI

In this case, subframe 2 ends up keyed by `Site{https://e.com}`, because when loading `https://x.e.com`, we look up `(https, x.e.com, 443)` in the historical map of agent cluster keys, and see that it was originally allocated to `Site{https://e.com}`.

This ensures that if the user navigates subframe 1 back, `https://x.e.com` again loads in `Site{https://e.com}`, as it did originally, and subframe 1 and subframe 2 are in the same `Site{https://e.com}` agent cluster. That is, we have avoided isolating same-origin pages from each other. And we've done so by using the rule of "first-seen agent cluster key wins", via the historical map of agent cluster keys.

### Inserting iframes and saving JS references

* Main frame: `https://e.com` w/ OI
* Subframe: `https://e.org` w/o OI
* Outcome (per above): Main in `Origin{https://e.com}` / Sub in `Site{https://e.org}`.
* Historical map of agent cluster keys for this browsing context group: `(https, e.com, 443) => Origin{https://e.com}`, `(https, e.org, 443) => Site{https://e.org}`

JavaScript code in the main frame goes through a variety of contortions:

```js
// Save a reference to the sub-frame window.
window.savedFrame = frames[0];

// Now remove it from the DOM.
frames[0].frameElement.remove();
```

Some time later, JavaScript code inserts a new subframe, again pointing at `http://e.org`. But in the meantime the server operator has deployed origin isolation on `https://e.org`. Thus the scenario is now:

* Main frame: `https://e.com` w/ OI
* Subframe: `https://e.org` w/ OI

Will the subframe get site-keyed, or origin-keyed?

The answer is site-keyed. We only consult the historical map of agent cluster keys, which says that `(https, e.org, 443)` goes in `Site{https://e.org}`. The fact that the iframe was removed (so no corresponding browsing context exists in the browsing context group), or that the `Window` was saved (so that the realm/agent/agent cluster still exist) do not impact our decision-making process here. We always consult the historical map of agent cluster keys to make the decision.

## Worked-out nested scenario

The following is a more detailed scenario, involving more levels of nesting and dynamic insertion of new frames. It illustrates the same fundamental principles as the above examples, but it goes through them in more detail, which might be helpful.

Consider the following server setup:

| URL                      | Has `Origin-Isolation` response header |
| -------------------------|----------------------------------------|
| `https://a.example.com`  | Yes                                    |
| `https://b.example.com/1`| Yes                                    |
| `https://b.example.com/2`| No                                     |
| `https://c.example.com`  | No                                     |
| `https://d.example.com`  | No                                     |

Now consider the following series of nested iframes:

```
https://example.org
└ https://a.example.com
  └ https://b.example.com/1
  └ https://b.example.com/2
    └ https://c.example.com
    └ https://d.example.com
```

Now, consider what happens if `https://b.example.com/2` tries to `postMessage()` a `WebAssembly.Module` to `https://c.example.com`?

The answer that the [proposed algorithm](./README.md#specification-plan) gives is that the `postMessage()` fails. Within the browsing context group, we find the origin `https://b.example.com` used as an agent cluster key, so even though the `https://b.example.com/2` iframe was loaded without the `Origin-Isolation` header, it still gets origin-isolated. The allocation is:

- `Origin{https://a.example.com}`: `https://a.example.com` page;
- `Origin{https://b.example.com}`: `https://b.example.com/1` page, `https://b.example.com/2` page;
- `Site{https://example.com}`: `https://c.example.com` page, `https://d.example.com` page.

OK, let's go further. While we have this tab open, the server operator makes some changes, removing `https://b.example.com/1`'s `Origin-Isolation` header:

| URL                      | Has `Origin-Isolation` response header |
| -------------------------|----------------------------------------|
| `https://a.example.com`  | Yes                                    |
| `https://b.example.com/1`| No                                     |
| `https://b.example.com/2`| No                                     |
| `https://c.example.com`  | No                                     |

If the user then opens a new tab to `https://example.org`, will the `postMessage()` succeed in the new tab?

Yes. This time, we are in a new browsing context group, with a new agent cluster map. So this time, the iframes are not origin-isolated (i.e. everything is using site agent cluster keys), and the sharing succeeds.

That is, within this separate browsing context group, the allocation is:

- `Origin{https://a.example.com}`: `https://a.example.com` page;
- `Site{https://example.com}`: `https://b.example.com/1` page, `https://b.example.com/2` page, `https://c.example.com` page, `https://d.example.com` page.

This means that, if you want to "un-isolate" your origin, you can to do so via a new, disconnected browsing context group, separate from the isolated one.

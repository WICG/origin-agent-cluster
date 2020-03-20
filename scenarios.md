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
| `https://e.com` w/o OI  | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/o OI  | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/ OI   | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/ OI   | `https://x.e.com` w/o OI | Main in `Origin{https://e.com}` <br>Sub in `Site{https://e.com}`     |

## Three-document scenarios

| Main frame                 | Subframe A                  | Subframe B                 | Result                                                               |
| ---------------------------|-----------------------------|----------------------------|----------------------------------------------------------|
| `https://e.com`<br>w/ OI   | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/o OI  | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Site{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/ OI   | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |
| `https://e.com`<br>w/o OI  | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main and A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |

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

Now, consider what happens if `https://b.example.com/2` tries to `postMessage()` a `SharedArrayBuffer` to `https://c.example.com`?

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

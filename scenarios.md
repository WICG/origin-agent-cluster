# Origin isolation scenarios

This document considers various combinations of documents being loaded, with different relationships to each other and different origin isolation signals, to illustrate the consequences of the [proposed algorithm](./README.md#specification-plan).

Notes:

* We use `https://e.com` as our example domain for brevity. "e" stands for "example" but is short enough to fit into tables.
* All of these considerations apply for popups, not just frames, in the same ways.
* All of these considerations apply for workers, not just documents, in the same ways.
* We use the notation `Origin{}` and `Site{}` to denote the type of agent cluster key.
* We assume that "blocking" origin policies are used at all times, without async updates. (Async updates would just mean that we re-run these scenarios on a future load.)

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

We start out as follows:

* `https://example.org/` embeds an iframe for `https://a.example.com/` which embeds an iframe for `https://b.example.com/1`
* We open tab #1 to `https://example.org/`. Both `https://a.example.com/` and `https://b.example.com/1` responses point to origin policies with `"isolation": true` set.

Now, things get fun:

* While we have tab #1 open, the server operator updates both `https://a.example.com/.well-known/origin-policy` and `https://b.example.com/.well-known/origin-policy` to set `"isolation": false`.
* Then, in tab #1, `https://a.example.com/` inserts a new iframe, pointing to `https://b.example.com/2`. Since the `https://b.example.com/` policy on the server has been updated, the response used for creating this second child iframe is no longer requesting origin isolation.
* This `https://b.example.com/2` iframe inserts an iframe for `https://c.example.com/`, which has no origin policy. Then `https://b.example.com/2` tries to `postMessage()` a `SharedArrayBuffer` to `https://c.example.com/`.

What happens?

The answer that the [proposed algorithm](./README.md#specification-plan) gives is that the `postMessage()` fails. Within the browsing context group (i.e. tab #1), we find the origin `https://b.example.com/` used as an agent cluster key, so even though the `https://b.example.com/2` iframe was loaded with an origin policy saying `"isolation": false`, it still gets origin-isolated.

OK, let's go further.

* Now we open up a new tab to `https://example.org/`, tab #2. Because of the server update, the origin policies corresponding to the nested iframes for `https://a.example.com/` and `https://b.example.com/1` have `"isolation": false` set.
* The `https://a.example.com/` iframe tries to `postMessage()` a `SharedArrayBuffer` to the `https://b.example.com/1` iframe.

What happens this time?

This time, we are in a new browsing context group, with a new agent cluster map. So this time, the iframes are not origin-isolated (i.e. everything is using site agent cluster keys), and the sharing succeeds.

This means that, if you want to "un-isolate" your origin, you can to do so via a new, disconnected browsing context group, separate from the isolated one.

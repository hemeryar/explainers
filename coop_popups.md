# Cross-Origin-Opener-Policy: Popups

## Terminology
In this explainer, we call _document_ what is rendered by any individual frame. A _page_ contains the top level document, as well as all of its iframe documents, if any. When you open a popup, you get a second page.

## Motivations
We would like websites to improve their performance using SharedArrayBuffers and be able to use cross-origin popup OAuth and payment flows at the same time. This would allow users to benefit from fast websites that support easy to use sign-in and payment flows. This has been asked by websites like Zoom, which wants to use SAB for better video conference performance while preserving existing OAuth flows.

SAB, among a couple of other APIs are considered dangerous because they can be used to create high precision timers that enable much easier exploitation of Spectre. For that reason, they should only be used for documents that can be process isolated, and that's what browsers have done, by gating them behind `window.crossOriginIsolated`. This bit is turned to true when COOP is Same-Origin, COEP: `require-corp` or `credentialless`, and for subframes, when their top level document sets the crossOriginIsolated `Permissions-Policy`.

`Cross-Origin-Opener-Policy` (COOP) and `Cross-Origin-Embedder-Policy` (COEP) have now been available for a while, but deployment has been slow and there are still significant pain points for web developers wishing to use these headers. Permissions-Policy is easy to deploy, and COEP has been at the core of previous efforts, with features like credentialless being designed explicitly to help with deployment. This document focuses on the difficulties raised by COOP.

Currently, all pages that have legitimate interactions with cross-origin popups and that want to use COOP need to set `COOP: Same-Origin-Allow-Popups` and the popup needs to set `COOP: Unsafe-None`. Cross-origin popups are used for important features like OAuth and payments, that cannot be implemented in a different way. That means any page that uses one of these services cannot use SharedArrayBuffers. It also means that cross-origin service providers cannot set a COOP value to protect themselves against side-channel leaks.

## Changing our approach
It is interesting to get back to the core of what COOP does. It restricts what BrowsingContext can and cannot be in a BrowsingContext group, depending on their top-level origin and COOP value. The initial goal was to be able to put different pages in different processes without breaking scripting invariants, without making assumptions about browser capabilities:

* Synchronous access of DOM requires that the documents live in the same process.
* Asynchronous access of a popup's windowProxy property can be out of process, if the browser supports page isolation. Page isolation is the capability to communicate asynchronously between two pages when they live in two different processes.
* Asynchronous access of an iframe's windowProxy property can be out of process, if the browser supports out-of-process iframes (OOPIF). OOPIF is the capability to communicate asynchronously between an iframe and another document when they live in another process. Note that this has an important impact on memory.

By severing the opener/openee link, it made sure that two pages wouldn't be able to communicate with each other and that they could be put in different processes, regardless of whether we have page isolation or OOPIF. For some use cases however, this does not provide a satisfactory solution.

</br>

![image](resources/coop_basic_issue.png)  
_The basic case COOP solves. Without COOP, we have to put all the documents in the same process, because the popup and the iframe are of origin b.com and have synchronous access to each other._

</br>

In the specification, documents that have synchronous DOM access live in the same AgentCluster. Pages that have asynchronous WindowProxy access live in the same BrowsingContext group. That means all documents in an AgentCluster must live in the same process, and that there is no fundamental reason why a BrowsingContext group should be represented by a single process, it is only an implementation limitation.

In a world where all browsers support OOPIF and page isolation, we can simply put same origin documents within their own process, without having COOP in the first place. Realistically, OOPIF is a complex piece of machinery and is not supported by all browsers and will not be in the near future, but page isolation already is. Both Firefox and Safari support page isolation, and it is a reasonable feature to build upon. The fundamental idea is that if we leverage page isolation we can create a more flexible solution than what exists today using only BrowsingContext groups.

The solution we propose is to add (yet) another COOP value: `Popups` (name likely to change). It says: _"Pages that set a different COOP policy or have different top level origin will be placed in a different BrowsingContext group. However the opener relationship will not be severed, but the only windowProxy properties that remain accessible through it are postMessage and closed."_ Served together with a suitable COEP policy it would enable `crossOriginIsolated`.


## Technical details of "COOP: Popups"
We're leveraging Page Isolation to transform _"No communication can happen cross page"_ into _"Only limited asynchronous communications can happen cross page"_.

This is not currently possible in the HTML spec because no "relationship" exists between BrowsingContext groups, but nothing prevent us from making that change. We essentially add the possibility to create a BrowsingContext group with an opener, and include a BrowsingContext group verification in the WindowProxy setter and getters that verifies we're not trying to access non-allowed properties accross this new link.

An alternative would be to not swap BrowsingContext group and add top-level origin, popup policy and cross-origin isolation state to the AgentCluster, to prevent synchronous communications across the opener link, all within a BrowsingContext group. The first solution has the benefit of keeping the AgentCluster simple (just an origin or a site), while preserving all the usual behavior of BrowsingContext groups (including things like name targeting, etc.).

TODO: Describe navigating popups to a different BCG in a popup, do we preserve the link?

## Interactions with other COOP values
The COOP algorithm needs to be adapted slightly. Since there is currently only one type of "BCG swap", everything works well, but we need to redefine which type of swap happens when there is two possibilities. One way we can tackle this is:

- We always do a BCG swap without an opener when `Same-Origin` is involved.
- `Same-Origin-Allow-Popups` can open `Unsafe-None` in the same BCG, and `Popups` in a different BCG with an opener.
- Pages with matching `Popups` and origins stay in the same BCG.

To sum up, `Popups` can be opened by `Unsafe-None` or `Same-Origin-Allow-Popups` in a different BCG with opener, and can open `Unsafe-None` in a different BCG with opener. The only practical modification needed is the extra special case in `Same-Origin-Allow-Popups` algorithm.

We could also ask whether COOP: `Same-Origin-Allow-Popups` is still useful. It says both _"I'm okay preserving scripability to popups I open"_ but _"I'm not okay being opened as a popup"_. This is not something you can achieve using `Popups` which is symetrical. Whether or not we want to preserve this capability is not yet decided.

## Security notes
Purposefully chosing the absolute minimum set of windowProxy attributes to make OAuth possible helps prevent side-channel leaks, which has been a concern both for openers and openees.

DOM access is not the only thing that is gated behind same-origin restrictions. We audited the spec to produce a [list](https://docs.google.com/spreadsheets/d/1e6LakHSKTD22XEYfULUJqUZEdLnzynMaZCefUe1zlRc/) of all places with such checks. Some points worthy of attention:

* The location object is quite sensitive and many of its methods/members are same-origin only. It is purposefully excluded from the list of allowed attributes by `Popups`, despite fairly high usage. We do not think we should allow a normal page to navigate a crossOriginIsolated page.
* For similar reasons name targeting does not go across BrowsingContext groups, and this change shouldn't change that.
* Javascript navigations are a big NO. They mean executing arbitrary javascript within the target frame. There should be no way to navigate a frame from a different BrowsingContext group with different crossOriginIsolated or COOP: `Popups` value given the restrictions above.
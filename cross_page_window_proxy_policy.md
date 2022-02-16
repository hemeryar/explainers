# Cross-Page-Window-Proxy-Policy explainer

## Motivations
Cross-Origin-Opener-Policy (COOP) and Cross-Origin-Embedder-Policy (COEP) have now been available for a while and with them the gating of certain powerful APIs like SharedArrayBuffer behind the crossOriginIsolated bit. This bit is turned to true when COOP is Same-Origin, COEP: require-corp or credentialless, and for subframes, when their top level document sets the crossOriginIsolated Permissions-Policy. Deployment has been slow and there are still significant pain points for web developers wishing to use these headers. Permissions-Policy is easy to deploy, and COEP has been at the core of previous efforts, with features like credentialless being designed explicitly to help with deployment. This document focuses on the difficulties raised by COOP in particular.

Maybe it is interesting to get back to the core of what COOP does. It restricts what BrowsingContext can and cannot be in a BrowsingContext group, depending on their top-level origin and COOP value. The initial goal was to be able to put different pages in different processes without breaking scripting invariants. That's important because crossOriginIsolated only applies to entire processes. By severing the opener/openee link, it made sure that two pages wouldn't be able to communicate with each other and that they could be put in different processes. For some use cases however, this does not provide a satisfactory solution.

Currently, all pages that have legitimate interactions with cross-origin popups and that want to use COOP need to set COOP: Same-Origin-Allow-Popups. That implies that the main page cannot be crossOriginIsolated, and that the popup must have COOP: Unsafe-None. This is not ideal, as cross-origin popups are used for important features like OAuth and payments, that do not have an alternative. On top of that, popups providing these services would like to also protect themselves against side-channel leaks using COOP.



## Changing our approach
We said above that COOP had to go around some basic scripting invariants. They are:

* Synchronous access of DOM requires that the documents live in the same process.
* Asynchronous access of a popup's windowProxy property can be out of process, if the browser supports page isolation.
* Asynchronous access of an iframe's windowProxy property can be out of process, if the browser supports out-of-process iframes (OOPIF).

In the specification, documents that have synchronous DOM access live in the same AgentCluster. Pages that have asynchronous WindowProxy access live in the same BrowsingContext group. By adding these constructs with the invariants above we get that:

* All documents in an AgentCluster must live in the same process.
* There is no fundamental reason why a BrowsingContext group should be represented by a single process, only implementation limitations.

In a world where all browsers support OOPIF and page isolation, we can simply put same origin documents within their own process, without having COOP in the first place. Realistically, this is not the case for OOPIF, but it is the case for page isolation. Both Firefox and Safari support page isolation, and it is a reasonable thing to expect.

The fundamental idea is that if we leverage page isolation we can create a more flexible solution than what exists today using only BrowsingContext groups.


## Introducing Cross-Page-Window-Proxy-Policy
The solution we propose is to add (yet) another header policy, meant for the top level document: Cross-Page-Window-Proxy-Policy. It says: _"This page is not interested in communicating synchronously with other pages, unless they're same-origin and have the same policy."_

By doing that, we essentially modify the keying of AgentClusters, that groups all documents within a BrowsingContext group, as long as they were same origin. We now say that on top of being same-origin they must have the same Cross-Page-Window-Proxy-Policy. We can now have Cross-Page-Window-Proxy-Policy + COEP enable crossOriginIsolated:

* Cross-Page-Window-Proxy-Policy applies to entire pages, so the state is consistent within a page.
* COEP is consistent within a page, or iframes will not load.
* crossOriginIsolated is consistent within a page.
* Page isolation is well supported by browsers.

Integrating with cross-origin popup flows only requires asynchronous communication since they're cross-origin. These flow can work as intended. We've solved the first integration issue of having both crossOriginIsolated and a popup flow.

## Using Cross-Page-Window-Proxy-Policy for side-channel protection
The problem of providing side-channel protection is still not solved. This is what the specific values of the 
Cross-Page-Window-Proxy-Policy are meant for. For now we plan to only give it only one possible value:
* Opener-Close-PostMessage

On top of limiting synchronous communications with same-origin pages that do not share this policy, it would also prevent all asynchronous access to WindowProxies for things that are not window.closed(), window.opener() and window.postMessage. 

Some [metrics](https://docs.google.com/spreadsheets/d/1fzqEzyINWlDCo-yIeasWiCdEBF3MG8pbIQlC4wiUiug/edit#gid=0) were produced in chrome to help determine if we could have a single value or if the policy would need to have finer grain. What comes out of it is that preserving these windowProxy attributes works for an overwhelming majority.

## Interactions with COOP
We could also ask whether COOP: Same-Origin-Allow-Popups is still useful. The short answer is yes, because it not only says _"I'm okay preserving scripability to popups I open"_. It also says _"I'm not okay being opened as a popup"_. This is not something you cannot achieve using only Cross-Page-Window-Proxy-Policy.

TODO: INTERACTION BETWEEN crossOriginIsolated via COOP: Same-Origin and COPWPP.
# Cross-Page-Window-Proxy-Policy explainer

## Motivations
Cross-Origin-Opener-Policy (COOP) and Cross-Origin-Embedder-Policy (COEP) have now been available for a while and with them the gating of certain powerful APIs like SharedArrayBuffer behind the crossOriginIsolated bit. Deployment however has been slow and there are still significant pain points for web developers wishing to use these headers. COEP has been at the core of previous efforts, with features like credentialless. This document focuses on the difficulties raised by COOP in particular.

COOP restricts what pages can and cannot be in a BrowsingContext group, depending on their top-level origin and COOP value. The initial goal was to be able to put pages in different processes without breaking scripting invariants. However for some situations that is not an acceptable trade-off.

Currently, all pages that have legitimate interactions with cross-origin popups need to set COOP: Same-Origin-Allow-Popups. That implies that the main page cannot be crossOriginIsolated, and that the popup must have a COOP: Unsafe-None. This is not ideal, as cross-origin popups are used for important features like OAuth, that do not have an alternative at the time (FedCM could be one but is still under development). On the other hand, popups providing these services would like to also protect themselves against XS leaks using COOP.


## Changing our approach
It's important to get back to the basic invariants of scripting:

* Synchronous access of DOM requires that the documents live in the same process.
* Asynchronous access of a popup's windowProxy property can be out of process, if the browser supports pages isolation.
* Asynchronous access of an iframe's windowProxy property can be out of process, if the browser supports out-of-process iframes (OOPIF).

In a world where all browsers support OOPIF and page isolation, we can simply put same origin documents within their own process, without having COOP in the first place. Realistically, this is not the case for OOPIF, but it is the case for page isolation. 


## Details


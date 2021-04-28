# Explainer - Partitioning Storage, Service Workers, and More

March, 2021


## Authors:



*   [wanderview@chromium.org](mailto:wanderview@chromium.org)
*   [mek@chromium.org](mailto:mek@chromium.org)


## Participate

*   [Issue Tracker](https://github.com/wanderview/quota-storage-partitioning/issues)
*   [Discussion Forum](https://github.com/wanderview/quota-storage-partitioning/discussions)

## Table of Contents

* [Introduction](#introduction)
* [Goals](#goals)
* [Non-Goals](#non-goals)
* [General API Approach](#general-api-approach)
* [Storage APIs](#storage-apis)
* [Communication APIs](#communication-apis)
* [ServiceWorker API](#serviceworker-api)
* [Interaction With Extension Pages](#interaction-with-extension-pages)
* [Key Scenarios](#key-scenarios)
  * [Communication Between Embeds on Different Top Sites](#communication-between-embeds-on-different-top-sites)
  * [Service Worker Controlled Iframes](#service-worker-controlled-iframes)
  * [Composability of the Web](#composability-of-the-web)
  * [Offline Support in 3rd Party Contexts](#offline-support-in-3rd-party-contexts)
* [Considered Alternatives](#considered-alternatives)
  * [Blocking APIs](#blocking-apis)
  * [Making APIs Ephemeral](#making-apis-ephemeral)
  * [First-Party Set Partition Key](#first-party-set-partition-key)
  * [Separate Partitioned API Entrypoint](#separate-partitioned-api-entrypoint)
  * [Escape Hatch to 1st Party Storage Access](#escape-hatch-to-1st-party-storage-access)
* [Privacy & Security Considerations](#privacy--security-considerations)
* [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
* [References & Acknowledgements](#references--acknowledgements)

## Introduction

This document proposes that a number of web platforms APIs used for storage and communication should be partitioned in 3rd party contexts.  In this context "partitioned" means that in addition to being isolated by the [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy), the affected APIs in 3rd party contexts would also be separated by the [site](https://html.spec.whatwg.org/multipage/origin.html#sites) of the top-level context.

The motivation for this partitioning is well defined in the [privacycg partitioning explainer](https://github.com/privacycg/storage-partitioning).  To quote that explainer, there are two problems:

1. "A top-level site `https://site-a.example` A [can] infer that a user is also visiting top-level site `https://site-b.example` B, by embedding resources or documents from B in A. Beyond visiting, it can also allow A to infer specific state from B that depends on the user, thereby revealing many aspects of the user. Timing Attacks on Web Privacy, XS-Leaks, and COSI discuss this in more detail."
2. "Conversely, it allows a site `https://tracker.example` whose resources might be embedded on many different sites, to track the end user across these sites."

Partitioning mitigates these privacy and security issues while largely maintaining web compatibility with currently existing sites.

As discussed in the [privacycg partitioning explainer](https://github.com/privacycg/storage-partitioning) there are many places in the browser where state is maintained.  For this effort we will focus on storage and communications APIs above the network stack.


## Goals

There are a number of high level goals for this effort:



*   Address the privacy and security issues highlighted in the [privacycg partitioning explainer](https://github.com/privacycg/storage-partitioning).
*   Minimize the breakage of existing web sites.  Some breakage is unavoidable since the effort is focused on removing capability leading to the privacy and security issues above.  We have an explicit goal to avoid unnecessarily breaking other features in 3rd party contexts.
*   Maintain support for legitimate use cases and capabilities on the web where possible.  For example, offline support for a site using composable 3rd party components.

As mentioned in the previous section, this effort is focused on storage and communication APIs above the network stack.


## Non-goals

There are a number of related topics that are not addressed in this proposal:



*   Cookies and network stack related state (see the separate [network state explainer](https://github.com/MattMenke2/Explainer---Partition-Network-State) and [cookie explainer](https://github.com/DCtheTall/CHIPS)).
*   Permission state (as exposed for example via navigator.permissions.query) is already specified as being determined in the context of a specific settings object (frame or worker). So while implementations should make sure the state of permissions can't be used as a communication mechanism, most browsers already split out permission state based on the top-level site in some manner.
*   Any "escape hatch" mechanism like requestStorageAccess() to gain access to 1st party state from a 3rd party context.


## General API Approach

In general our proposed approach is to partition APIs by default in 3rd party contexts.  All of these APIs are currently isolated by origin.  This proposal would further include the top-level window's [site](https://html.spec.whatwg.org/multipage/origin.html#sites) (scheme+etld+1) in the isolation or partition key.

For example, consider two browser tabs:



1. An `https://a.com` window.
2. An `https://b.com` window with a nested `https://a.com` iframe.

Previously the browser would allow the nested iframe in tab 2 to access the same IndexedDB as the top-level window in tab 1.  In addition the iframe could use communication APIs such as BroadcastChannel to communicate directly with the top-level window in tab 1.

With partitioning we propose that the APIs will be "double-keyed" with the top-level site of the current tab in addition to the normal origin of the current context.  So instead of being able to access `https://a.com` storage and communication channels it would instead be accessing a logical partition associated with `https://b.com+https://a.com`.  The top-level window in tab 1 would instead be accessing logical partition `https://a.com+https://a.com` instead, and would therefore no longer share storage.

Further, consider two more browser tabs:



1. An `https://a.com` tab.
2. An `https://foo.a.com` tab with a nested `https://a.com` iframe.

In this scenario the second tab has a top-level window with a different origin, but the site (`https://a.com`) is the same as the first tab.  Therefore the iframe in tab 2 and the top-level window in tab 1 would both be accessing the same `https://a.com+https://a.com` logical partition.

Site is used for the partition key for a number of reasons.  It helps reduce website breakage during rollout since many companies share resources across subdomains.  In addition, it is nice to maintain consistency with [http cache partitioning](https://github.com/shivanigithub/http-cache-partitioning) which also chose top-level site in its partitioning key.


## Storage APIs

There are numerous APIs for storing data on disk.  In general these are bound to the origin of the context.  In addition, they are managed by the quota system which ensures that the origin does not use too much space.  The quota system is also responsible for wiping all the storage at the same time if it must be cleared.

Partitioning generally means that data stored by each of these storage APIs will no longer be available to all contexts in the same origin.  Instead the data will only be available to contexts with the same origin and same top-level site.

Partitioning will apply to all non-cookie storage:



*   **Quota**: The quota system must manage each partition as a separate bucket for determining how much space is permitted and when it should be cleared. This also includes APIs that give information on quota to the website, such as **navigator.storage.estimate()**, and Chrome-only deprecated versions of this such as **window.webkitStorageInfo** and **navigator.webkitTemporaryStorage**.
*   **IndexedDB:** A simple quota-managed storage API.
*   **Cache API:** A simple quota-managed storage API.
*   **localStorage** and **sessionStorage**, are not currently quota-managed, but should still be partitioned.
*   The **Origin Private File System**, as exposed via the File System Access API at **navigator.storage.getDirectory()**, the legacy File System API at **window.webkitRequestFileSystem**, and the currently being developed Storage Foundation API.
*   The currently being developed **Storage Buckets API**: both the metadata associated with buckets, as well as the actual data stored in Storage Buckets will be partitioned.
*   Requests to clear “storage” via the **Clear-Site-Data** header should also be limited to only clear storage within one partition. 

Additionally there are a couple of storage APIs we will treat slightly differently:



*   The **blob url store** could be used as a limited communication mechanism as well. Though rather than partitioning this by top-level site, we will partition this by agent cluster with an exception to still allow navigation in a top-level browser context to any blob URL, as discussed in [w3c/FileAPI#153](https://github.com/w3c/FileAPI/issues/153).
*   **AppCache** is in the process of being removed completely, so no further work is planned for it.
*   **WebSQL** is a Chrome-only API, scheduled for eventual deprecation and removal. Rather than investing in partitioning, we propose to complete the removal for this API in at least 3rd party contexts.


## Communication APIs

In addition to partitioning storage APIs we must also partition APIs that allow one context to communicate to another along origin boundaries.  This is necessary to avoid a 3rd party iframe from communicating with a 1st party window to exfiltrate state from the storage partition.

These APIs are:



*   **BroadcastChannel:** This API allows any window, iframe, or worker to publish a message that can be received by any other context within the same origin.
*   **SharedWorker:** This API allows multiple windows and frames within a single origin to connect to the same SharedWorker instance.  The worker postMessage() feature can then be used to funnel messages from one context to another.
*   **WebLocks:** This API creates a mutual exclusion lock that can be used to prevent different contexts within the same origin from operating on the same state at the same time.  This can be used to signal bits of information from one context to another.  While not as high bandwidth as postMessage, we still must partition this API to avoid the information leakage. Since these contexts will no longer be able to operate on the same state, no loss of functionality is anticipated.

Note, the partitioning here is mainly about APIs that allow the discovery of other contexts via broadcasting or same-origin rendezvous.  We are not changing postMessage() APIs where the relationship between contexts is clearly defined and fixed; e.g. cross-site iframe postMessage() is not proposed to be changed.


## ServiceWorker API

The ServiceWorker API is an interesting case.  It is both a storage API and a communication API.  Sites create persistent registrations that can then create a completely new worker context to respond to edge triggered events for different features.  This worker context can then communicate with any same-origin context.

In addition, the ServiceWorker API can change the timing of navigation requests leading to the potential for cross-site information leaks; e.g. [history sniffing](https://www.ndss-symposium.org/wp-content/uploads/ndss2021_1C-2_23104_paper.pdf).

For these reasons it's important to partition the ServiceWorker API in 3rd party contexts.  This will roughly mean:



1. When a context registers a ServiceWorker the partition key will be stored in the registration data.
2. When a ServiceWorker context is created from a registration it must use the same partition key that was stored in (1).
3. APIs called from the ServiceWorker context must use the partition key from (2).
4. [ExtendableEvents](https://w3c.github.io/ServiceWorker/#extendableevent-interface) that are fired at the ServiceWorker and may cause it to "wake up" must have a partition key associated with them and match the key in the registration.
5. The [Clients API](https://w3c.github.io/ServiceWorker/#clients-interface) that allows the ServiceWorker to find and communicate with other contexts must also be partitioned by the partition key.

This means APIs like [background-sync](https://wicg.github.io/background-sync/spec/), [background-fetch](https://wicg.github.io/background-fetch/), etc must also store the partition key in their registration data to fire events for the correct partition.  The [push notification](https://w3c.github.io/push-api/) API is an exception in that most browsers do not allow it to operate in 3rd party contexts already.


## Interaction with Extension Pages

In Chromium, extensions can create [background pages](https://chrome-apps-doc2.appspot.com/extensions/background_pages.html) with manifest v2.  These pages have an extension URL origin, but can embed iframes with normal web content origins.  One reason extensions do this is to update the state of the web page from the background.  The extension permissions make this much more powerful than current APIs available in the web platform.

Partitioning as currently outlined will break this use case.  The iframe will be partitioned by the top-level window's extension URL.  This will cause the iframe to update a partitioned storage area instead of the real first party storage.

Toward our goal of minimizing breakage for existing sites we plan to implement a mitigation.  If the extension has [host\_permissions](https://developer.chrome.com/docs/extensions/mv2/runtime_host_permissions/) for the iframe origin, then the iframe will be treated as the top-level frame and not the extension page.

Ultimately this is a short term issue since chrome has [deprecated](https://blog.google/products/chrome/making-chrome-extensions-more-private-and-secure/) manifest v2 extension support and ultimately plans to remove it.  Manifest v3 is [already available](https://blog.chromium.org/2020/12/manifest-v3-now-available-on-m88-beta.html).  So far we believe manifest v3 does not currently have a similar issue to the one described above for manifest v2.  If a similar issue arises in manifest v3 we plan to address it with similar principles to above; i.e. 1st party access based on permissions.


## Key scenarios


### Communication Between Embeds on Different Top Sites

One of the main scenarios this proposal addresses is a 3rd party that is embedded into multiple different top level pages.  As discussed in the [privacycg partitioning explainer](https://github.com/privacycg/storage-partitioning) this permits the 3rd party to effectively track user activity across the web.


### Service Worker Controlled Iframes

Another important case occurs when an embedded iframe is controlled by a service worker that performs caching for improved performance or offline support.  Since the parent window can see the iframe's load event it's possible to observe timing changes due to the existence of the service worker.  This in turn allows the parent window to infer whether or not the user has ever visited that site.  This is in effect a [history sniffing](https://www.ndss-symposium.org/wp-content/uploads/ndss2021_1C-2_23104_paper.pdf) security issue that is addressed by partitioning.


### Composability of the Web

In addition to scenarios where we must restrict capability to solve problems, at the same time we also want to maintain positive features of the web.  One of the strengths of the web over time has been its composability allowing context to be combined or "re-mixed" in new and innovative ways.  While iframes have their problems, they are a major source of this kind of composability.  We feel it's important to minimize breakage of features in iframes as much as possible to maintain this important aspect of the web ecosystem.


### Offline Support in 3rd Party Contexts

For example, one feature of the web is the ability to build offline experiences using the storage and ServiceWorker APIs.  If we were to block or make storage ephemeral in 3rd party contexts it would prevent offline support.  This would in turn prevent products that use composable iframe plugins from support offline.  Products such as these already exist in the form of productivity software with plugins for adjacent capabilities; e.g. email, calendar, video conferencing, contacts, etc.


## Considered alternatives


### Blocking APIs

One simple alternative to partitioning these APIs would be to simply remove them or make them throw exceptions in 3rd party contexts.  This would be effectively making the browser "disable 3rd-party cookies" setting the default.  Unfortunately this is quite breaking for a large number of sites.  As one of our goals is to minimize unnecessary site breakage we have chosen not to pursue this approach.


### Making APIs Ephemeral

In addition to partitioning storage, storage APIs could also be made ephemeral.  This would result in state changes only being made in memory and never persisted to disk.  The site would effectively start from a blank slate every time the user visited it.

We feel this creates too much friction for users for legitimate use cases.  For example, it would no longer be possible to build an offline-capable site using composable iframes.  Another example is 3rd-party iframes representing components on a site would no longer be able to remember user settings or preferences.


### First-Party Set Partition Key

The current proposal is to use the top-level document site as the partition key.  We also considered using the [First-Party Sets](https://github.com/privacycg/first-party-sets/blob/main/README.md) (FPS) of the top-level document instead.  It has been [suggested](https://github.com/privacycg/first-party-sets/issues/35), however, that FPS should not be used as a security boundary.  This was discussed in the context of http cache, but the same security concerns exist in quota managed storage and service workers.  Therefore we have chosen not to partition by FPS for now.


### Separate Partitioned API Entrypoint

One alternative approach would be to add a new `navigator.partitioned` API that would expose each storage API.  For example, `navigator.partitioned.indexedDB`, etc.  Sites that need storage in 3rd party contexts would have to change to use this new API endpoint.  After some transition period we would then disable the normal storage API endpoints.

This would provide a more managed transition to a partitioned state.  It is closer to what is being [proposed for cookies](go/chips-explainer).  Cookies, however, come with particular web compat issues when partitioning that are not present in these other storage APIs.  In addition, a separate API endpoint comes at the cost of requiring every website to be modified.  The benefit of this approach to these APIs does not seem to outweigh the cost.


### Escape Hatch to 1st Party Storage Access

When partitioning or blocking storage other browsers have sometimes included an "escape hatch" to allow the site to gain access to 1st-party storage again.  For example, the iframe could call [`requestStorageAccess()`](https://privacycg.github.io/storage-access/) to gain access after a heuristic or user consent prompt confirms access is permitted.  So far other browsers have mainly used this mechanism to provide access to 1st party cookies, but it could also be used to provide access to 1st party quota managed storage as well.

There are a couple issues with systems like `requestStorageAccess()`.  First, prompts are difficult for users to understand.  For example, they may not understand the page is composed of code running under origins different from what the URL bar shows.  This makes it hard to craft a prompt that a user can understand and answer correctly.  In addition, prompts can become spammy and an abuse in their own right.  We would like to avoid prompts.

Heuristics are also used, but we would prefer to avoid them as they produce an unpredictable platform for web developers to build on.  Web developers should be able to understand requirements for access up front when designing their application.

Finally, there are technical implementation concerns around switching the underlying storage from partitioned to unpartitioned.  For example, what happens if the switch occurs while there are outstanding IndexedDB transactions or other operations?  What happens to a page controlled by a partitioned service worker?  While perhaps solvable, this definitely adds a great deal of complexity to the solution.

For these reasons we are not currently pursuing an escape hatch mechanism.


## Privacy & Security Considerations

As discussed above this main focus of this proposal is to improve the privacy and security characteristics of 3rd party contexts.  Partitioning prevents a 3rd party from sharing state between different 1st party hosting pages.  In addition, partitioning prevents the parent hosting context from inferring the user's history visiting the 3rd party site directly.


## Stakeholder Feedback / Opposition

Tableau has provided feedback in a [privacycg storage-partitioning issue](https://github.com/privacycg/storage-partitioning/issues/21) that partitioning seems to work well so far for their 3rd party use case; embedded analytics.

AMP currently uses a 3rd party iframe to preload assets, like a [service worker](https://amp.dev/documentation/examples/components/amp-install-serviceworker/), when a page is served from the AMP cache.  This will break with the proposed partitioning.  Preloading across the origin boundary will likely require a separate, targeted API.

We expect additional stakeholder feedback to be shared after publishing the explainer.  It will be added here when available.

## References & acknowledgements

Storage partitioning has had a long history in other browsers.  In particular WebKit and Safari were the first to [double-key their storage](https://bugs.webkit.org/show_bug.cgi?id=110269).  Firefox also has been active in [building out storage partitioning](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Privacy/State_Partitioning).  This proposal builds on the groundwork from these other efforts.

Storage partitioning is also an effort ongoing in the privacycg.  While we reference this important work, we felt that our approach may differ enough to warrant its own separate explainer.  For example, this proposal's intention to avoid requestStorageAccess() seems to deviate from the privacycg effort currently.

External References:

*   [privacycg partitioning explainer](https://github.com/privacycg/storage-partitioning)
*   [CHIPS explainer](https://github.com/DCtheTall/CHIPS)
*   https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin\_policy
*   https://github.com/privacycg/first-party-sets/blob/main/README.md
*   https://github.com/privacycg/first-party-sets/issues/35
*   https://privacycg.github.io/storage-access/
*   https://github.com/shivanigithub/http-cache-partitioning
*   https://www.ndss-symposium.org/wp-content/uploads/ndss2021\_1C-2\_23104\_paper.pdf
*   https://w3c.github.io/ServiceWorker/#extendableevent-interface
*   https://wicg.github.io/background-sync/spec/
*   https://wicg.github.io/background-fetch/
*   https://w3c.github.io/push-api/
*   https://bugs.webkit.org/show\_bug.cgi?id=110269
*   https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Privacy/State\_Partitioning
*   https://github.com/w3c/FileAPI/issues/153
*   https://github.com/privacycg/storage-partitioning/issues/21
*   https://amp.dev/documentation/examples/components/amp-install-serviceworker/

Many thanks for valuable feedback and advice from:

*   Mike Taylor
*   Lily Chen
*   Steven Bingler
*   Kaustubha Govind
*   Brad Lassey
*   Jade Kessler
*   Rowan Merewood

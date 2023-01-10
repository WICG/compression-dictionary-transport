# Compression disctionary transport

## What is this?
This explainer outlines the benefits of compression dictionaries, details the different use case for them, and then proposes a way to deliver such dictionaries to browsers to enable these use cases.

### What are compression dictionaries?
Compression dictionaries are bits of compressible content known ahead of time. They are being used by compression engines to reduce the size of compressed content.

Because they are known ahead of time, the compression engine can refer to the content in the dictionary when representing the compressed content, reducing the size of the compressed payload. The decompression engine can then interpret the content based on that pre-defined knowledge..

Taken to the extreme, if the compressed content is identical to the dictionary, the entire delivered content be a few bytes referring to the dictionary.

Now, you may ask, if dictionaries are so awesome, then..

### Why aren't browsers already using compression dictionaries?

At some point, Chrome did use a compression dictionary.
When Chrome was first released, it supported a dictionary compression method called [SDCH](https://en.wikipedia.org/wiki/SDCH) (Shared-dictionary Compression over HTTP).
That support was [unshipped](https://groups.google.com/a/chromium.org/g/blink-dev/c/nQl0ORHy7sw/m/HNpR96sqAgAJ) in 2016 due to complexities around the protocol’s implementation, specification and lack of an interoperability story.

SDCH enabled Chrome and Chromium-based browsers to create origin-specific dictionaries, that were downloaded once for the origin and enabled multiple pages to be compressed with significantly higher rates.
That's one use case for compression dictionaries we will call the "Shared dictionary" use case.

There's another major use case for shared dictionaries that was never supported by browsers - [delta compression](https://en.wikipedia.org/wiki/Delta_encoding).

That use-case would enable the browser to reuse past resources (e.g. your site's main JS v1.2) in order to compress future ones (e.g. main JS v1.3).
But traditionally, this use-case raised complexities around the abilities of the browser to coordinate its cache state with the server, and agree on what the dictionary would be. It also raised issues with both sides having to store all past versions of each resource in order to successfully be able to compress and decompress it.

The common thread is that the use of compression dictionaries had run into various complexities over the years which resulted in deployment issues.

### This time will be different

A few things about this current proposal are different from past attempts, in ways we're hoping are meaningful:
* CORS-based restrictions can ensure that public and private resources don't get mixed in ways that can leak user data.
  - Dictionaries will be fetched with "omit" CORS mode to ensure they are public resources and don't contain any user secrets
  - Non-shared dictionaries must be CORS fetched, and only apply to same destination
* Path-based scoping would help us manage a "single possible dictionary per request" policy, which will minimize client-side cache fan-out.
* Diff-caching on the server can simplify and enable the server-side deployment story.

## Use cases


### Compression types

As mentioned above, use of compression dictionaries in browsers can be thought of as two very distinct categories:

* Delta compression - reusing past downloaded resources for compressing future updates of the same or similar resources.
* Shared dictionary - a dedicated dictionary is downloaded out-of-band, and then used to compress and decompress resources on the page.

There are pros and cons to either of those approaches:


* Shared dictionaries 
    * Are likely to provide better compression ratios for resources never before seen by the user.
    * They are easy to manage on the server side
        * Only a small set of dictionaries to use for compression purposes.
        * Precompression based on dictionaries users are likely to have is feasible.
    * OTOH, they require out-of-band download which may only be worth the user’s while if they plan to visit this origin again. Predicting the future is hard..
* Delta compression using past resources 
    * Likely to provide better compression ratios for resource updates. 
    * Doesn’t require out-of-band download of the dictionary.
    * OTOH, server-side management can be hard:
        * Servers may need to keep a large number of past versions to account for any user/past-resource combination
        * As such, precompression may not be practical.


### Resource types

Otherwise, we have multiple categories of resources that can benefit from compression dictionaries.


#### Static resources



* Javascript, CSS, WASM, SVG, font (uncompressed) resources
* Typically not personalized, hence publicly cacheable
* Update frequently but in small increments
* Typically fetched using their dedicated [request destination](https://developer.mozilla.org/en-US/docs/Web/API/Request/destination)
* Can be fetched using CORS, with or without credentials


#### Dynamic resources



* Navigation HTML resources
* Often personalized, hence not publicly cacheable
* Update frequently, but in larger increments (although their structure remains the same or similar)
* Can be fetched using both their dedicated request destination or using XHR/fetch(), for SPAs
* Cannot use CORS


### To each its own

Due to the different nature and constraints, it seems better to focus on compressing static resources using delta compression, and dynamic resources using shared dictionaries. That means we need two different solutions, if we want to tackle both use cases.


## Risks


### Security

The Shared Brotli draft does a good job [describing](https://datatracker.ietf.org/doc/html/draft-vandevenne-shared-brotli-format-08#section-10) the security risks. I’ll try to summarize that here:



* [CRIME](https://en.wikipedia.org/wiki/CRIME) and [BREACH](https://en.wikipedia.org/wiki/BREACH) mean that both the resource being compressed and the dictionary itself can be considered readable by the document deploying them. That is Bad™ if any of them contains information that the document cannot already obtain by other means.
* An out-of-band dictionary needs to be carefully examined to ensure that it wasn’t created using users’ private data, nor using content that’s user controlled.


### Privacy

Dictionaries will need to be cached using a triple key (top-level site, nested context site, URL) similar to other cached resources (or any other partitioning scheme that’s good enough for cached resources from a privacy and security perspective). That’s not an issue for the delta compression use case, but can become a burden fast for the out-of-band dictionaries, as multiple nested contexts may need to download the same dictionary multiple times)

**_<span style="text-decoration:underline;">Note:</span>_** [Common payload caching](https://groups.google.com/a/chromium.org/g/blink-dev/c/9xWJK3IgJb4) may be useful in such cases.

There’s also the issue of users advertising resource versions in their cache to servers as part of the request. This already has a precedence in terms of cache validators (ETags, If-Modified-Since), so maybe that’s fine, given that the cache is partitioned.


### Adverse performance effects

Downloading an out-of-band dictionary means that the site owner is making a certain bet regarding the amount of visits that would enable the user to amortize that dictionary’s cost.

At worst, if the user never visits the site again until the dictionary’s lifetime expires, the user has paid the cost of downloading the dictionary with no benefits.

For some large and heavily trafficked sites, that case is rare. For others, it’s extremely common, and we should be wary of both the tools we’d be putting in developers’ hands, as well as the messaging we’re providing them regarding when to use them.


## Proposal

We propose to have two different types of dictionaries, for two different type of flows, outlined below:



* Shared dictionaries for dynamic resource flows. Such dictionaries would be fetched out-of-band, and (hopefully) amortized over multiple resources after that. 
    * The dictionaries themselves would represent common strings that are likely to be present in dynamic content, but won’t necessarily be content themselves.
* Non-shared dictionaries for static resource flows. Such dictionaries would be content themselves (e.g. JS files) that would then be also reusable as dictionaries for future versions of the same (or similar) resources.


### Static resources flow

In this flow, we’re reusing static resources themselves as dictionaries that would be used to compress future updates of themselves, or similar resources.

From an operational standpoint, it’d be hard for servers to keep around many versions of a resource in cache, to be used as compression dictionaries. At the same time, it might be easier to keep a few popular pre-compressed diffs of the current resource version, compressed with a past resource.

Since those diffs should be small, their impact on the server’s cache hit rate should not be huge. Origins that deploy more often may have more versions in the wild that they’d need to keep in cache, but the diffs for them would be smaller than for origins that deploy less frequently, with larger updates.



* [example.com](http://example.com/) downloads [example.com/large-module.wasm](http://example.com/large-module.wasm) for the first time.
* The response for [example.com/large-module.wasm](http://example.com/large-module.wasm) contains a `bikeshed-use-as-dictionary: <path>` response header
* The browser takes note of that, and saves that response to a special dictionary cache for that path and that [request destination](https://fetch.spec.whatwg.org/#concept-request-destination). That cache would be triple-key partitioned.
* The next time the browser fetches a resource from said path and destination, it includes a `sec-bikeshed-available-dictionary:` request header, which lists a **single** SHA-256 hash
    * SHA-256 hashes are long. Their hex representation would be 64 bytes, and we can base64 them to be ~42 (I think). We can't afford to send many hashes for both performance and privacy reasons.
    * We could truncate those hashes, or use shorter ones, but then we'd run into potential correctness issues in case of collisions. We can run the math on that, to see if it's worth the risk.
    * The `sec-` prefix is there to ensure that requests are not attacker-generated.
    * Ideally, we can get developers to pick a single resource to be the best dictionary for a certain path (where that path may only contain future versions of the same resource)
    * Any new resource as a dictionary for that path would override older ones. When sending requests, the browser would use the _most specific path_ for the request to get its dictionary.
* When the server gets a request with the `sec-bikeshed-available-dictionary` header in it:
    * The server can simply ignore the dictionary if it doesn't have a diff that corresponds to said dictionary. In that case the server can serve the response without delta compression.
    * If the server does have a corresponding diff, it can respond with that, indicating that as part of its `content-encoding` header, or some other header. There's no need to repeat the hash value, as there's only one.
      - For example, if we're using [shared brotli compression](https://datatracker.ietf.org/doc/draft-vandevenne-shared-brotli-format/), the `content-encoding: sbr` header can indicate that.
    * In case the browser advertized a dictionary but then fails to successfuly fetch it from its cache *and* the dictionary was used by the server, the resource request should be terminated.


### Dynamic resources flow



* Shared dictionary is downloaded out of band and declared using a dedicated header. That means that a dynamic resource's response would contains a `bikeshed-shared-dictionary-url` header, with the URL for the resource to be downloaded.
    * The dictionary resource will be downloaded with CORS in “omit” mode, ensuring lack of credentials.
    * It will be downloaded with “idle” priority, once the site is actually idle
    * Browsers may decide to not download it when they suspect that the user is paying for bandwidth, or when used by sites that are not likely to amortize the dictionary costs (e.g. sites that the user isn’t visiting frequently enough).
    * Browsers may decide to not use a shared dictionary if it contains hints that its contents are not public (e.g. `Cache-Control: private` headers).
* Once the dictionary is downloaded, future navigation requests may advertise the presence of the dictionary using a `sec-bikeshed-available-dictionary:` request header.
    * Only a single shared dictionary will be advertised. Shared dictionary will only be used if the path in question doesn't have another available dictionary (from the static resource flow). New shared dictionaries will override older ones.
    * Browsers may avoid advertising the header in some cases to reduce abuse risk.
    * Due to the dynamic nature of the resource, dynamically compressing it with the dictionary will not pose a regression compared to common practices today (e.g. serving-time compression with brotli level 5).
    * Similar to the static resource case, the server will indicate a use of the dictionary using the `content-encoding` header.
    * Similarly again, in case the browser advertized a dictionary but then fails to successfuly fetch it from its cache *and* the dictionary was used by the server, the resource request should be terminated.

#### Open questions:
* Should shared dictionaries also have a "path" component, enabling multiple ones for different parts of the origin?


### Compression API

The above can also be exposed to the Compression API.

TODO(yoav): think about what that’d mean and how would that look like.


### Security mitigations

Only credentialless CORS-fetched resources can be used as dictionaries, to ensure they don’t contain private data that the origin cannot read otherwise.

Only same-origin or CORS-fetched resources can have dictionaries apply to them. Unless the dictionary is “shared”, it’d need to match the request destination of the resource to which it is applied. Shared dictionaries will be applicable to any request destination.

We may also need to rely on browser caches being keyed on credentials mode, although if that’s what would make the resource/dictionary readable to the origin, that could already be a concern.


### Other considerations

The implementation would need to be carefully considered, to avoid too many IPC hops between the renderer and network service.
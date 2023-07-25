This is the W3C TAG [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for [Compression Dictionary Transport](README.md)

> 01.  What information does this feature expose,
>      and for what purposes?

This feature exposes the browser's support for dictionary-based content encoding
as well as the hash of a single previously-fetched dictionary that applies to a given
request. Both are for the purpose of compressing the response of the given request
using the previously-downloaded content as a compression dictionary.

> 02.  Do features in your specification expose the minimum amount of information
>      necessary to implement the intended functionality?

Yes.

> 03.  Do the features in your specification expose personal information,
>      personally-identifiable information (PII), or information derived from
>      either?

No.

> 04.  How do the features in your specification deal with sensitive information?

The feature explicitly excludes opaque responses from using compression dictionaries
to ensure that no additional information can be revealed through attacks on the
compression.

> 05.  Do the features in your specification introduce state
>      that persists across browsing sessions?

Yes. The compression dictionaries and URL paths that they apply to is persisted. They are
partitioned with the cache and cookie partitioning and cleared any time that cookies or
cache content is cleared to ensure that it can not be abused as a tracking vector.

> 06.  Do the features in your specification expose information about the
>      underlying platform to origins?

No.

> 07.  Does this specification allow an origin to send data to the underlying
>      platform?

No.

> 08.  Do features in this specification enable access to device sensors?

No.

> 09.  Do features in this specification enable new script execution/loading
>      mechanisms?

No.

> 10.  Do features in this specification allow an origin to access other devices?

No.

> 11.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?

No.

> 12.  What temporary identifiers do the features in this specification create or
>      expose to the web?

None.

> 13.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?

The feature follows the existing browser support for opaque responses and requires
that the dictionaries are only usable when both the dictionary and the content they
apply to are non-opaque to the context they are being loaded in.

> 14.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?

There is no change. The dictionary store uses the ephemeral storage of the browser's
private browsing mode and is cleared when the ephemeral storage is cleared.

> 15.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?

Yes.

> 16.  Do features in your specification enable origins to downgrade default
>      security protections?

No.

> 17.  What happens when a document that uses your feature is kept alive in BFCache
>      (instead of getting destroyed) after navigation, and potentially gets reused
>      on future navigations back to the document?

No impact.

> 18.  What happens when a document that uses your feature gets disconnected?

No impact.

> 19.  What should this questionnaire have asked?

The dictionary and the resources they apply to must also be same-origin to each other
to ensure that origins can not have impact over other origins.
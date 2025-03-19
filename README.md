# Compression dictionary transport

The compression dictionary transport specification has moved to the IETF: [Compression Dictionary Transport](https://datatracker.ietf.org/doc/draft-ietf-httpbis-compression-dictionary/)

# Changelog
These are the changes that have been made to the specs as it has progressed through various standards organizations and based on developer feedback during browser experiments.

## July 2024
* The `dictionary` link relation type was changed to `compression-dictionary`.
* The `br-d` content encoding changed to `dcb` and a header with the hash of the dictionary was added to the stream.
* The `zstd-d` content encoding changed to `dcz` and a header with the hash of the dictionary was added to the stream.
* The `Content-Dictionary` header was eliminated (since the dictionary hash was moved into the `dcb` and `dcz` response stream).

## Feb 2023
* The `Sec-Available-Dictionary` request header changed to `Available-Dictionary`.
* The value of the `Available-Dictionary` request header changed to be a [Structured Field Byte Sequence](https://www.rfc-editor.org/rfc/rfc8941.html#name-byte-sequences) (base-64 encoding of the dictionary hash, surrounded by colons) instead of hex-encoded string.
* The content encoding string for brotli with a dictionary changed from `sbr` to `br-d`.
* The `match` field of the `Use-As-Dictionary` response header is now a URLPattern.
* The expiration of the dictionary now uses the cache expiration of the dictionary resource instead of a separate `expires`.
* The server can provide an `id` in the `Use-As-Dictionary` response header which is echoed in the `Dictionary-ID` request header by the client in future requests.
* The server needs to send a `Content-Dictionary` response header with the hash of the dictionary used when compressing a response with a dictionary (must match the `Available-Dictionary` from the request).
* `match-dest` was added to the `Use-As-Dictionary` response header to allow for matching on fetch destinations (e.g. `match-dest=("document" "frame")` and have the dictionary only be used for document requests).

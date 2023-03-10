# Compression dictionary transport examples
This is a collection of tests that were run using shared dictionaries to gauge their likly effectiveness for [static](https://github.com/yoavweiss/compression-dictionary-transport#static-resources-flow) and [dynamic](https://github.com/yoavweiss/compression-dictionary-transport#dynamic-resources-flow) resource flows.

As of the time of this writing, the brotli command-line utility only uses external compression dictionaries for compression levels of 5 or higher so gains over brotli alone will only be visible at those compression settings. The vast majority of gains are achieved at level 5 which is what is used for the examples below (higher levels tended to show roughly the same gains over regular brotli at the same compression level).

## Static resource flows
In static resource flows, an existing version of a resource can be used as a dictionary for a new version of the resource. This can be useful for application updates, shipping just the differences from what the user currently has. It can be used for updating the exact same resource in-place (exact same URL) like with well-known scripts that expire and update in-place or it can be used with URL variations where the build number or hash of the code is included in the URL.

It works independent of resource type and can work well for JavaScript, CSS, WASM and most text-based resources.

There is a test tool available [here](https://test.patrickmeenan.com/shared-brotli/static/) that, provided an old and new URL for a given resource, will evaluate how well the new version of a resource compresses if the previous version is used as a dictionary and compares it to regular brotli compression. To get URLs for old versions of resources, the [Wayback machine](https://archive.org/web/) works very well for getting previous versions of resources if they have a snapshot of a page that used it (open the page in the wayback machine with dev tools open to get the URL for the snapshot version of the resource you are looking for).

## Dynamic resource flows
In dynamic resource flows, an external dictionary is built specifically for a set of resources, is downloaded out-of-band and used as a dictionary for requests on a specified path. An example of this is building a dictionary for the HTML for product listing pages. The dictionary can extract the "common" parts of the page so that navigating across multiple, different pages only sends down the part of the HTML that is unique for each page. The effectiveness of the external dictionary depends on how much of the content across pages is common and the useful lifetime of the dictionary depends on how frequently that common content changes.

Since the dictionaries are scoped to specific paths, different dictionaries can be built for different parts of a product experience (search pages vs category pages vs detail pages, etc).

Caution needs to be taken when building external dictionaries that they do not include private user data.

The dictionaries themselves can be brotli-compressed for transfer and can use the static resource flow themselves when updating, making them much smaller than the raw as-tested dictionary size.

There is a test tool available [here](https://test.patrickmeenan.com/shared-brotli/) that, provided a list of URLs to test and a target dictionary size:
* Fetches the HTML (or whatever content it is) for all of the URLs in the list. The fetches are done from the server with no cookies and using the provided user agent string.
* For each URL on the list:
    * Builds a dictionary using [dictionary_generator](https://github.com/google/brotli/blob/master/research/dictionary_generator.cc) from all of the other URLs fetched, excluding the URL being tested. This makes sure that the dictionary is being tested for usefulness across pages other than those in the training set.
    * Measures the effectiveness of compressing the URL using the generated dictionary vs compressing the URL using brotli alone.

# Static resource flow results

## Youtube desktop player
This looks at the JavaScript for YouTube's desktop player from January 2023 vs March 2023, simulating a user that visited in January and came back again in March. More frequent usage would result in much smaller incremental changes.

The player JavaScript was [78% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=500dea94805eb1bf0d786e2992f220c96d45d9f9) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 10mb JavaScript was 1.8mb with brotli alone and 384kb when using brotli and the previous version as a dictionary.

## CNN react bundle
This looks at the `cnn-header-second-react.min.js` file on www.cnn.com from March 2022 to March 2023, simulating a user that visited in March 2022 and came back again a year later. More frequent usage should result in much smaller incremental changes.

The JavaScript was [63% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=25260065d6825e4fac333374e273a9987e482b75) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 1.5mb JavaScript was 344kb with brotli alone and 128kb when using brotli and the previous version as a dictionary.

## CNN header bundle
This looks at the `header.<hash>.bundle.js` file on www.cnn.com from March 2022 to March 2023, simulating a user that visited in March 2022 and came back again a year later. More frequent usage should result in much smaller incremental changes.

The JavaScript was [98% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=9fa0c32122bba2f0b118b7b4d3e35a4477ef9e35) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 278kb JavaScript was 90kb with brotli alone and 2kb when using brotli and the previous version as a dictionary.

## Cookielaw banner SDK
This looks at the `otBannerSdk.js` file used by cookielaw (cookie consent) from March 2022 to March 2023, simulating a user that visited a site in March 2022 and came back again a year later. More frequent usage should result in much smaller incremental changes.

The JavaScript was [56% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=234a9d228a8436e81cedfa730fa946260b77ff33) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 392kb JavaScript was 83kb with brotli alone and 36kb when using brotli and the previous version as a dictionary.

## Doubleclick pubads_impl
This looks at the `pubads_impl_<date>.js` file used by DoubleClick (ads) from December 2022 to March 2023, simulating a user that visited a site using doubleclick in December 2022 and came back again 3 months later. More frequent usage should result in much smaller incremental changes.

The JavaScript was [51% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=e68bf9cdc870f66bfc902bc680d5be635a279259) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 402kb JavaScript was 125kb with brotli alone and 61kb when using brotli and the previous version as a dictionary.

## Google tag manager on Android Authority
This looks at the `https://www.googletagmanager.com/gtag/js` file used for Google tag manager on the Android Authority website from January 2023 to March 2023, simulating a user that visited the site in January 2023 and came back again 2 months later. More frequent usage should result in much smaller incremental changes depending on the changes made to the tag settings.

The JavaScript was [68% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=bee58104dd79c0feb07fd6601600a5cf64718838) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 187kb JavaScript was 65kb with brotli alone and 21kb when using brotli and the previous version as a dictionary.

## Vercel app bundle
This looks at the `https://vercel.com/_next/static/chunks/pages/_app-<hash>.js` file on vercel.com from January 2023 to March 2023, simulating a user that visited in January 2023 and came back again 2 months later. More frequent usage should result in much smaller incremental changes.

The JavaScript was [71% smaller](https://test.patrickmeenan.com/shared-brotli/static/result.php?id=82330335427ce99f8794585b1fa87a3adeff5d7b) using the previous version as a dictionary for the new version than if the new version was downloaded with brotli alone. Specifically, the 1.3mb JavaScript was 327kb with brotli alone and 94kb when using brotli and the previous version as a dictionary.


# Dynamic resource flow results

## Google search results
10 different search terms were tested that generate wildly different types of search results pages (commerce, text-only, maps, news, IMDB listings, etc). They were tested with varying dictionary sizes including [64k](https://test.patrickmeenan.com/shared-brotli/result.php?id=d2ce67ffe1a0e20ad23eb239935e1752f81c9c0e), [128k](https://test.patrickmeenan.com/shared-brotli/result.php?id=376832e20f4a53e5602a85eed46988a62563535d), [256k](https://test.patrickmeenan.com/shared-brotli/result.php?id=0a052e2e730b553c97e7f9e58980005f82a6b647), [512k](https://test.patrickmeenan.com/shared-brotli/result.php?id=4b800b32a384947c4ffda5d3bd411aa669e71017) and [1mb](https://test.patrickmeenan.com/shared-brotli/result.php?id=0e27d769b7ebae74a311749d35b719c152a3653a).

A dictionary between 128k and 256k (40k and 71k compressed) showed the bulk of the gains. With the HTML foar each search results page being anywhere from [30% to 66% smaller](https://test.patrickmeenan.com/shared-brotli/result.php?id=0a052e2e730b553c97e7f9e58980005f82a6b647) using a shared dictionary (depending on the page content). Pages with more embedded images were on the lower end of the gains and text-only search results pages were commonly 50-65% smaller.

## Etsy product listing pages
10 different product listing pages from Etsy were used with a 1mb dictionary (132kb compressed). Most of the gains could probably have been had with a much smaller dictionary but this was useful for exploring the upper bound in savings.

Using a custom dictionary resulted in pages that were [84-90% smaller](https://test.patrickmeenan.com/shared-brotli/result.php?id=4a1e74fc1a8624f3757b1fbdac684ff7bc983369) to transfer than using brotli alone (98% smaller than the uncompressed size). For example, a 539kb uncompressed HTML page was 84kb using brotli and 10kb using brotli with a custom dictionary.

## Amazon product listing pages
10 different product listing pages from Amazon were used with a 1mb dictionary (235kb compressed). Most of the gains could probably have been had with a much smaller dictionary but this was useful for exploring the upper bound in savings.

Using a custom dictionary resulted in pages that were [60-70% smaller](https://test.patrickmeenan.com/shared-brotli/result.php?id=9b1668a1de4280d1488d44fc5c05abc214ea00c0) to transfer than using brotli alone (95% smaller than the uncompressed size). For example, a 2.7mb uncompressed HTML page was 377kb using brotli and 148kb using brotli with a custom dictionary.

## Ebay product listing pages
10 different product listing pages from Ebay were used with a 512kb dictionary (132kb compressed).

Using a custom dictionary resulted in pages that were [67-75% smaller](https://test.patrickmeenan.com/shared-brotli/result.php?id=c3000616d72124d0cbae76223fe792301ae80b4b) to transfer than using brotli alone (96% smaller than the uncompressed size). For example, a 606kb uncompressed HTML page was 81kb using brotli and 21kb using brotli with a custom dictionary.
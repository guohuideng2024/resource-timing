# Expose resource dependency in Resource Timing

## Contacts

<guohuideng@microsoft.com>

## References & acknowledgements

yashjoshimail@gmail.com put up [4 CLs](https://chromium-review.googlesource.com/q/owner:yashjoshimail@gmail.com)(wpt tests, and implementation for the cases where the initiators are html and javascript) and a [design doc](https://docs.google.com/document/d/1ODMUQP9ua-0plxe0XhDds6aPCe_paZS6Cz1h1wdYiKU/edit?tab=t.0) . More discussion can be found [here](https://github.com/w3c/resource-timing/issues/263) and [here](https://github.com/w3c/resource-timing/issues/380).

## Goal
To expose the dependency of the resources from RUM(real user monitoring) data.
As `nicjansma@` points out [here](https://github.com/w3c/resource-timing/issues/263), the data exposed by this proposal is a RUM version of [RequestMap tool](http://requestmap.webperf.tools/). The RequestMap tool can be a good demonstration of the data to be exposed.
Chrome devtool exposes an attribute [`initiator`](https://developer.chrome.com/docs/devtools/network) for a similar purpose.
    
## User Research

The idea was brought up in year [2021](https://github.com/w3c/resource-timing/issues/263). To sum up the discussion, quoting from `nicjansma@`, a more accurate dependency tree from RUM can help:
-	A web site to optimize load speed
-	The CDN to optimize delivering content
-	Security products to backtrace rogue requests


## Yosh's approach

The following two new fields are added to `PerformanceResourceTiming`:
-	`resourceId`: an unsigned integer that’s unique in a session. It identifies the current fetched resource.
-	`Initiator`: the `resourceId` of the resource that triggered the fetch of current resource.

Suppose we have two PRT(PerformanceResourceTiming) entries: `prt1` and `prt2`, and `prt1.resourceId == prt2.initiator`. We conclude that the resource described by `prt1` triggered the fetch of resource described by `prt2`.

```javascript
const entry_list = performance.getEntriesByType("resource");
for(const entry of entry_list) {
    console.log(entry.name+": resourceId="+ 
       entry.resourceId+", initiator="+entry.initiator) ;
}

/* We could get:
url_to_main_page: resourceId=1, initiator=0
url_to_an_img_included_in_main_page_markup: resourceId=13, initiator=1 

Then we conclude that "main_page" triggered the fetch of "an_img_included_in_main_page_markup".
*/
```

### Generating `resourceId`
We need to generate a unique `resourceId` for every resource we fetch. The initial `resourceId` could be randomly generated from a range, for example, [1, 10]. Then how to generate a next `resourceId`?

1. Yash's current implementation: generate a random number `increase` between [1,10] at the beginning of session. Then, a next `resourceId` is the sum of the previous `resourceId` and `increase`.
2. According to the discussion in the past, the method above was following how chromium generates [`current_interaction_event_id_for_event_timing`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/timing/responsiveness_metrics.cc;l=681;drc=763100e0bf9a25ba6f203612af5a4331fbd2d048). However, the mothod above is different from it. It looks to me that we can do the same with `current_interaction_event_id_for_event_timing`: the `increase` value is a fixed number picked up by the user agent and it is not generated randomly at the beginning of a session.

## A hopeful alternative: expose the initiator's url directly in a field `initiator`

```javascript
const entry_list = performance.getEntriesByType("resource");
for(const entry of entry_list) {
    console.log(entry.name+": initiator="+ entry.initiator) ;
}

/* We could get:
url_to_main_page: initiator="OTHER"
url_to_an_img_included_in_main_page_markup: initiator=url_to_main_page

Then we see that "main_page" triggered the fetch of "an_img_included_in_main_page_markup".
*/
```


## Stakeholder Feedback/Opposition
TBD.

## Security/Privacy Considerations
All the attributes proposed to be exposed are behind `CORS` check. `Time-Allow-Origin` doesn’t expose these values.

### [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

>1.	What information does this feature expose, and for what purposes?

The new fields from `PerformanceResourceTiming` expose what resource triggered the fetch of what resource. Collected from RUM(real user monitoring), they can help the web sites and CDN optimize content delivery, and help security products track down rogue content.

>2.	Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes
>3.	Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No.
>4.	How do the features in your specification deal with sensitive information?

It does not deal with sensitive information.
>5.	Does data exposed by your specification carry related but distinct information that may not be obvious to users??

No.
>6.	Do the features in your specification introduce state that persists across browsing sessions?

No.
>7.	Do the features in your specification expose information about the underlying platform to origins?

No.
>8.	Does this specification allow an origin to send data to the underlying platform?

No.
>9.	Do features in this specification enable access to device sensors?

No.
>10.	Do features in this specification enable new script execution/loading mechanisms?

No.
>11.	Do features in this specification allow an origin to access other devices?

No.
>12.	Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.
>13.	What temporary identifiers do the features in this specification create or expose to the web?

`resouceId` that identifies the resources, which are already been exposed by PerformanceResoruceTiming.

>14.	How does this specification distinguish between behavior in first-party and third-party contexts?

No distinction.
>15.	How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

No difference.
>16.	Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.
>17.	Do features in your specification enable origins to downgrade default security protections?

No.
>18.	What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

The `resourceId` would be larger if the page has been in BFCache.

>19.	What happens when a document that uses your feature gets disconnected?

No difference.
>20.	Does your spec define when and how new kinds of errors should be raised?

No new errors should be raised.
>21.	Does your feature allow sites to learn about the users use of assistive technology?

Not directly.
>22.	What should this questionnaire have asked?

Nothing else.
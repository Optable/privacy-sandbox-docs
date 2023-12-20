# Optable Protected Audience integration with SSPs

Optable supports exclusively on-device bidding activities and doesn't rely on [sequential auctions](https://developers.google.com/privacy-sandbox/relevance/protected-audience-api/sequential-auction-setup)

## Minimum Requirements for SSPs
- [Enroll](https://developer.chrome.com/blog/announce-enrollment-privacy-sandbox/) for the Protected Audience API.
- Update its server/tag to generate auctionConfigs with https://ads.optable.co in `interestGroupBuyers[]`.
- Implement the requirements as an SSP to [runAdAuction()](https://developers.google.com/privacy-sandbox/relevance/protected-audience-api/ad-auction) directly, or through a multi-seller setup (eg: Prebid.js with [fledgeForGPT module](https://docs.prebid.org/dev-docs/modules/fledgeForGpt.html)).
- Pass the [expected auction signals](#expected-auction-signals) by our bidding-logic.

## Provided Ad Metadata for scoreAd
Optable bid with currency https://wicg.github.io/turtledove/#bid-with-currency USD in CPM.

Optable intent to support at the minimum the latest stable version of the IAB Tech Lab driven bidder guidelines.
As it is currently being specified, it is best described by this [specification](https://docs.google.com/document/d/1LOfkk2asw1S6NZs0hBAzmU1V8t8GXu_2hfAkSVvn9AM/edit#heading=h.4pamn58w7gl)

The following schema represents the fields that will always be set in Optable ad metadata:

```protobuf
message AdMetadata {
  // The currency of the bid as ISO-4217 alpha codes, always 'USD' when coming from Optable.
  string cur = 1;

  // The unique identifier of the buyer account in Optable platform.
  string seat = 2;

  // The advertiser domains for block list checking, eg: [optable.co]
  // Optable will always provide at least one.
  repeated string adomain = 3;

  // The unique identifier of the campaign in Optable platform.
  string cid = 4;

  // The unique identifier of the creative in Optable platform.
  string crid = 5;

  // The taxonomy major version in use in the "cat" attribute.
  // Optable currently always set this to 3.
  // See https://github.com/InteractiveAdvertisingBureau/AdCOM/blob/master/AdCOM%20v1.0%20FINAL.md#list_categorytaxonomies
  int32 cattax = 6;

  // The content categories of the creative.
  repeated string cat = 7;

  // The list of attributes ids describing the creative
  // See https://github.com/InteractiveAdvertisingBureau/AdCOM/blob/master/AdCOM%20v1.0%20FINAL.md#list--creative-attributes-
  repeated int32 attr = 8;

  // The language of the creative using ISO-639-1-alpha-2. "xx" will be used to indicate non-linguistic content
  string language = 9;

  // The width of the creative in device independent pixels (DIPS)
  int32 w = 10;

  // The height of the creative in device independent pixels (DIPS)
  int32 h = 11;
}
```

### Event reporting
Optable's creatives will always opt-in automatic beacons on "reserved.top_navigation" (then "reserved.top_navigation_start" once widely available) and share them with `direct-seller`.
A `view` event is also shared with `direct-seller` when the ad's document have fully been executed in the fenced-frame.

### Creative Audits
Creatives are immutable in Optable and should be identified by their renderURL. Any change on a creative result in a new renderURL.

In addition to Optable's declared ad metadata signals, SSP auditing can be achieved by submitting creatives to SSP's APIs upfront.
SSPs can maintain a mapping table `renderURL -> audit state` and expose this to their scoreAd through trustedScoringSignals in order to validate at score time that a creative is approved for winning.

Those audit APIs should expose the audit state to Optable so that Optable doesn't bid with non-approved creatives.

## Expected Auction Signals
Optable intent to support at the minimum the latest stable version of the IAB Tech Lab driven bidder guidelines.
As suggested in this [specification](https://docs.google.com/document/d/1LOfkk2asw1S6NZs0hBAzmU1V8t8GXu_2hfAkSVvn9AM/edit) it is possible to communicate contextual data to Optable using auctionConfig's `auctionSignals` by populating it with a [OpenRTB 2.5](https://www.iab.com/wp-content/uploads/2016/03/OpenRTB-API-Specification-Version-2-5-FINAL.pdf) bid request object containing at the minimum a single `imp[]` with a `banner` object.

It is only required, when creative size is not communicated through `auctionConfig.requestedSize` as documented below.

In order to expose segmented inventory in Optable, the bid request's `site` property can be provided including its `publisher` object as documented below.

### Exposing segmented inventory
Initial integrations with SSPs are exposing the whole inventory as a single seller in Optable platform.
At the minimum the SSP "auctionConfig.seller" seller identifier should be shared with Optable.

Exposing segmented (eg: publisher level) SSP inventory can be done on a per-SSP basis by sharing mapping tables either manually or programatically of publisher ID -> Name.
The publisher ID will be matched against `auctionSignals.site.publisher.id` field when present, in case it isn't present or doesn't match the inventory, the bid request will be assumed and reported to be on the "catchall" inventory for that seller.

### Creative size
Optable will consider both the auctionConfig's `requestedSize` and `auctionSignals.imp[0].banner` restrictions when provided.
At least one of those should be specified for Optable to bid at all.

---
layout: post
title: "Integrating Vertex AI Search for commerce Part 1: Importing GA4 Data"
description: "Learn how to import historical data into Google Cloud's Vertex AI Search for Commerce. This comprehensive guide covers importing product catalogs using API and user events from Google Analytics(GA4) via BigQuery, including solutions for common import challenges."
date: 2024-11-16 16:00:00 +0800
categories: code
image: /assets/images/2024-11-16-vertex-ai-search-retail-1-importing-data/cover.png
tags: google-cloud vertex-ai retail-search product-recommendation bigquery ga4 e-commerce data-import machine-learning cloud-api
---

![Google Cloud Vertex AI Search for Commerce](/assets/images/2024-11-16-vertex-ai-search-retail-1-importing-data/cover.png)

## Background

In one of our e-commerce projects, a very useful feature we always wanted was personalized item recommendations for users. Since we don't have a dedicated data scientist, we don't have the resources to home-bake our own model, and were looking for a suitable managed cloud service for this.

We looked into [Amazon Personalize](https://aws.amazon.com/personalize/), which seems easy and promising, but unfortunately, we didn't have time to set up a new data pipeline just for it, and couldn't even try setting up models.

Recently, I came across Google Cloud's [Vertex AI Search for Commerce](https://cloud.google.com/solutions/retail-product-discovery), which seems to have seamless integration with [Google Merchant Center](https://www.google.com/retail/) and [Google Analytics](https://marketingplatform.google.com/about/analytics/)(GA4). This lowers integration costs, so I decided to give it a try. Turns out, it is not that easy.

## Importing Historical Data

To train a recommendation model in Vertex AI Search, we need data, including products and user events. User events must include proper IDs for products so that the model can learn the relationship between products. Events that contain invalid product IDs are called unjoined events, and will be ignored by the model.

## Importing Product Catalog

There are a few ways to import product catalogs into Vertex AI Search. Here, we will cover two of them. Note that there are [different limitations](https://cloud.google.com/retail/docs/upload-catalog#import-bp) for each import method.

### Importing Product Catalog from Google Merchant Center

If you already have Google Merchant Center set up either for shopping ads or Google Ads, you can easily [import the product catalog from there](https://cloud.google.com/retail/docs/upload-catalog#mc). This is the easiest way to import a product catalog, especially when you have [product structured data](https://developers.google.com/search/docs/appearance/structured-data/product) already set up in your e-commerce site. Google Merchant Center will fetch all the products automatically from your website without any additional import procedure.

Sadly, the last time I used Google Merchant Center, all my products were disapproved since they were considered as [unsupported shopping content](https://support.google.com/merchants/answer/6150006). After a while, they were completely removed from the product list even when I don't want shopping ad, and wouldn't reappear. So, I can't use this method.

### Importing Product Catalog via API

As a developer, [importing via API](https://cloud.google.com/retail/docs/upload-catalog#inline) is the most flexible way to import a product catalog. The product schema is defined as follows:

```javascript
    const { data } = await axios.post('https://retail.googleapis.com/v2/projects/${your-project-number}/locations/global/catalogs/default_catalog/branches/0/products:import', {
      "inputConfig": {
        "productInlineSource": {
          "products": [
            ${your products}
          ],
        }
      }
    }, {
      headers: {
        'Authorization': `Bearer $(gcloud auth print-access-token)`,
      },
    });
```

To get your project number, use the following command:

```bash
gcloud projects list \
--filter="$(gcloud config get-value project)" \
--format="value(PROJECT_NUMBER)"
```

However, there is one extra thing to add to make the API work. If you encounter the following error:

```json
{
  "error": {
    "code": 403,
    "message": "Your application is authenticating by using local Application Default Credentials. The retail.googleapis.com API requires a quota project, which is not set by default. To learn how to set your quota project, see https://cloud.google.com/docs/authentication/adc-troubleshooting/user-creds .",
    "status": "PERMISSION_DENIED"
  }
}
```

Then you need to add the following header to your request:

```javascript
  headers: {
    'x-goog-user-project': ${your-project-id},
    ...
  }
```

## Importing User Events

Like the product catalog, there are a few ways to [import user events](https://cloud.google.com/retail/docs/import-user-events) into Vertex AI Search. We will only cover Google Analytics(GA4) data import here since it requires the least effort for sites already set up with GA4.

### Importing GA4 Data from BigQuery

Before we can import Google Analytics 4(GA4) events into Vertex AI Search, we need to have the data in BigQuery. Follow [the guide](https://cloud.google.com/retail/docs/import-user-events#bq-ga4) to set up BigQuery export to GA4. Normally, GA4 events are exported daily to a dataset named `analytics_123456789.events_20241116` where `123456789` is your GA4 property ID and `20241116` is the date partition of the export.

Once the GA4 data is in BigQuery, we can import the table using the Vertex AI Search console. However, the web UI console can only import one table at a time. Since GA4 exports are partitioned by date, it would be tedious to import them one by one.

One simple solution is to merge all the tables into one table and import them. However, this is only feasible if the size of historical data is small. The sample SQL is as follows:

```sql
CREATE TABLE `analytics_123456789.combined_events` AS
SELECT * FROM `analytics_123456789.event_*` WHERE _PARTITIONTIME BETWEEN '2023-03-01' AND '2023-03-31'

```

### GA4 Event Mapping

Many user events required in Vertex AI have a [direct mapping](https://cloud.google.com/retail/docs/user-events#ga4-mapping) to GA4 events, especially e-commerce events.

Search-related events are more tricky, but as long as `view_list` is set up properly with `search_term` param set, it should be fine. Another way is to use `view_search_results`, which is an automated event if you have enabled GA4's [enhanced measurement](https://support.google.com/analytics/answer/9216061). However, this requires the search term to be in the URL query string with predefined keys.

### What about `home-page-view`?

There is no GA4 event that can directly map to the retail user event `home-page-view`. During import, `page_view` with a path of / is automatically used as a substitute. This is not ideal if your homepage is not at `/`. For example, if your homepage has multiple locales, the home page might have paths like `/en` and `/zh`.

To correctly import these events, we would have to query for these events:

```sql
CREATE TABLE `analytics_123456789.ga_homepage` (
  eventType STRING,
  visitorId STRING,
  userId STRING,
  eventTime STRING
);

INSERT INTO `analytics_123456789.ga_homepage` (eventType, visitorId, userId, eventTime)
SELECT
  "home-page-view" as eventType,
  user_pseudo_id as visitorId,
  user_id as userId,
  CAST(FORMAT_TIMESTAMP("%Y-%m-%dT%H:%M:%SZ",timestamp_seconds(CAST ((event_timestamp/1000000) as int64))) as STRING) AS eventTime
FROM `analytics_123456789.combined_events` where event_name = 'page_view' AND `event_params`[SAFE_OFFSET(0)].`key` = 'page_path' and (`event_params`[SAFE_OFFSET(0)].`value`.`string_value` = '/zh-Hant' or `event_params`[SAFE_OFFSET(0)].`value`.`string_value` = '/en')
```

## Import ordering of event and product catalog matters!

Note that if you import events before the product catalog, events will still be unjoined even if you filled in the correct product IDs in product catalog import. This is because product IDs are joined when ingesting user events, not vice versa.

In this case, you would have to trigger a [user event rejoin job](https://cloud.google.com/retail/docs/manage-user-events#rejoin-event) for the historical events to be joined according to the new catalog.

```bash

curl -X POST \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    -H "Content-Type: application/json; charset=utf-8" \
    --data "{
     'userEventRejoinScope': 'UNJOINED_EVENTS'
     }" \
    "https://retail.googleapis.com/v2/projects/${your-project-number}/locations/global/catalogs/default_catalog/userEvents:rejoin"

```

## Testing out the model

Once the data is imported, we can start training the model. [There are different models](https://cloud.google.com/retail/docs/models#model-types), each suited for different use cases, e.g. "Recommended for you", "Others you may like", "Frequently bought together", etc. Each has a minimum requirement for the data. The console will not allow training of the model if the data requirement is not met.

However, even if the requirements are met, the training model can still fail with an `INSUFFICIENT_TRAINING_DATA` error. The message isn't very helpful, but it likely relates to the quality of the training data. For instance, poor data quality or a high unjoined event rate could be the issue.

### Trying the Similar Product model

Luckily, the "similar product" model [only requires the product catalog](https://cloud.google.com/retail/docs/create-models#import-reqs) to be imported. To start training the model, go to the Model tab of the Vertex AI Search console and create a "similar product" model. The training will take a while.

After the model is finished, we need to create a serving config to use this model. Create one in the "Serving Configs" tab. There are some configurations in serving config that we can tweak, but the default config should be good enough for testing.

After the serving config is created, we can go to the "Evaluate" tab to test the model. Select the serving config we just created and pick a product ID as input. The model should return a list of other similar products.

## Conclusion

Since my historical GA4 events do not meet the model requirement, I could not try the recommendation models that I am interested in. To improve the data quality, I will be implementing methods to [collect real-time user data](https://cloud.google.com/retail/docs/record-events). In the [next post]({% post_url 2025-02-02-vertex-ai-search-retail-2-collecting-real-time-events %}), we will cover how to collect real-time user data.

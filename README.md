# example_company event dictionary

## Overview

Before you can send your own event and context types into Snowplow (using the track unstructured events and custom contexts features of Snowplow), you need to:

1. Define a JSON schema for each of the events and context types
2. Upload those schemas to your Iglu schema repository
3. Define a corresponding jsonpath file, and make sure this is uploaded your jsonpaths directory in Amazon S3
4. Create a corresponding Redshfit table definition, and create this table in your Redshift cluster

Once you have completed the above, you can send in data that conforms to the schemas as custom unstructured events or custom contexts.

## 1. Creating the schemas

Useful resources

* Example jsonschemas can be found in [Iglu Central](https://github.com/snowplow/iglu-central/tree/master/schemas). Note how schemas are namespaced in different folders
* [Snowplow 0.9.5 release blog post](http://snowplowanalytics.com/blog/2014/07/09/snowplow-0.9.5-released-with-json-validation-shredding/), which gives an overview of the way that Snowplow uses jsonschemas to process, validate and shred unstructured event and custom context JSONs
* It can be useful to test jsonschemas using online validators e.g. [this one]() and [this one]()
* [json-schema.org](http://json-schema.org/) contains links to the actual jsonschema specification, examples and guide for schema authors
* The original specification for self-describing JSONs, produced by the Snowplow team, can be found [here](http://snowplowanalytics.com/blog/2014/05/15/introducing-self-describing-jsons/) 


## 2. Uploading the schemas to Iglu

Useful resources

* [Iglu schema repository 0.1.0 release blog post](http://snowplowanalytics.com/blog/2014/07/01/iglu-schema-repository-released/)
* [Iglu central](https://github.com/snowplow/iglu-central) - centralized repository for all the schemas hosted by the Snowplow team
* [Iglu](https://github.com/snowplow/iglu) - respository with both Iglu server and client libraries

Note that all users of the Snowplow Managed Service will have their Iglu servers setup and managed by the Snowplow team


## 3. Defining jsonpath files

Useful resources

* Example jsonpath files can be found on the [Snowplow repo](https://github.com/snowplow/snowplow/tree/master/4-storage/redshift-storage/jsonpaths). Note that the corresponding jsonschema definitions are stored in [Iglu central](https://github.com/snowplow/iglu-central/tree/master/schemas)
* Amazon documentation on jsonpath files can be found [here](http://docs.aws.amazon.com/redshift/latest/dg/copy-usage_notes-copy-from-json.html)

## 4. Creating corresponding Redshift table definitions

* Example Redshift table definitions can be found on the [Snowplow repo](https://github.com/snowplow/snowplow/tree/master/4-storage/redshift-storage/sql). Note that corresponding jsonschema definitions are stored in [Iglu central](https://github.com/snowplow/iglu-central/tree/master/schemas)
* Amazon documentation on Redshift create table statements can be found [here](http://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_TABLE_NEW.html). A list of Redshift data types can be found [here](http://docs.aws.amazon.com/redshift/latest/dg/c_Supported_data_types.html)

## 5. Sending data into Snowplow using the schema reference as custom unstructured events or contexts

Once you have gone through the above process, you can start sending data that conforms to the schema(s) you've created into Snowplow as unstructured events and custom contexts.

In both cases (custom unstructured events and contexts), the data is sent in as a JSON with two fields, a schema field with a reference to the location of the schema in Iglu, and a data field, with the actual data being sent, e.g.

```json
{
    "schema": "iglu: com.acme_company/viewed_product/jsonschema/2-0-0",
    "data": {
        "productId": "ASO01043",
        "category": "Dresses",
        "brand": "ACME",
        "price": 49.95,
        "sizes": [
            "xs",
            "s",
            "l",
            "xl",
            "xxl"
        ],
        "availableSince": "2013-03-07"
    }
}
```

For more detail, please see the technical documentation for the specific tracker you're implementing.

## 6. Managing schema migrations

When you use Snowplow, the schema for each event and context lives with the data. That means you have the flexibility to evolve your schema definition over time.

If you want to change your schema over time, you will need to:

1. Create a new jsonschema file. Depending on how different this is to your current version, you will need to give it the appropriate version number. The [SchemaVer](http://snowplowanalytics.com/blog/2014/05/13/introducing-schemaver-for-semantic-versioning-of-schemas/) specification we use when versioning data schemas can be found [here](http://snowplowanalytics.com/blog/2014/05/13/introducing-schemaver-for-semantic-versioning-of-schemas/)
2. Update the corresponding jsonpath files. If you've created a new major schema version, you'll need to create a new jsonpath file e.g. `exmaple_event_2.json`, that exists alongside your existing `example_event_1.json`
3. For minor schema updates, you should be able to update your existing Redshift table definition e.g. to add add additional columns. For major schema updates, you'll need to create a new Redshift table definition e.g. `com_mycompany_exmaple_event_2.sql`
4. Start sending data into Snowplow using the new schema version (i.e. update the Iglu reference to point at the new version e.g. `2-0-0` or `1-0-1` rather than `1-0-0`). Note that you will continue to be able to send in data that conforms to the old schema at the same time. In the event that you have an event with two different major schema definitions, each event version will be loaded into a different Redshift table
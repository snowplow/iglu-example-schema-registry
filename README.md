# example-schema-registry

[![License][license-image]][license]
[![Discourse posts][discourse-image]][discourse]

## Overview

Before you can send your own self-describing events and context entities into Snowplow, you need to:

1. Define a JSON schema for each of the events and entities.
2. Upload those schemas to your Iglu schema registry
3. Define a corresponding jsonpath file, and make sure this is uploaded your jsonpaths directory in Amazon S3
4. Create a corresponding Redshfit table definition, and create this table in your Redshift cluster

Once you have completed the above, you can send in data that conforms to the schemas as custom self-describing events or custom context entities.

## Prerequisites

We recommend setting up the following two tools before staring:

1. Git so you can easily clone the repo and make updates to it.
2. [Igluctl][igluctl]. This is a command line tool for validating schemas, auto-generating associated SQL table definition and jsonpath files and publishing them to snowplow-mini or S3

## 1. Creating the schemas

In order to start sending a new event or context type into Snowplow, you first need to define a new schema for that event/entity.

1. Create a file in the repo for the new schema e.g. `/schemas/com.mycompany/new_event_or_context_name/jsonschema/1-0-0`
2. Create the schema in that file. Follow the `/schemas/com.example_company/example_event/jsonschema/1-0-0` example
3. Save the schema file

Note that if you have JSON data already and you want to create a corresponding schema, you can do so using the [Schema Guru CLI][schema-guru-github] tool.

Once you have your schema, make sure to validate it using [Igluctl][igluctl]:

```bash
/path/to/igluctl lint /path/to/schemas/com.mycompany/my_new_event_or_context
```

For Windows:

```bash
java -jar /path/to/igluctl lint /path/to/schemas/com.mycompany/my_new_event_or_context
```

Igluctl will fail validating:

- JSON Schema that has inconsistent self-describing information and path on filesystem
- JSON Schema that has invalid `$schema` keyword. It should be always set to iglu-specific, while users tend to set it to Draft v4 or even to self-referencing Iglu URI
- JSON Schema that is invalid against its standard (empty required, string maximum and similar)
- JSON Schema that contains properties which contradict each other, like `{"type": "integer", "maxLength": 0}` or `{"maximum": 0, "minimum": 10}`. These schemas are inherently useless as for some valiators there is no JSON instance they can validate

You can find more information about the `igluctl lint` command in the [corresponding documentation][igluctl-lint].

## 2. Uploading the schemas to Iglu

Once you have created or updated your schemas, you need to push them to Iglu so that they're accessible to the Snowplow data pipeline.

This is a two part process:

1. [Upload the schemas to Snowplow Mini](#21-upload-the-schemas-to-snowplow-mini) (for testing)
2. [Upload the schemas to Iglu for the full pipeline](#22-upload-the-schemas-to-iglu-for-the-full-pipeline)

### 2.1 Upload the schemas to snowplow-mini

In order to upload schemas you need to have Snowplow Mini IP and Iglu Registry Master key.
Here we are refer to them as environment variables (those are just examples, you'll have unique IP and key):

```bash
SNOWPLOW_MINI_IP=127.0.0.1
IGLU_REGISTRY_MASTER_KEY=fd08697f-435c-4916-9c85-d0e50bbb8913
```

Or for Windows:

```bash
SET SNOWPLOW_MINI_IP=127.0.0.1
SET IGLU_REGISTRY_MASTER_KEY=fd08697f-435c-4916-9c85-d0e50bbb8913
```

Run the following command to publish all schemas to the Iglu server bundled with Snowplow Mini:

```bash
/path/to/igluctl static push ./schemas $SNOWPLOW_MINI_IP/iglu-server/ $IGLU_REGISTRY_MASTER_KEY --public
```

Note that you can specify individual schemas if you prefer e.g.

```bash
/path/to/igluctl static push ./schemas/com.mycompany/my_new_event_schema $SNOWPLOW_MINI_IP/iglu-server/ $IGLU_REGISTRY_MASTER_KEY --public
```

Also note that if you're editing existing schemas, the server applications will need to be rebooted to clear the schema cache. This can be done directly from the server using the Control Plane tab that can be found in the UI.

### 2.2 Upload the schemas to Iglu for the full pipeline

Once you've created your schemas, you need to upload them to Iglu.

This can also be done via Igluctl. In the project root, first commit the schema to Git:

```bash
git add .
git commit -m "Committed finalized schema"
git push
```

Then, to publish your schemas stored locally to a remote Iglu Server:

```bash
/path/to/igluctl static push /path/to/static/registry $HOST $APIKEY
```

Alternatively, to push your schemas to an S3 bucket that serves as remote Iglu registry:

```bash
/path/to/igluctl static s3cp ./schemas snowplow-com-mycompany-iglu-schemas-bucket --accessKeyId ABCDEF --secretAccessKey GHIJKILM/12345XYZ --region us-east-1
```

Note that you also can pass credentials via configuration file or environment variables, as with any [AWS tool][aws-credentials].

Useful resources:

- [Iglu schema repository 0.1.0 release blog post][schema-repo-blog]
- [Iglu][iglu]: umbrella repository for the Iglu ecosystem
- [Igluctl `static push`][igluctl-push]: documentation of the `igluctl static push` command
- [Igluctl `static s3cp`][igluctl-s3cp]: documentation of the `igluctl static s3cp` command

## 3. Creating the JSON Path files and SQL table definitions

Once you've defined the jsonschema for your new event or context type you need to create a correpsonding jsonpath file and sql table definition. This can be done programmatically using Igluctl. From the root of the repo:

```bash
/path/to/igluctl static generate --with-json-paths /path/to/schemas/com.mycompany/new_event_or_context_name
```

A corresponding jsonpath file and sql table definition file will be generated in the appropriate folder in the repo.

Note that you can create SQL table definition and jsonpath files for all the events / contexts schema'd as follows:

```bash
/path/to/igluctl static generate --with-json-paths /path/to/schemas/com.mycompany
```

## 4. Uploading the jsonpath files to Iglu

Once you've finalized the new jsonpath file, commit it to Git. From the project root:

```bash
git add .
git commit -m "Committed finalized jsonpath"
git push
```

Then push to Iglu:

```bash
/path/to/igluctl static s3cp ./jsonpaths snowplow-com-mycompany-iglu-jsonpaths-bucket --accessKeyId ABCDEF --secretAccessKey GHIJKILM/12345XYZ --region us-east-1
```

## 5. Creating or updating the table definition in Redshift

Once you've committed your updated table definition into Github, you need to either create or modify the table in Redshift, either by executing the `CREATE TABLE...` statement directly, or `ALTER TABLE` (if you're e.g. adding a column to an existing table).

Note that it is essential that any new tables you create are owned by the `storageloader` user. This is the user that we use to load and model data in Redshift. Once you've created your new table:

```sql
CREATE TABLE my_new_table...
```

Please run the following statement to assign ownership of it to `storageloader`

```sql
ALTER TABLE my_new_table OWNER TO storageloader;
```

## 6. Sending data into Snowplow using the schema reference as custom unstructured events or contexts

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

Note: we recommend testing that the data you're sending into Snowplow conforms to the schemas you've defined and uploaded into Iglu, before pushing updates into production. This [online JSON schema validator][schema-validator] is a very useful resource for doing so.

We also recommend testing that the events are sent successfully using Snowplow-Mini. You do this by configuring the collector in the tracker to `$SNOWPLOW_MINI_IP` and then logging onto `http://$SNOWPLOW_MINI_IP/home/` to review the results e.g. in Kibana. (Follow the links on the page.)

## 7. Managing schema migrations

When you use Snowplow, the schema for each event and context lives with the data. That means you have the flexibility to evolve your schema definition over time.

If you want to change your schema over time, you will need to:

1. Create a new jsonschema file. Depending on how different this is to your current version, you will need to give it the appropriate version number. The [SchemaVer][schema-ver-blog] specification we use when versioning data schemas can be found [here][semver]
2. Rerun `igluctl static generate --with-json-paths` to update the jsonpath file, sql table definition and create a SQL migration script. Note that you'll need to add the `--force` flag to ensure that the updated jsonpath and sql table definition files overwrite the existing files.
3. Start sending data into Snowplow using the new schema version (i.e. update the Iglu reference to point at the new version e.g. `2-0-0` or `1-0-1` rather than `1-0-0`). Note that you will continue to be able to send in data that conforms to the old schema at the same time. In the event that you have an event with two different major schema definitions, each event version will be loaded into a different Redshift table

## Additional resources

### Documentation on jsonschemas

- Other example jsonschemas can be found in [Iglu Central][iglu-central]. Note how schemas are namespaced in different folders.
- [Schema Guru][schema-guru-github] is a command line tool for programmatically generating schemas from existing JSON data.
- [Snowplow 0.9.5 release blog post][versioning-release-blog], which gives an overview of the way that Snowplow uses jsonschemas to process, validate and shred unstructured event and custom context JSONs.
- It can be useful to test jsonschemas using online validators e.g. [this one][schema-validator].
- [json-schema.org][json-schema] contains links to the actual jsonschema specification, examples and guide for schema authors.
- The original specification for self-describing JSONs, produced by the Snowplow team, can be found [here][self-desc-blog].

### Documentation on jsonpaths

- Example jsonpath files can be found in [Iglu central][iglu-central-jsonpaths]. Note that the corresponding jsonschema definitions are also stored in [Iglu central][iglu-central-schemas].
- Amazon documentation on jsonpath files can be found [here][aws-copy-json].

### Documentation on creating tables in Redshift

- Example Redshift table definitions can be found on the [Snowplow repo][snowplow-redshift-sql].
- Amazon documentation on Redshift create table statements can be found [here][redshift-create-table]. A list of Redshift data types can be found [here][redshift-data-types].

## Copyright and license

The Iglu Example Schema Registry is copyright 2014-2022 Snowplow Analytics Ltd.

Licensed under the **[Apache License, Version 2.0][license]** (the "License");
you may not use this software except in compliance with the License.

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

[license-image]: https://img.shields.io/badge/license-Apache--2-blue.svg?style=flat
[license]: https://www.apache.org/licenses/LICENSE-2.0

[discourse-image]: https://img.shields.io/discourse/posts?server=https%3A%2F%2Fdiscourse.snowplowanalytics.com%2F
[discourse]: https://discourse.snowplowanalytics.com/

[igluctl]: https://docs.snowplowanalytics.com/docs/pipeline-components-and-applications/iglu/igluctl-2/
[igluctl-lint]: https://docs.snowplowanalytics.com/docs/pipeline-components-and-applications/iglu/igluctl-2/#lint-1
[igluctl-push]: https://docs.snowplowanalytics.com/docs/pipeline-components-and-applications/iglu/igluctl-2/#static-push
[igluctl-s3cp]: https://docs.snowplowanalytics.com/docs/pipeline-components-and-applications/iglu/igluctl-2/#static-s3cp

[iglu-central]: https://docs.snowplowanalytics.com/docs/pipeline-components-and-applications/iglu/iglu-repositories/iglu-central/
[iglu-central-jsonpaths]: https://github.com/snowplow/iglu-central/tree/master/jsonpaths
[iglu-central-schemas]: https://github.com/snowplow/iglu-central/tree/master/schemas
[iglu]: https://docs.snowplowanalytics.com/docs/pipeline-components-and-applications/iglu/
[schema-guru-github]: https://github.com/snowplow/schema-guru
[snowplow-redshift-sql]: https://github.com/snowplow/snowplow/tree/master/4-storage/redshift-storage/sql

[aws-credentials]: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#config-settings-and-precedence
[schema-validator]: https://www.jsonschemavalidator.net/
[semver]: https://semver.org/
[json-schema]: https://json-schema.org/
[aws-copy-json]: https://docs.aws.amazon.com/redshift/latest/dg/copy-usage_notes-copy-from-json.html
[redshift-create-table]: https://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_TABLE_NEW.html
[redshift-data-types]: https://docs.aws.amazon.com/redshift/latest/dg/c_Supported_data_types.html

[schema-ver-blog]: https://snowplowanalytics.com/blog/2014/05/13/introducing-schemaver-for-semantic-versioning-of-schemas/
[versioning-release-blog]: https://snowplowanalytics.com/blog/2014/07/09/snowplow-0-9-5-released-with-json-validation-shredding/
[self-desc-blog]: https://snowplowanalytics.com/blog/2014/05/15/introducing-self-describing-jsons/
[schema-repo-blog]: https://snowplowanalytics.com/blog/2014/07/01/iglu-schema-repository-released/

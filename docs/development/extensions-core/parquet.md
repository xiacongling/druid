---
id: parquet
title: "Apache Parquet Extension"
---

<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one
  ~ or more contributor license agreements.  See the NOTICE file
  ~ distributed with this work for additional information
  ~ regarding copyright ownership.  The ASF licenses this file
  ~ to you under the Apache License, Version 2.0 (the
  ~ "License"); you may not use this file except in compliance
  ~ with the License.  You may obtain a copy of the License at
  ~
  ~   http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing,
  ~ software distributed under the License is distributed on an
  ~ "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  ~ KIND, either express or implied.  See the License for the
  ~ specific language governing permissions and limitations
  ~ under the License.
  -->


This Apache Druid module extends [Druid Hadoop based indexing](../../ingestion/hadoop.md) to ingest data directly from offline
Apache Parquet files.

Note: If using the `parquet-avro` parser for Apache Hadoop based indexing, `druid-parquet-extensions` depends on the `druid-avro-extensions` module, so be sure to
 [include  both](../../development/extensions.md#loading-extensions).

## Parquet and Native Batch
This extension provides a `parquet` input format which can be used with Druid [native batch ingestion](../../ingestion/native-batch.md).

### Parquet InputFormat
|Field | Type | Description | Required|
|---|---|---|---|
|type| String| This should be set to `parquet` to read Parquet file| yes |
|flattenSpec| JSON Object |Define a [`flattenSpec`](../../ingestion/index.md#flattenspec) to extract nested values from a Parquet file. Note that only 'path' expression are supported ('jq' is unavailable).| no (default will auto-discover 'root' level properties) |
| binaryAsString | Boolean | Specifies if the bytes parquet column which is not logically marked as a string or enum type should be treated as a UTF-8 encoded string. | no (default == false) |

### Example

```json
    ...
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "local",
        "baseDir": "/some/path/to/file/",
        "filter": "file.parquet"
      },
      "inputFormat": {
        "type": "parquet"
        "flattenSpec": {
          "useFieldDiscovery": true,
          "fields": [
            {
              "type": "path",
              "name": "nested",
              "expr": "$.path.to.nested"
            }
          ]
        }
        "binaryAsString": false
      },
      ...
    }
    ...
```
## Parquet Hadoop Parser

For Hadoop, this extension provides two parser implementations for reading Parquet files:

* `parquet` - using a simple conversion contained within this extension
* `parquet-avro` - conversion to avro records with the `parquet-avro` library and using the `druid-avro-extensions`
 module to parse the avro data

Selection of conversion method is controlled by parser type, and the correct hadoop input format must also be set in
the `ioConfig`:

* `org.apache.druid.data.input.parquet.DruidParquetInputFormat` for `parquet`
* `org.apache.druid.data.input.parquet.DruidParquetAvroInputFormat` for `parquet-avro`


Both parse options support auto field discovery and flattening if provided with a
[`flattenSpec`](../../ingestion/index.md#flattenspec) with `parquet` or `avro` as the format. Parquet nested list and map
[logical types](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md) _should_ operate correctly with
JSON path expressions for all supported types. `parquet-avro` sets a hadoop job property
`parquet.avro.add-list-element-records` to `false` (which normally defaults to `true`), in order to 'unwrap' primitive
list elements into multi-value dimensions.

The `parquet` parser supports `int96` Parquet values, while `parquet-avro` does not. There may also be some subtle
differences in the behavior of JSON path expression evaluation of `flattenSpec`.

We suggest using `parquet` over `parquet-avro` to allow ingesting data beyond the schema constraints of Avro conversion.
However, `parquet-avro` was the original basis for this extension, and as such it is a bit more mature.


|Field     | Type        | Description                                                                            | Required|
|----------|-------------|----------------------------------------------------------------------------------------|---------|
| type      | String      | Choose `parquet` or `parquet-avro` to determine how Parquet files are parsed | yes |
| parseSpec | JSON Object | Specifies the timestamp and dimensions of the data, and optionally, a flatten spec. Valid parseSpec formats are `timeAndDims`, `parquet`, `avro` (if used with avro conversion). | yes |
| binaryAsString | Boolean | Specifies if the bytes parquet column which is not logically marked as a string or enum type should be treated as a UTF-8 encoded string. | no(default == false) |

When the time dimension is a [DateType column](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md), a format should not be supplied. When the format is UTF8 (String), either `auto` or a explicitly defined [format](http://www.joda.org/joda-time/apidocs/org/joda/time/format/DateTimeFormat.html) is required.

### Examples

#### `parquet` parser, `parquet` parseSpec
```json
{
  "type": "index_hadoop",
  "spec": {
    "ioConfig": {
      "type": "hadoop",
      "inputSpec": {
        "type": "static",
        "inputFormat": "org.apache.druid.data.input.parquet.DruidParquetInputFormat",
        "paths": "path/to/file.parquet"
      },
      ...
    },
    "dataSchema": {
      "dataSource": "example",
      "parser": {
        "type": "parquet",
        "parseSpec": {
          "format": "parquet",
          "flattenSpec": {
            "useFieldDiscovery": true,
            "fields": [
              {
                "type": "path",
                "name": "nestedDim",
                "expr": "$.nestedData.dim1"
              },
              {
                "type": "path",
                "name": "listDimFirstItem",
                "expr": "$.listDim[1]"
              }
            ]
          },
          "timestampSpec": {
            "column": "timestamp",
            "format": "auto"
          },
          "dimensionsSpec": {
            "dimensions": [],
            "dimensionExclusions": [],
            "spatialDimensions": []
          }
        }
      },
      ...
    },
    "tuningConfig": <hadoop-tuning-config>
    }
  }
}
```

#### `parquet` parser, `timeAndDims` parseSpec
```json
{
  "type": "index_hadoop",
  "spec": {
    "ioConfig": {
      "type": "hadoop",
      "inputSpec": {
        "type": "static",
        "inputFormat": "org.apache.druid.data.input.parquet.DruidParquetInputFormat",
        "paths": "path/to/file.parquet"
      },
      ...
    },
    "dataSchema": {
      "dataSource": "example",
      "parser": {
        "type": "parquet",
        "parseSpec": {
          "format": "timeAndDims",
          "timestampSpec": {
            "column": "timestamp",
            "format": "auto"
          },
          "dimensionsSpec": {
            "dimensions": [
              "dim1",
              "dim2",
              "dim3",
              "listDim"
            ],
            "dimensionExclusions": [],
            "spatialDimensions": []
          }
        }
      },
      ...
    },
    "tuningConfig": <hadoop-tuning-config>
  }
}

```
#### `parquet-avro` parser, `avro` parseSpec
```json
{
  "type": "index_hadoop",
  "spec": {
    "ioConfig": {
      "type": "hadoop",
      "inputSpec": {
        "type": "static",
        "inputFormat": "org.apache.druid.data.input.parquet.DruidParquetAvroInputFormat",
        "paths": "path/to/file.parquet"
      },
      ...
    },
    "dataSchema": {
      "dataSource": "example",
      "parser": {
        "type": "parquet-avro",
        "parseSpec": {
          "format": "avro",
          "flattenSpec": {
            "useFieldDiscovery": true,
            "fields": [
              {
                "type": "path",
                "name": "nestedDim",
                "expr": "$.nestedData.dim1"
              },
              {
                "type": "path",
                "name": "listDimFirstItem",
                "expr": "$.listDim[1]"
              }
            ]
          },
          "timestampSpec": {
            "column": "timestamp",
            "format": "auto"
          },
          "dimensionsSpec": {
            "dimensions": [],
            "dimensionExclusions": [],
            "spatialDimensions": []
          }
        }
      },
      ...
    },
    "tuningConfig": <hadoop-tuning-config>
    }
  }
}
```

For additional details see [Hadoop ingestion](../../ingestion/hadoop.md) and [general ingestion spec](../../ingestion/index.md) documentation.

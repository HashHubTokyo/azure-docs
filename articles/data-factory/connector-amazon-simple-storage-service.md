---
title: Copy data from Amazon Simple Storage Service using Azure Data Factory | Microsoft Docs
description: Learn about how to copy data from Amazon Simple Storage Service (S3) to supported sink data stores by using Azure Data Factory.
services: data-factory
author: linda33wj
manager: craigg
ms.reviewer: douglasl

ms.service: data-factory
ms.workload: data-services
ms.topic: conceptual
ms.date: 12/20/2018
ms.author: jingwang

---
# Copy data from Amazon Simple Storage Service using Azure Data Factory
> [!div class="op_single_selector" title1="Select the version of Data Factory service you are using:"]
> * [Version 1](v1/data-factory-amazon-simple-storage-service-connector.md)
> * [Current version](connector-amazon-simple-storage-service.md)

This article outlines how to use the Copy Activity in Azure Data Factory to copy data from Amazon S3. It builds on the [copy activity overview](copy-activity-overview.md) article that presents a general overview of copy activity.

## Supported capabilities

You can copy data Amazon S3 to any supported sink data store. For a list of data stores that are supported as sources or sinks by the copy activity, see the [Supported data stores](copy-activity-overview.md#supported-data-stores-and-formats) table.

Specifically, this Amazon S3 connector supports copying files as-is or parsing files with the [supported file formats and compression codecs](supported-file-formats-and-compression-codecs.md).

>[!TIP]
>You can use this Amazon S3 connector to copy data from **any S3-compatible storage providers** e.g. [Google Cloud Storage](#copy-from-google-cloud-storage). Specify the corresponding service URL in the linked service configuration.

## Required permissions

To copy data from Amazon S3, make sure you have been granted the following permissions:

- **For copy activity execution:**: `s3:GetObject` and `s3:GetObjectVersion` for Amazon S3 Object Operations.
- **For Data Factory GUI authoring**: `s3:ListAllMyBuckets` and `s3:ListBucket`/`s3:GetBucketLocation` for Amazon S3 Bucket Operations permissions are additionally required for operations like test connection and browse/navigate file paths. If you don't want to grant these permission, skip test connection in linked service creation page and speicify the path directly in dataset settings.

For details about the full list of Amazon S3 permissions, see [Specifying Permissions in a Policy](https://docs.aws.amazon.com/AmazonS3/latest/dev/using-with-s3-actions.html).

## Getting started

[!INCLUDE [data-factory-v2-connector-get-started](../../includes/data-factory-v2-connector-get-started.md)] 

The following sections provide details about properties that are used to define Data Factory entities specific to Amazon S3.

## Linked service properties

The following properties are supported for Amazon S3 linked service:

| Property | Description | Required |
|:--- |:--- |:--- |
| type | The type property must be set to **AmazonS3**. | Yes |
| accessKeyId | ID of the secret access key. |Yes |
| secretAccessKey | The secret access key itself. Mark this field as a SecureString to store it securely in Data Factory, or [reference a secret stored in Azure Key Vault](store-credentials-in-key-vault.md). |Yes |
| serviceUrl | Specify the custom S3 endpoint if you are copying data from a S3-compatible storage provider other than the official Amazon S3 service. For example, to [copy data from Google Cloud Storage](#copy-from-google-cloud-storage), specify `https://storage.googleapis.com`. | No |
| connectVia | The [Integration Runtime](concepts-integration-runtime.md) to be used to connect to the data store. You can use Azure Integration Runtime or Self-hosted Integration Runtime (if your data store is located in private network). If not specified, it uses the default Azure Integration Runtime. |No |

>[!TIP]
>Specify the custom S3 service URL if you are copying data from a S3-compatible storage other than the official Amazon S3 service.

>[!NOTE]
>This connector requires access keys for IAM account to copy data from Amazon S3. [Temporary Security Credential](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html) is not supported.
>

Here is an example:

```json
{
    "name": "AmazonS3LinkedService",
    "properties": {
        "type": "AmazonS3",
        "typeProperties": {
            "accessKeyId": "<access key id>",
            "secretAccessKey": {
                "type": "SecureString",
                "value": "<secret access key>"
            }
        },
        "connectVia": {
            "referenceName": "<name of Integration Runtime>",
            "type": "IntegrationRuntimeReference"
        }
    }
}
```

## Dataset properties

For a full list of sections and properties available for defining datasets, see the datasets article. This section provides a list of properties supported by Amazon S3 dataset.

To copy data from Amazon S3, set the type property of the dataset to **AmazonS3Object**. The following properties are supported:

| Property | Description | Required |
|:--- |:--- |:--- |
| type | The type property of the dataset must be set to: **AmazonS3Object** |Yes |
| bucketName | The S3 bucket name. Wildcard filter is not supported. |Yes for Copy/Lookup activity, No for GetMetadata activity |
| key | The **name or wildcard filter** of S3 object key under the specified bucket. Applies only when "prefix" property is not specified. <br/><br/>The wildcard filter is only supported for file name part but not folder part. Allowed wildcards are: `*` (matches zero or more characters) and `?` (matches zero or single character).<br/>- Example 1: `"key": "rootfolder/subfolder/*.csv"`<br/>- Example 2: `"key": "rootfolder/subfolder/???20180427.txt"`<br/>Use `^` to escape if your actual file name has wildcard or this escape char inside. |No |
| prefix | Prefix for the S3 object key. Objects whose keys start with this prefix are selected. Applies only when "key" property is not specified. |No |
| version | The version of the S3 object, if S3 versioning is enabled. |No |
| modifiedDatetimeStart | Files filter based on the attribute: Last Modified. The files will be selected if their last modified time are within the time range between `modifiedDatetimeStart` and `modifiedDatetimeEnd`. The time is applied to UTC time zone in the format of "2018-12-01T05:00:00Z". <br/><br/> The properties can be NULL which mean no file attribute filter will be applied to the dataset.  When `modifiedDatetimeStart` has datetime value but `modifiedDatetimeEnd` is NULL, it means the files whose last modified attribute is greater than or equal with the datetime value will be selected.  When `modifiedDatetimeEnd` has datetime value but `modifiedDatetimeStart` is NULL, it means the files whose last modified attribute is less than the datetime value will be selected.| No |
| modifiedDatetimeEnd | Files filter based on the attribute: Last Modified. The files will be selected if their last modified time are within the time range between `modifiedDatetimeStart` and `modifiedDatetimeEnd`. The time is applied to UTC time zone in the format of "2018-12-01T05:00:00Z". <br/><br/> The properties can be NULL which mean no file attribute filter will be applied to the dataset.  When `modifiedDatetimeStart` has datetime value but `modifiedDatetimeEnd` is NULL, it means the files whose last modified attribute is greater than or equal with the datetime value will be selected.  When `modifiedDatetimeEnd` has datetime value but `modifiedDatetimeStart` is NULL, it means the files whose last modified attribute is less than the datetime value will be selected.| No |
| format | If you want to **copy files as-is** between file-based stores (binary copy), skip the format section in both input and output dataset definitions.<br/><br/>If you want to parse or generate files with a specific format, the following file format types are supported: **TextFormat**, **JsonFormat**, **AvroFormat**, **OrcFormat**, **ParquetFormat**. Set the **type** property under format to one of these values. For more information, see [Text Format](supported-file-formats-and-compression-codecs.md#text-format), [Json Format](supported-file-formats-and-compression-codecs.md#json-format), [Avro Format](supported-file-formats-and-compression-codecs.md#avro-format), [Orc Format](supported-file-formats-and-compression-codecs.md#orc-format), and [Parquet Format](supported-file-formats-and-compression-codecs.md#parquet-format) sections. |No (only for binary copy scenario) |
| compression | Specify the type and level of compression for the data. For more information, see [Supported file formats and compression codecs](supported-file-formats-and-compression-codecs.md#compression-support).<br/>Supported types are: **GZip**, **Deflate**, **BZip2**, and **ZipDeflate**.<br/>Supported levels are: **Optimal** and **Fastest**. |No |

>[!TIP]
>To copy all files under a folder, specify **bucketName** for bucket and **prefix** for folder part.<br>To copy a single file with a given name, specify **bucketName** for bucket and **key** for folder part plus file name.<br>To copy a subset of files under a folder, specify **bucketName** for bucket and **key** for folder part plus wildcard filter.

**Example: using prefix**

```json
{
    "name": "AmazonS3Dataset",
    "properties": {
        "type": "AmazonS3Object",
        "linkedServiceName": {
            "referenceName": "<Amazon S3 linked service name>",
            "type": "LinkedServiceReference"
        },
        "typeProperties": {
            "bucketName": "testbucket",
            "prefix": "testFolder/test",
            "modifiedDatetimeStart": "2018-12-01T05:00:00Z",
            "modifiedDatetimeEnd": "2018-12-01T06:00:00Z",
            "format": {
                "type": "TextFormat",
                "columnDelimiter": ",",
                "rowDelimiter": "\n"
            },
            "compression": {
                "type": "GZip",
                "level": "Optimal"
            }
        }
    }
}
```

**Example: using key and version (optional)**

```json
{
    "name": "AmazonS3Dataset",
    "properties": {
        "type": "AmazonS3",
        "linkedServiceName": {
            "referenceName": "<Amazon S3 linked service name>",
            "type": "LinkedServiceReference"
        },
        "typeProperties": {
            "bucketName": "testbucket",
            "key": "testFolder/testfile.csv.gz",
            "version": "XXXXXXXXXczm0CJajYkHf0_k6LhBmkcL",
            "format": {
                "type": "TextFormat",
                "columnDelimiter": ",",
                "rowDelimiter": "\n"
            },
            "compression": {
                "type": "GZip",
                "level": "Optimal"
            }
        }
    }
}
```

## Copy activity properties

For a full list of sections and properties available for defining activities, see the [Pipelines](concepts-pipelines-activities.md) article. This section provides a list of properties supported by Amazon S3 source.

### Amazon S3 as source

To copy data from Amazon S3, set the source type in the copy activity to **FileSystemSource** (which includes Amazon S3). The following properties are supported in the copy activity **source** section:

| Property | Description | Required |
|:--- |:--- |:--- |
| type | The type property of the copy activity source must be set to: **FileSystemSource** |Yes |
| recursive | Indicates whether the data is read recursively from the sub folders or only from the specified folder. Note when recursive is set to true and sink is file-based store, empty folder/sub-folder will not be copied/created at sink.<br/>Allowed values are: **true** (default), **false** | No |

**Example:**

```json
"activities":[
    {
        "name": "CopyFromAmazonS3",
        "type": "Copy",
        "inputs": [
            {
                "referenceName": "<Amazon S3 input dataset name>",
                "type": "DatasetReference"
            }
        ],
        "outputs": [
            {
                "referenceName": "<output dataset name>",
                "type": "DatasetReference"
            }
        ],
        "typeProperties": {
            "source": {
                "type": "FileSystemSource",
                "recursive": true
            },
            "sink": {
                "type": "<sink type>"
            }
        }
    }
]
```

## Copy from Google Cloud Storage

Because Google Cloud Storage provides S3-compatible interoperability, you can use Amazon S3 connector to copy data from Google Cloud Storage to any [supported sink data stores](copy-activity-overview.md#supported-data-stores-and-formats). 

You can find the specific Google Cloud Storage entry in the ADF authoring UI connector gallery, which automatically fills in the service URL as `https://storage.googleapis.com`. To find the access key and secret, go to **Google Cloud Storage** > **Settings** > **Interoperability**. Refer to this article from the beginning for a detailed overview of using S3 connector to copy data.

**Example linked service:**

```json
{
    "name": "GoogleCloudStorageLinkedService",
    "properties": {
        "type": "AmazonS3",
        "typeProperties": {
            "accessKeyId": "<access key id>",
            "secretAccessKey": {
                "type": "SecureString",
                "value": "<secret access key>"
            },
            "serviceUrl": "https://storage.googleapis.com"
        },
        "connectVia": {
            "referenceName": "<name of Integration Runtime>",
            "type": "IntegrationRuntimeReference"
        }
    }
}
```

## Next steps
For a list of data stores that are supported as sources and sinks by the copy activity in Azure Data Factory, see [supported data stores](copy-activity-overview.md##supported-data-stores-and-formats).

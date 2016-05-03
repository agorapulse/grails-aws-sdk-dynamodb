Grails AWS SDK DynamoDB Plugin
==============================

[![Build Status](https://travis-ci.org/agorapulse/grails-aws-sdk-dynamodb.svg?branch=master)](https://travis-ci.org/agorapulse/grails-aws-sdk-dynamodb)
[![Download](https://api.bintray.com/packages/agorapulse/plugins/aws-sdk-dynamodb/images/download.svg)](https://bintray.com/agorapulse/plugins/aws-sdk-dynamodb/_latestVersion)

The [AWS SDK Plugins for Grails3](https://medium.com/@benorama/aws-sdk-plugins-for-grails-3-cc7f910fdc0d#.5gdwdxei3) are a suite of plugins that adds support for the [Amazon Web Services](http://aws.amazon.com/) infrastructure services.

The aim is to to get you started quickly by providing friendly lightweight utility [Grails](http://grails.org) service wrappers, around the official [AWS SDK for Java](http://aws.amazon.com/sdkforjava/) (which is great but very “java-esque”).
See [this article](https://medium.com/@benorama/aws-sdk-plugins-for-grails-3-cc7f910fdc0d#.5gdwdxei3) for more info.

The following services are currently supported:

* [AWS SDK DynamoDB Grails Plugin](http://github.com/agorapulse/grails-aws-sdk-dynamodb)
* [AWS SDK Kinesis Grails Plugin](http://github.com/agorapulse/grails-aws-sdk-kinesis)
* [AWS SDK S3 Grails Plugin](http://github.com/agorapulse/grails-aws-sdk-s3)
* [AWS SDK SES Grails Plugin](http://github.com/agorapulse/grails-aws-sdk-ses)
* [AWS SDK SQS Grails Plugin](http://github.com/agorapulse/grails-aws-sdk-sqs)

# Introduction

This plugin adds support for [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), a fast and flexible NoSQL database service for all applications that need consistent, single-digit millisecond latency at any scale. 
It is a fully managed cloud database and supports both document and key-value store models.


# Installation

Add plugin dependency to your `build.gradle`:

```groovy
repositories {
    ...
    maven { url "http://dl.bintray.com/agorapulse/plugins" } // TEMP, to remove once the plugin is officially released
    ...
}

dependencies {
  ...
  compile 'org.grails.plugins:aws-sdk-dynamodb:2.0.0-beta2'
  ...
```

# Config

Create an AWS account [Amazon Web Services](http://aws.amazon.com/), in order to get your own credentials accessKey and secretKey.


## AWS SDK for Java version

You can override the default AWS SDK for Java version by setting it in your _gradle.properties_:

```
awsJavaSdkVersion=1.10.66
```

## Credentials

Add your AWS credentials parameters to your _grails-app/conf/application.yml_:

```yml
grails:
    plugin:
        awssdk:
            accessKey: {ACCESS_KEY}
            secretKey: {SECRET_KEY}
```

If you do not provide credentials, a credentials provider chain will be used that searches for credentials in this order:

* Environment Variables - `AWS_ACCESS_KEY_ID` and `AWS_SECRET_KEY`
* Java System Properties - `aws.accessKeyId and `aws.secretKey`
* Instance profile credentials delivered through the Amazon EC2 metadata service (IAM role)

## Region

The default region used is **us-east-1**. You might override it in your config:

```yml
grails:
    plugin:
        awssdk:
            region: eu-west-1
```

If you're using multiple AWS SDK Grails plugins, you can define specific settings for each services.

```yml
grails:
    plugin:
        awssdk:
            accessKey: {ACCESS_KEY} # Global default setting 
            secretKey: {SECRET_KEY} # Global default setting
            region: us-east-1       # Global default setting
            dynamodb:
                accessKey: {ACCESS_KEY} # Service setting (optional)
                secretKey: {SECRET_KEY} # Service setting (optional)
                region: eu-west-1       # Service setting (optional)
            
```

# Usage

## Data modeling

The plugin is based on [DynamoDBMapper](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.html) to model data.
Just create simple groovy beans in `src/main/groovy` with specific DynamoDB java annotations (no GORM here).

Please check the documentation to see all the available mapper [java annotations](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.Annotations.html).

Here is an example.

```groovy
import com.amazonaws.services.dynamodbv2.datamodeling.*

@DynamoDBTable(tableName="FooItem")
@ToString(includeNames=true, includeFields=true)
class FooItem {

    @DynamoDBHashKey
    Long accountId // Hash key
    @DynamoDBRangeKey
    @DynamoDBAutoGeneratedKey
    String id // Range key, with an automatically generated ID

    @DynamoDBAttribute
    int count = 0
    @DynamoDBAttribute
    @DynamoDBIndexRangeKey(localSecondaryIndexName="creationDate")
    Date creationDate
    @DynamoDBAttribute
    String message
    @DynamoDBAttribute
    String title

}
```

## Service definition

Once you have modeled your data, create one DB Grails service per bean by extending `AbstractDBService`.

```groovy
import grails.plugins.awssdk.dynamodb.AbstractDBService

class FooItemDBService extends AbstractDBService<FooItem> {

    FooItemDBService() {
        super(FooItem)
    }

}
```

## Table management

If your table does not exist yet, the DB service provides a method for that.

```groovy
// Create table
ctx.fooItemDBService.createTable()
```

## Creating or updating items

The DB service provides plenty of methods to manipulate your data. 
Here are some common examples.

```groovy
hashKey = 123456789L
rangeKey = '6733308f-5f64-48d2-824d-665290664ae0'

// Save item
foo = new FooItem(
    accountId: hashKey,
    creationDate: new Date(),
    message: 'Some message...',
    title: 'Some title'
)
ctx.fooItemDBService.save(foo)

// Saving multiple items
ctx.fooItemDBService.saveAll(foo, bar)

// Updating a single attribute
ctx.fooItemDBService.updateItemAttribute(hashKey, rangeKey, 'title', 'Some title updated')

// Decrementing a numeric attribute (atomic operation)
ctx.fooItemDBService.decrement(hashKey, rangeKey, 'count', 2)
// Incrementing a numeric attribute (atomic operation)
ctx.fooItemDBService.increment(hashKey, rangeKey, 'count', 10)
```

## Loading items

```groovy
// Load an item
foo = ctx.fooItemDBService.get(hashKey, rangeKey)

// Load multiple items
items = ctx.fooItemDBService.get(hashKey, [rangeKey1, rangeKey2])
```

## Deleting items

To avoid to consume too much read units, all delete methods use the following settings by default:

```groovy
settings = [
    limit: 100
]
```

You can override those settings when calling each methods by passing a settings map (last parameter).

```groovy
// Delete an item
ctx.fooItemDBService.delete(foo)
// Or by hash and range keys
ctx.fooItemDBService.delete(hashKey, rangeKey)

// Delete a list of items
ctx.fooItemDBService.deleteAll([foo, bar])

// Delete all items for a given hashKey (WARNING: deleting can consume a lot of write units)
ctx.fooItemDBService.deleteAll(hashKey)

// Delete all items by simple condition
ctx.fooItemDBService.deleteAll(hashKey, 'id', '4532432-', ComparisonOperator.BEGINS_WITH)

// Delete all items by conditions (with a Map<String, Condition>)
ctx.fooItemDBService.deleteAllByConditions(hashKey, rangeKeyConditions)
```

## Counting items

To avoid to consume too much read units, all count methods use the following settings by default:

* `consistentRead` (default to false)
* `limit` (default to 100)

You can override those settings when calling each methods by passing a settings map (last method parameter).

```groovy
// Count items by simple condition
ctx.fooItemDBService.count(hashKey, 'id', '4532432-', ComparisonOperator.BEGINS_WITH)

// Count items by conditions (with a Map<String, Condition>)
ctx.fooItemDBService.countByConditions(hashKey, rangeKeyConditions)

// Count items by dates range
ctx.fooItemDBService.countByDates(hashKey, 'creationDate', [after: new Date() - 7, before: new Date() - 1])
```

## Querying items

To avoid to consume too much read units, all count methods use the following settings by default:

* `batchGetDisabled` (only when secondary indexes are used, useful for count when all item attributes are not required)
* `consistentRead` (default to false)
* `exclusiveStartKey` a map with the rangeKey (ex: [id: 2555]), with optional indexRangeKey when using LSI (ex.: [id: 2555, totalCount: 45])
* `limit` (default to 100)
* `returnAll` disable paging to return all items, WARNING: can be expensive in terms of throughput (default to false)
* `scanIndexForward` (default to false)

```groovy
// Query items by hash key
items = ctx.fooItemDBService.query(hashKey).results

// Query items by simple condition
items = ctx.fooItemDBService.query(hashKey, 'id', '4532432-', ComparisonOperator.BEGINS_WITH).results

// Query items by conditions (with a Map<String, Condition>)
items = ctx.fooItemDBService.queryByConditions(hashKey, rangeKeyConditions).results

// Query items by dates range
items = ctx.fooItemDBService.queryByDates(hashKey, 'creationDate', [after: new Date() - 7, before: new Date() - 1]).results
```

## Advanced usage

If required, you can also directly use **AmazonDynamoDBClient** instance available at **fooItemDBService.client** and **DynamoDBMapper** instance available at **fooItemDBService.mapper**.

For more info, AWS SDK for Java documentation is located here:

* [AWS SDK for Java](http://docs.amazonwebservices.com/AWSJavaSDK/latest/javadoc/index.html)


# Bugs

To report any bug, please use the project [Issues](http://github.com/agorapulse/grails-aws-sdk-dynamodb/issues) section on GitHub.

Feedback and pull requests are welcome!
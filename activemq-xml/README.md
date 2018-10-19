# Introduction

Confluent has a handy dandy connector for pulling data from 
[ActiveMQ](https://www.confluent.io/connector/kafka-connect-activemq/) as well as other 
[JMS](https://www.confluent.io/connector/kafka-connect-jms/) systems like 
[IBM MQ](https://www.confluent.io/connector/kafka-connect-ibm-mq/). This does a great job monitoring
a queue or topic for changes and writing their data to Kafka in real time. One item of feedback commonly 
encounter is that users want something other than the envelope format. For example this is written 
to Kafka by default. This approach uses a couple transformations to assist. 
[org.apache.kafka.connect.transforms.ExtractField$Value](https://kafka.apache.org/documentation/#connect_transforms) and [com.github.jcustenborder.kafka.connect.transform.xml.FromXml$Value](https://www.confluent.io/connector/xml-transformation/)

```json
/* Example is avro represented as json */
{
  "messageID": "ID:42a7c10aa0f9-39545-1539880636464-6:1:1:1:1",
  "messageType": "text",
  "timestamp": 1539880718965,
  "deliveryMode": 1,
  "correlationID": {
    "string": ""
  },
  "replyTo": null,
  "destination": {
    "io.confluent.connect.jms.Destination": {
      "destinationType": "queue",
      "name": "jmsdata"
    }
  },
  "redelivered": false,
  "type": {
    "string": ""
  },
  "expiration": 0,
  "priority": 0,
  "properties": {},
  "bytes": null,
  "map": null,
  "text": {
    "string": "<?xml version=\"1.0\"?>\r\n<x:books xmlns:x=\"urn:books\">\r\n    <book id=\"bk001\">\r\n        <author>Writer</author>\r\n        <title>The First Book</title>\r\n        <genre>Fiction</genre>\r\n        <price>44.95</price>\r\n        <pub_date>2000-10-01</pub_date>\r\n        <review>An amazing story of nothing.</review>\r\n    </book>\r\n\r\n    <book id=\"bk002\">\r\n        <author>Poet</author>\r\n        <title>The Poet's First Poem</title>\r\n        <genre>Poem</genre>\r\n        <price>24.95</price>\r\n        <pub_date>2000-10-01</pub_date>\r\n        <review>Least poetic poems.</review>\r\n    </book>\r\n</x:books>"
  }
}
```

What if I only want the value of the `text` field? So something like this example. This transformation
is handled by [org.apache.kafka.connect.transforms.ExtractField$Value](https://kafka.apache.org/documentation/#connect_transforms)

```json
/* Example is avro represented as json */
"<?xml version=\"1.0\"?>\r\n<x:books xmlns:x=\"urn:books\">\r\n    <book id=\"bk001\">\r\n        <author>Writer</author>\r\n        <title>The First Book</title>\r\n        <genre>Fiction</genre>\r\n        <price>44.95</price>\r\n        <pub_date>2000-10-01</pub_date>\r\n        <review>An amazing story of nothing.</review>\r\n    </book>\r\n\r\n    <book id=\"bk002\">\r\n        <author>Poet</author>\r\n        <title>The Poet's First Poem</title>\r\n        <genre>Poem</genre>\r\n        <price>24.95</price>\r\n        <pub_date>2000-10-01</pub_date>\r\n        <review>Least poetic poems.</review>\r\n    </book>\r\n</x:books>"
```

Taking it even further lets use an XSD to convert XML data to data that has a connect schema. This is 
done by [com.github.jcustenborder.kafka.connect.transform.xml.FromXml$Value](https://www.confluent.io/connector/xml-transformation/)

```json
/* Example is avro represented as json */
{
  "book": {
    "array": [
      {
        "xsdabaa18e1e7d7ec3921c64f8e7ccaa5026b514d55.BookForm": {
          "author": "Writer",
          "title": "The First Book",
          "genre": "Fiction",
          "price": {
            "float": 44.95
          },
          "pub_date": 11231,
          "review": "An amazing story of nothing.",
          "id": {
            "string": "bk001"
          }
        }
      },
      {
        "xsdabaa18e1e7d7ec3921c64f8e7ccaa5026b514d55.BookForm": {
          "author": "Poet",
          "title": "The Poet's First Poem",
          "genre": "Poem",
          "price": {
            "float": 24.95
          },
          "pub_date": 11231,
          "review": "Least poetic poems.",
          "id": {
            "string": "bk002"
          }
        }
      }
    ]
  }
}
```

# Getting started

Start the cluster

```bash
docker-compose up -d
```

Send the following to ActiveMQ using the [admin interface](http://localhost:8161/admin/send.jsp?JMSDestination=jmsdata&JMSDestinationType=queue). Use the following credentials. `admin/admin`

```xml
<?xml version="1.0"?>
<x:books xmlns:x="urn:books">
    <book id="bk001">
        <author>Writer</author>
        <title>The First Book</title>
        <genre>Fiction</genre>
        <price>44.95</price>
        <pub_date>2000-10-01</pub_date>
        <review>An amazing story of nothing.</review>
    </book>

    <book id="bk002">
        <author>Poet</author>
        <title>The Poet's First Poem</title>
        <genre>Poem</genre>
        <price>24.95</price>
        <pub_date>2000-10-01</pub_date>
        <review>Least poetic poems.</review>
    </book>
</x:books>
```

Start the connector

```bash
curl -s -X POST -H 'Content-Type: application/json' --data @connector.json http://localhost:8083/connectors
```

You should receive a response like this:

```json
{"name":"connector1","config":{"connector.class":"io.confluent.connect.activemq.ActiveMQSourceConnector","kafka.topic":"jms","activemq.url":"tcp://activemq:61616","jms.destination.name":"jmsdata","confluent.license":"","confluent.topic.bootstrap.servers":"kafka:9092","confluent.topic.replication.factor":"1","transforms":"messageField,FromXml","transforms.messageField.type":"org.apache.kafka.connect.transforms.ExtractField$Value","transforms.messageField.field":"text","transforms.FromXml.type":"com.github.jcustenborder.kafka.connect.transform.xml.FromXml$Value","transforms.FromXml.schema.path":"file:///books.xsd","name":"connector1"},"tasks":[],"type":null}
```

Now that the connector is running lets check out the data

```bash
docker-compose exec connect kafka-avro-console-consumer --bootstrap-server kafka:9092 --property schema.registry.url=http://schema-registry:8081 --from-beginning -topic jms
```

```json
{"book":{"array":[{"xsdabaa18e1e7d7ec3921c64f8e7ccaa5026b514d55.BookForm":{"author":"Writer","title":"The First Book","genre":"Fiction","price":{"float":44.95},"pub_date":11231,"review":"An amazing story of nothing.","id":{"string":"bk001"}}},{"xsdabaa18e1e7d7ec3921c64f8e7ccaa5026b514d55.BookForm":{"author":"Poet","title":"The Poet's First Poem","genre":"Poem","price":{"float":24.95},"pub_date":11231,"review":"Least poetic poems.","id":{"string":"bk002"}}}]}}
```


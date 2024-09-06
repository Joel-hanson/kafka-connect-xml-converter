# kafka-connect-xml-converter

A Kafka Connect plugin to make it easier to work with XML data in Kafka Connect pipelines.

## Contents

- `com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlConverter`
    - a Kafka Connect converter for converting to/from XML strings
- `com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlTransformation`
    - a Kafka Connect transformation for converting Kafka Connect records to/from XML strings
- `com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlMQRecordBuilder`
    - an MQ Source Record builder for parsing MQ messages containing XML strings

## Overview

The XML Converter plugin for Kafka Connect enables Kafka to ingest and produce messages in XML format. This converter can be used with both source connectors (to bring XML data into Kafka) and sink connectors (to write XML data from Kafka to external systems). Additionally, it supports schema generation with XSD (XML Schema Definition) to validate message structures.

## Why Use the XML Converter?

Kafka Connect natively handles data in binary form, but many external systems use structured data formats like JSON, Avro, or XML. The XML Converter helps transform XML-formatted messages into Kafka’s internal data structure and vice versa. This allows seamless integration with systems that rely on XML while ensuring compatibility with Kafka’s processing capabilities.

## Configuration

Optional configuration that can be set when using the plugin to turn XML strings into Connect records

| **Option**             | **Default value** | **Notes**                                                        |
|------------------------|-------------------|------------------------------------------------------------------|
| `root.element.name`    | `root`            | The name of the root element in the XML document being parsed.   |
| `xsd.schema.path`      |                   | Location of a schema file to use to parse the XML string. |
| `xml.doc.flat.enable`  | `false`           | Set to true if the XML strings contain a single value (e.g. `<root>the message</root>`) |

Optional configuration that can be set when using the plugin to create XML strings from Connect records

| **Option**             | **Default value** | **Notes**                                                        |
|------------------------|-------------------|------------------------------------------------------------------|
| `root.element.name`    | `root`            | The name to use for the root element of the XML document being created. Only used when no name can be found within the schema of the Connect record. |

## Example uses


### 1. Using **`XmlConverter`** with Source Connectors

**`XmlConverter`** can be used with source connectors to produce structured records as XML strings. This is helpful when you want to stream data from a source system into Kafka in XML format.

#### Basic Example (No Schema):
```properties
value.converter=com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlConverter
value.converter.schemas.enable=false
```
This configuration converts Kafka records to plain XML strings without attaching any schema information. It's ideal for simple XML messages that don't need validation.

**Example Output**:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<order>
    <id>68579620-fccc-4acb-bfcc-3686919346aa</id>
    <customer>Taren Reichert</customer>
    <customerid>ef8db9c1-c685-4600-be30-bbd1fa2f7efd</customerid>
    <description>XS Retro Boyfriend Jeans</description>
    <price>16.41</price>
    <quantity>6</quantity>
    <region>APAC</region>
    <ordertime>2023-10-29 13:20:17.389</ordertime>
</order>
```

#### Example with Schema (Requires Structs):
```properties
value.converter=com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlConverter
value.converter.schemas.enable=true
```
Enabling `schemas.enable=true` ensures that an XSD schema is embedded within the XML message, making it suitable for scenarios where data validation or structure enforcement is required.

**Example Output with Embedded XSD Schema**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<order xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="#connectSchema">
    <xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" id="connectSchema">
        <xs:element name="order">
            <xs:complexType>
                <xs:sequence>
                  <xs:any maxOccurs="1" minOccurs="0" namespace="http://www.w3.org/2001/XMLSchema" processContents="skip"/>
                  <xs:element name="id" type="xs:string" />
                  <xs:element name="customer" type="xs:string" />
                  <xs:element name="customerid" type="xs:string" />
                  <xs:element name="description" type="xs:string" />
                  <xs:element name="price" type="xs:double" />
                  <xs:element name="quantity" type="xs:integer" />
                  <xs:element name="region" type="xs:string" />
                  <xs:element name="ordertime" type="xs:string" />
                </xs:sequence>
            </xs:complexType>
        </xs:element>
    </xs:schema>

    <id>55031d01-1bea-4f3b-9b29-12a968d8230e</id>
    <customer>Gabriel Stracke</customer>
    <customerid>7a75009c-5781-4074-9bf0-20dee537dc9d</customerid>
    <description>XXL Stonewashed Capri Jeans</description>
    <price>28.71</price>
    <quantity>3</quantity>
    <region>SA</region>
    <ordertime>2023-10-29 13:13:20.675</ordertime>
</order>
```

---

### 2. Using **`XmlTransformation`** with Sink Connectors

The **`XmlTransformation`** can be used with sink connectors to transform an incoming XML string into a structured Kafka Connect record. This is useful when consuming XML messages from Kafka topics and sending them to external systems in structured formats.

#### Example:
```properties
transforms.xmlconvert.type=com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlTransformation
transforms.xmlconvert.converter.type=value
transforms.xmlconvert.root.element.name=order
value.converter=org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable=false
```
This configuration transforms the XML string in the value part of the Kafka message into a structured Connect record, which can then be processed by the sink connector.

**Example Input**:
```xml
<order>
    <id>12345</id>
    <customer>John Doe</customer>
    <description>Product Description</description>
    <price>100.00</price>
</order>
```

**Example Structured Record Output**:
```json
{
    "id": "12345",
    "customer": "John Doe",
    "description": "Product Description",
    "price": 100.00
}
```

---

### 3. Sending Non-XML Kafka Messages as XML to MQ (Using **MQ Sink Connector**)

The **`XmlConverter`** can also be used with the MQ Sink Connector to convert non-XML messages from Kafka into XML format before sending them to an MQ queue.

#### Example:
```properties
mq.message.builder=com.ibm.eventstreams.connect.mqsink.builders.ConverterMessageBuilder
mq.message.builder.value.converter=com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlConverter
```
This configuration converts Kafka messages into XML format using the `XmlConverter` before forwarding them to the target MQ queue.

**Example Input** (Kafka message):
```json
{
    "id": "12345",
    "customer": "John Doe",
    "description": "Product Description",
    "price": 100.00
}
```

**Example Output** (XML in MQ queue):
```xml
<order>
    <id>12345</id>
    <customer>John Doe</customer>
    <description>Product Description</description>
    <price>100.00</price>
</order>
```

---

### 4. Converting XML from MQ Queues into Connect Records (Using **MQ Source Connector**)

With the **`XmlMQRecordBuilder`**, you can convert XML messages from MQ queues into structured Kafka Connect records, making it possible to ingest XML data from MQ into Kafka topics.

#### Example:
```properties
mq.record.builder=com.ibm.eventstreams.kafkaconnect.plugins.xml.XmlMQRecordBuilder
mq.record.builder.schemas.enable=true
mq.record.builder.xsd.schema.path=/location/of/mq-message-schema.xsd
```
This setup extracts XML messages from MQ, validates them against the specified XSD schema, and converts them into structured Kafka Connect records.

**Example Input** (XML from MQ):
```xml
<order>
    <id>12345</id>
    <customer>John Doe</customer>
    <description>Product Description</description>
    <price>100.00</price>
</order>
```

**Example Output** (Structured Kafka record):
```json
{
    "id": "12345",
    "customer": "John Doe",
    "description": "Product Description",
    "price": 100.00
}
```

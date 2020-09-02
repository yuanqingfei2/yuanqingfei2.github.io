---
title: XML Processing disable external
date: "2020-09-02T11:23:00.000Z"
description: "XML Processing disable external"
---
最近用Sonar扫描老代码的时候，发现了一个blocker级别的问题，就是xml在处理之前，生成的Facotry必须声明Disable外部实体。否则会有安全风险，具体如下。

## 问题

```java
return DocumentBuilderFactory.newInstance().newDocumentBuilder();
```

上面的代码就没有配置，直接用的默认设置，这就是问题根源。根据[Sonar Rule](https://rules.sonarsource.com/java/tag/owasp/RSPEC-2755)，这个问题有如下表现形式：

```java
// DocumentBuilderFactory library:
String xml = "xxe.xml";
DocumentBuilderFactory df = DocumentBuilderFactory.newInstance();  // Noncompliant
DocumentBuilder builder = df.newDocumentBuilder();
Document document = builder.parse(new InputSource(xml));
DOMSource domSource = new DOMSource(document);

// SAXParserFactory library:
String xml = "xxe.xml";
SaxHandler handler = new SaxHandler();
SAXParserFactory factory = SAXParserFactory.newInstance();
SAXParser parser = factory.newSAXParser();  // Noncompliant
parser.parse(xml, handler);

// XMLInputFactory library:
XMLInputFactory factory = XMLInputFactory.newInstance();  // Noncompliant
XMLEventReader eventReader = factory.createXMLEventReader(new FileReader("xxe.xml"));

// TransformerFactory library:
String xslt = "xxe.xsl";
String xml = "xxe.xml";
TransformerFactory transformerFactory = javax.xml.transform.TransformerFactory.newInstance();  // Noncompliant
Transformer transformer = transformerFactory.newTransformer(new StreamSource(xslt));

StringWriter writer = new StringWriter();
transformer.transform(new StreamSource(xml), new StreamResult(writer));
String result = writer.toString();

// SchemaFactory library:
String xsd = "xxe.xsd";
StreamSource xsdStreamSource = new StreamSource(xsd);

SchemaFactory schemaFactory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);  // Noncompliant
Schema schema = schemaFactory.newSchema(xsdStreamSource);

// Validator library:
String xsd = "xxe.xsd";
String xml = "xxe.xml";
StreamSource xsdStreamSource = new StreamSource(xsd);
StreamSource xmlStreamSource = new StreamSource(xml);

SchemaFactory schemaFactory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
Schema schema = schemaFactory.newSchema(xsdStreamSource);
Validator validator = schema.newValidator();   // Noncompliant

StringWriter writer = new StringWriter();
validator.validate(xmlStreamSource, new StreamResult(writer));

//Dom4j library:
SAXReader xmlReader = new SAXReader(); // Noncompliant by default
Document xmlResponse = xmlReader.read(xml);

// Jdom2 library:
SAXBuilder builder = new SAXBuilder(); // Noncompliant by default
Document document = builder.build(new File(xml));
```

## 解决问题

解决也很简单，就是要修改两个默认配置。在JDK1.7之前，用 **XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES**和**XMLInputFactory.SUPPORT_DTD**,　之后用**XMLConstants.ACCESS_EXTERNAL_DTD** 以及 **XMLConstants.ACCESS_EXTERNAL_SCHEMA**


```java
// DocumentBuilderFactory library:
String xml = "xxe.xml";
DocumentBuilderFactory df = DocumentBuilderFactory.newInstance();
df.setAttribute(XMLConstants.ACCESS_EXTERNAL_DTD, ""); // Compliant
df.setAttribute(XMLConstants.ACCESS_EXTERNAL_SCHEMA, ""); // compliant
DocumentBuilder builder = df.newDocumentBuilder();
Document document = builder.parse(new InputSource(xml));
DOMSource domSource = new DOMSource(document);

// SAXParserFactory library:
String xml = "xxe.xml";
SaxHandler handler = new SaxHandler();
SAXParserFactory factory = SAXParserFactory.newInstance();
SAXParser parser = factory.newSAXParser();
parser.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, ""); // Compliant
parser.setProperty(XMLConstants.ACCESS_EXTERNAL_SCHEMA, ""); // compliant
parser.parse(xml, handler);

// XMLInputFactory library:
XMLInputFactory factory = XMLInputFactory.newInstance();
factory.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, ""); // Compliant
factory.setProperty(XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");  // compliant

XMLEventReader eventReader = factory.createXMLEventReader(new FileReader("xxe.xml"));

// TransformerFactory library:
String xslt = "xxe.xsl";
String xml = "xxe.xml";
TransformerFactory transformerFactory = javax.xml.transform.TransformerFactory.newInstance();
transformerFactory.setAttribute(XMLConstants.ACCESS_EXTERNAL_DTD, ""); // Compliant
transformerFactory.setAttribute(XMLConstants.ACCESS_EXTERNAL_STYLESHEET, ""); // Compliant
// ACCESS_EXTERNAL_SCHEMA not supported in several TransformerFactory implementations
Transformer transformer = transformerFactory.newTransformer(new StreamSource(xslt));

StringWriter writer = new StringWriter();
transformer.transform(new StreamSource(xml), new StreamResult(writer));
String result = writer.toString();

// SchemaFactory library:
String xsd = "xxe.xsd";
StreamSource xsdStreamSource = new StreamSource(xsd);

SchemaFactory schemaFactory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
schemaFactory.setProperty(XMLConstants.ACCESS_EXTERNAL_SCHEMA, ""); // Compliant
schemaFactory.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, ""); // Compliant
Schema schema = schemaFactory.newSchema(xsdStreamSource);

// Validator library:
String xsd = "xxe.xsd";
String xml = "xxe.xml";
StreamSource xsdStreamSource = new StreamSource(xsd);
StreamSource xmlStreamSource = new StreamSource(xml);

SchemaFactory schemaFactory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
Schema schema = schemaFactory.newSchema(xsdStreamSource);
schemaFactory.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, "");
schemaFactory.setProperty(XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");
// validators will also inherit of these properties
Validator validator = schema.newValidator();

validator.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, "");   // Compliant
validator.setProperty(XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");   // Compliant

StringWriter writer = new StringWriter();
validator.validate(xmlStreamSource, new StreamResult(writer));

// For dom4j library, ACCESS_EXTERNAL_DTD and ACCESS_EXTERNAL_SCHEMA are not supported, thus a very strict fix is to disable
// doctype declarations:
SAXReader xmlReader = new SAXReader();
xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true); // Compliant
Document xmlResponse = xmlReader.read(xml);

// Jdom2 library:
SAXBuilder builder = new SAXBuilder(); // Compliant
builder.setProperty(XMLConstants.ACCESS_EXTERNAL_DTD, ""); // Compliant
builder.setProperty(XMLConstants.ACCESS_EXTERNAL_SCHEMA, ""); // Compliant
Document document = builder.build(new File(xml));
```

还有一种说法：

This rule raises an issue when any of the following are used without first disabling external entity processing: javax.xml.validation.Validator, JAXP's DocumentBuilderFactory, SAXParserFactory, Xerces 1 and Xerces 2 StAX's XMLInputFactory and XMLReaderFactory.

To disable external entity processing for XMLInputFactory, configure one of the properties XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES or XMLInputFactory.SUPPORT_DTD to false.

To disable external entity processing for SAXParserFactory, XMLReader or DocumentBuilderFactory configure one of the features XMLConstants.FEATURE_SECURE_PROCESSING or "http://apache.org/xml/features/disallow-doctype-decl" to true.

To disable external entity processing for Validator, configure both properties XMLConstants.ACCESS_EXTERNAL_DTD, XMLConstants.ACCESS_EXTERNAL_SCHEMA to the empty string "".

## 其他库

XMLStants是JDK 1.5(JAXP 1.3)引入的，而上面两个新属性是JDK 1.7(JAXP 1.4.5)引入的。很多老的库不支持，比如以下是项目中的两个例子：

```xml
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>3.15</version>
    <exclusions>
        <exclusion>
            <artifactId>xml-apis</artifactId>
            <groupId>xml-apis</groupId>
        </exclusion>
        <!-- as this old XMLConstants duplicate JDK 1.8 -->
        <exclusion>
            <groupId>stax</groupId>
            <artifactId>stax-api</artifactId>
        </exclusion>                 
    </exclusions>
</dependency>
```

```xml
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jaxb-provider</artifactId>
    <version>2.2.1.GA</version>
    <exclusions>
        <exclusion>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
        </exclusion>
        <!-- as this old XMLConstants duplicate JDK 1.8 -->
        <exclusion>
            <groupId>javax.xml.stream</groupId>
            <artifactId>stax-api</artifactId>
        </exclusion>        		
    </exclusions>
</dependency>
```

## 资源

１．　https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html
２　　https://docs.oracle.com/javase/7/docs/api/javax/xml/parsers/DocumentBuilderFactory.html#setFeature(java.lang.String,%20boolean)

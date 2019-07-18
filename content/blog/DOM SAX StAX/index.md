---
title: DOM SAX StAX
date: "2019-07-18T16:21:00.000Z"
description: "重新梳理一下Java解析XML的三个库"
---

都现在了，我也没有想到自己又回到十年前重新又碰Java解析XML，现在一个老项目中间正好用到，自己也算复习一下。

## DOM

DOM（Document Object Model） 是第一种解析办法，也是最早的一种，基本就是把整个XML文件都读到内存中去构造树结构，然后再进行增删改查。主要的缺点就是对待大的XML文件无能为力。

```java
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import org.w3c.dom.Document;

// ...

DocumentBuilderFactory df;
DocumentBuilder builder;
Document document;

try {
    // Obtain DocumentBuilder factory
    df = DocumentBuilderFactory.newInstance();
    
    // Get DocumentBuilder instance from factory
    builder = df.newDocumentBuilder();
    
    // Document object instance now is the in-memory representation of the XML file
    document = builder.parse("src/students.xml");
} catch (Exception e) {
    e.printStackTrace();
}
```

这种解析办法已经包含在JDK/JAXP中了。　另外还有[JDOM](http://www.jdom.org) 和 [DOM4J](http://www.dom4j.org/)

## SAX

SAX(Simple API for XML)是边解析边推送，用户可以根据预先定义的callback来对推送的解析内容进行处理，优点就是内存不是问题了。但是由于它使用的推模式，也就是我们无法控制解析过程，给我们什么我们就用什么，无法暂停，也无法做些特殊处理。

```java
import java.util.ArrayList;

import javax.xml.parsers.SAXParser;
import javax.xml.parsers.SAXParserFactory;

import org.xml.sax.XMLReader;

public class SAXDemo {
    public static void main(String[] args) {
        ArrayList<BookBean> bookList = null;
        BookHandler bookHandler = new BookHandler();
        
        SAXParserFactory saxParserFactory = SAXParserFactory.newInstance();
        SAXParser saxParser;
        
        try {
            saxParser = saxParserFactory.newSAXParser();
            
            XMLReader xmlReader = saxParser.getXMLReader();
            xmlReader.setContentHandler(bookHandler);
            xmlReader.parse("src/Books.xml");
            
            /* or */
            // saxParser.parse("src/Books.xml", bookHandler);
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        bookList = bookHandler.getBookList();
        
        if (bookList != null) {
            for (BookBean book : bookList) {
                System.out.println(book);
            }
        }
    }
}
```

Handler就是callback，包含三个方法，需要仔细定义。比如：

```java
import java.util.ArrayList;

import org.xml.sax.Attributes;
import org.xml.sax.SAXException;
import org.xml.sax.helpers.DefaultHandler;

public class BookHandler extends DefaultHandler {

    private String mCurrentTagName;
    private BookBean mBook;
    
    private ArrayList<BookBean> mBookList = new ArrayList<BookBean>();
    
    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
        
        // Remember the current element tag
        this.mCurrentTagName = qName;
        
        // If current tag is a new book element item, create a new BookBean object
        if ("book".equals(this.mCurrentTagName)) {
            this.mBook = new BookBean();
            this.mBook.setISBN(attributes.getValue("ISBN"));
        }
    }
    
    @Override
    public void characters(char[] ch, int start, int length) throws SAXException {
    
        if ("book".equals(this.mCurrentTagName)) {
            String name = new String(ch, start, length);
            this.mBook.setName(name);
        }
        
    }
    
    @Override
    public void endElement(String uri, String localName, String qName) throws SAXException {
    
        // If parsing of a book item is finished, add it to the list and reset mBook
        if ("book".equals(qName)) {
            this.mBookList.add(this.mBook);
            this.mBook = null;
        }
        
        // Reset current element tag
        this.mCurrentTagName = null;
    }
    
    public ArrayList<BookBean> getBookList() {
        return this.mBookList;
    }
}
```


这种解析办法也已经包含在JDK/JAXP中了。　另外还有[Xerces](https://xerces.apache.org/)

## StAX

这个是后来出现的解析办法，采用的是拉模式，这样就会比较灵活，另外由于和SAX一样采用流处理，所需内存也不大。

```java
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.util.Iterator;

import javax.xml.namespace.QName;
import javax.xml.stream.XMLEventReader;
import javax.xml.stream.XMLInputFactory;
import javax.xml.stream.XMLStreamException;
import javax.xml.stream.events.Attribute;
import javax.xml.stream.events.Characters;
import javax.xml.stream.events.EndElement;
import javax.xml.stream.events.StartElement;
import javax.xml.stream.events.XMLEvent;

public class StAXEventReaderExample {
    public static void main(String[] args) throws XMLStreamException,
            FileNotFoundException {
        XMLInputFactory factory = XMLInputFactory.newInstance();
        XMLEventReader reader = factory.createXMLEventReader("sample.xml",
                new FileInputStream("src/com/javarticles/jaxp/sample.xml"));

        while (reader.hasNext()) {
            XMLEvent event = reader.nextEvent();
            if (event.isStartElement()) {
                StartElement element = (StartElement) event;
                System.out.println("Start Element: " + element.getName());

                Iterator iterator = element.getAttributes();
                while (iterator.hasNext()) {
                    Attribute attribute = (Attribute) iterator.next();
                    QName name = attribute.getName();
                    String value = attribute.getValue();
                    System.out.println("Attribute name/value: " + name + "/"
                            + value);
                }
            }
            if (event.isEndElement()) {
                EndElement element = (EndElement) event;
                System.out.println("End element:" + element.getName());
            }
            if (event.isCharacters()) {
                Characters characters = (Characters) event;
                System.out.println("Text:[" + characters.getData() + "]");
            }
        }
    }
}
```

这种解析办法是现在Java主流的解析办法，也包含在JDK中了 --- [JSR-173](https://jcp.org/en/jsr/detail?id=173)。

## DOM vs SAX vs StAX

|                              |                 |                 |                |           | 
|------------------------------|-----------------|-----------------|----------------|-----------| 
| Feature                      | StAX            | SAX             | DOM            | TrAX      | 
| API Type                     | Pull, streaming | Push, streaming | In memory tree | XSLT Rule | 
| Ease of Use                  | High            | Medium          | High           | Medium    | 
| XPath Capability             | No              | No              | Yes            | Yes       | 
| CPU and Memory Efficiency    | Good            | Good            | Varies         | Varies    | 
| Forward Only                 | Yes             | Yes             | No             | No        | 
| Read XML                     | Yes             | Yes             | Yes            | Yes       | 
| Write XML                    | Yes             | No              | Yes            | Yes       | 
| Create, Read, Update, Delete | No              | No              | Yes            | No        | 

## JAXP vs JAXB

JAXB 要比JAXP 更抽象一点，更高一点。它不仅提供了Java对象和xml对象的解析，也提供了绑定，这样就可以不局限在底端解析的细节上了。从这个意义上来说。JAXP 已经过时。

## 今天的情况

今天是项目中用到了一个老的SAX库 ---- Xerces 1.2.3，导致每次解析到80,000条记录时就报错

```
Caused by: java.lang.RuntimeException: Internal Error: fPreviousChunk == NULL
	at org.apache.xerces.utils.UTF8DataChunk.addSymbol(UTF8DataChunk.java:389)
	at org.apache.xerces.readers.UTF8Reader.addSymbol(UTF8Reader.java:124)
	at org.apache.xerces.framework.XMLDocumentScanner$ContentDispatcher.dispatch(XMLDocumentScanner.java:1315)
	at org.apache.xerces.framework.XMLDocumentScanner.parseSome(XMLDocumentScanner.java:381)
	at org.apache.xerces.framework.XMLParser.parse(XMLParser.java:948)
	... 31 more
	
```

升级到1.4.4也不行，干脆把这个依赖去掉，也就是使用JDK自带的SAX库，就解决问题了。

## 感谢

* https://www.codevoila.com/post/62/xml-processing-in-java-jaxp-dom-example

* https://medium.com/@assertis/processing-large-xml-files-fa23b271e06d

* https://www.codevoila.com/post/63/xml-processing-in-java-jaxp-sax-example

* https://docs.oracle.com/cd/E17802_01/webservices/webservices/docs/1.6/tutorial/doc/SJSXP2.html




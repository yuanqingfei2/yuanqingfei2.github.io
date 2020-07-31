---
title: XJC task in Maven
date: "2020-07-30T17:07:00.000Z"
description: "XJC task in Maven"
---

谁也没有想到XSD的生命周期会这么长，即便在如今json大行天下的现在估计一时半会也不会死掉。近日在用Maven的插件生成代码的时候遇到两个需求，记录如下：

## 使生成的Java代码要实现Serilizable接口

这个需求源于Sonar代码检查，我的模型类的最上面是扩展了Serilizable接口的，而在具体的模型类上，有的field直接使用了XSD生成的Java类，这样好处是将来即使XSD有更新，Java模型类不需要改变，可是Sonar认为所有实现了Serilizable接口的类的field都应该也是实现Serilizable接口。 

[原文](https://rules.sonarsource.com/java/RSPEC-1948)如下：
>Fields in a Serializable class must themselves be either Serializable or transient even if the class is never explicitly serialized or deserialized. For instance, under load, most J2EE application frameworks flush objects to disk, and an allegedly Serializable object with non-transient, non-serializable data members could cause program crashes, and open the door to attackers. In general a Serializable class is expected to fulfil its contract and not have an unexpected behaviour when an instance is serialized.

实现方法很简单，就是使用XJB文件里面的global binding，这样生成代码的时候就可以客户化定制了。

XJB文件:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<jxb:bindings 
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:jxb="http://java.sun.com/xml/ns/jaxb"
    jxb:version="1.0">
	<jxb:globalBindings>
		<jxb:serializable uid="1" />
	</jxb:globalBindings>
</jxb:bindings>
```

Maven文件：

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>jaxb2-maven-plugin</artifactId>
    <version>2.5.0</version>
    <executions>
       <execution>
            <id>xsd-v1</id>
            <goals>
                <goal>xjc</goal>
            </goals>
            <configuration>
                <sources>
                    <source>src/main/xsd/xxx/xxx-input-file-v4.2.xsd</source>
                </sources>
                <xjbSources>
                    <xjbSource>src/main/xsd/xxx/xxx.xjb</xjbSource>
                </xjbSources>
                <packageName>com.xxx.xxx.xml.v1_0</packageName>
                <clearOutputDir>false</clearOutputDir>
            </configuration>
        </execution> 
    </executions>
</plugin>    
```

## 使生成的Java代码不用内部类

有些XSD写的时候就不是正常那种定义类型的方式，而是嵌套的写法，比如下面这种：

```xml
<xs:schema attributeFormDefault="unqualified"
	elementFormDefault="qualified" xmlns:xs="http://www.w3.org/2001/XMLSchema">
	<xs:element name="CollateralDefinitions">
		<xs:complexType>
			<xs:sequence>
				<xs:element name="CollateralDefinition" maxOccurs="unbounded"
					minOccurs="0">
					<xs:complexType>
						<xs:sequence>
							<xs:element type="xs:short" name="CollateralTypeID" />>
							<xs:element name="CollateralTypeCurriencies">
								<xs:complexType mixed="true">
									<xs:sequence>
										<xs:element name="CollateralTypeCurrency"
											maxOccurs="unbounded" minOccurs="0">
											<xs:complexType>
												<xs:sequence>
													<xs:element type="xs:string" name="CurrencyCode" />
													<xs:element type="xs:string" name="IncludeExclude" />
												</xs:sequence>
											</xs:complexType>
										</xs:element>
									</xs:sequence>
								</xs:complexType>
							</xs:element>
                        </xs:sequence>
                    </xs:complexType>  
                </xs:element>          
		    </xs:sequence>
		</xs:complexType>
	</xs:element>
</xs:schema>
```

你可以看到它所有的类型定义都是在element里面，如果默认情况下生成的代码是下面这种都在一个Java文件里面的嵌套类的。可是如果你需要所有的类型都在自己单独的类里面而不是嵌套类，可以在XJB中如下配置：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<jaxb:bindings
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns:jaxb="http://java.sun.com/xml/ns/jaxb"
    xmlns:namespace="http://jaxb2-commons.dev.java.net/namespace-prefix"
    version="2.1">
    <jaxb:globalBindings localScoping="toplevel"/>
</jaxb:bindings>
```

这样生成出来的就都是top类而不是内部类了。这种情况有时是需要的，比如我需要利用生成的Java类来做Spring支持的MongoDB模型类，如果是嵌套类，就会抛出不支持类作为field的错误。

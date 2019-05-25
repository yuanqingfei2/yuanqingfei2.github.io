---
title: 如何提高单元测试覆盖率
date: "2019-05-23T11:08:00.000Z"
description: "根据自己的经验给出几条建议"
---

最近把项目打扫了下，代码行数从18万减少到14万，代码覆盖率从65%提高到75%，获得一些体会，分享如下：

## 打扫

在增加测试前呢，一定要先打扫一下，就如同你想添置些新家具，肯定要把房子先打扫一下，归置一下一样。

* 不要把生成代码放在代码库里
这条其实很重要，在目前的开发大环境下，很多代码生成的工具用来生成辅助代码，时不时就会混入代码库，一呆就是永远。搜索"Generated"，你就会发现有多少这样的代码了。我们去除了下面两种：

1. [JAXB](https://www.oracle.com/technetwork/articles/javase/index-140168.html)根据xsd生成的Java代码

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>jaxb2-maven-plugin</artifactId>
    <version>2.4</version>
    <executions>
        <execution>
            <id>xxx-xsd-v2</id>
            <goals>
                <goal>xjc</goal>
            </goals>
            <configuration>
                <sources>
                    <source>src/main/xsd/xxx/v2</source>
                </sources>
                <packageName>com.xxx.v2_0</packageName>
                <clearOutputDir>false</clearOutputDir>
            </configuration>
        </execution>
        <execution>
            <id>xxy-xsd-v1</id>
            <goals>
                <goal>xjc</goal>
            </goals>
            <configuration>
                <sources>
                    <source>src/main/xsd/xxy/XXY_SCHEMA.xsd</source>
                </sources>
                <packageName>com.xxy.v1_0</packageName>
                <clearOutputDir>false</clearOutputDir>
            </configuration>
        </execution>                 
    </executions> 
</plugin>    
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <version>1.7</version>
    <executions>
        <execution>
            <id>add-source</id>
            <phase>generate-source</phase>
            <goals>
                <goal>add-source</goal>
            </goals>
            <configuration>
                <sources>
                    <source>${project.build.directory}/generated-sources/jaxb/com/xxx/v2_0</source>
                    <source>${project.build.directory}/generated-sources/jaxb/com/xxy/v1_0</source>
                </sources>
            </configuration>
        </execution>
    </executions>
</plugin>                  
```

这样，你每次都时生成最新的Java代码并使用，可以放心删除掉代码库里的了。

2. [AVRO](https://avro.apache.org/)

```xml
<plugin>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro-maven-plugin</artifactId>
    <version>1.8.2</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>schema</goal>
                <goal>idl-protocol</goal>
            </goals>
            <configuration>
                <sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory>
                <outputDirectory>${project.basedir}/target/generated-source/avro</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <version>1.7</version>
    <executions>
        <execution>
            <id>add-source</id>
            <phase>generate-source</phase>
            <goals>
                <goal>add-source</goal>
            </goals>
            <configuration>
                <sources>
                    <source>${project.build.directory}/generated-sources/avro/com/citi/icva/tmintegrateion/avro</source>
                    <source>${project.build.directory}/generated-sources/avro/com/citi/icva/quattro/event/service/avro</source>
                </sources>
            </configuration>
        </execution>
    </executions>
</plugin>             
```
通过以上两个plugin就可以了。

* 删除掉没用代码

有很多办法可以删除，我是通过一个叫[UCDetector](http://www.ucdetector.org/)的工具来做的。它有很方便的[Eclipse插件](http://ucdetector.sourceforge.net/update)。 原理很简单，通过对所有方法的尝试找到所有没有被用到过的方法，属性以及类。你可以通过它给出的列表来逐一检查是否真的应该删掉，只所以还要人工介入，主要原因是有些时候它不知道一个方法是否真的有用，比如你在Spring Context通过一个方法定义了一个bean，这个方法当然不会在编译器显式调用，但是这个方法在运行时肯定有用的，所以你不能删掉。

```csv
Description	Resource	Path	Location	Type
Method "AbstractDsp4RdsDao.loadAll()" has 0 references	AbstractDsp4RdsDao.java	/xxx/quattro/dao	line 237	UCDetector Marker - References
Method "AbstractDsp4RdsDao.getStringFieldClause(String,String)" has 0 references	AbstractDsp4RdsDao.java	/xxx/quattro/dao	line 149	UCDetector Marker - References
Constant "AbstractDsp4RdsDao.LESS_THAN" has 0 references	AbstractDsp4RdsDao.java	/xxx/quattro/dao	line 37	UCDetector Marker - References
Constant "AbstractDsp4RdsDao.GREATER_THAN" has 0 references	AbstractDsp4RdsDao.java	/xxx/quattro/dao	line 38	UCDetector Marker - References
Field "AbstractFacItemWriter.rapidDateTimeFmt" has 0 references	AbstractFacItemWriter.java	/xxx/writers	line 29	UCDetector Marker - References
Field "AbstractFXLMTradeReader.firmAccountLEMap" has 0 references	AbstractFXLMTradeReader.java	/xxx/reader	line 32	UCDetector Marker - References
Method "AbstractMongoDao.getDB()" has 0 references	AbstractMongoDao.java	/xxx/util/mongoDbDao	line 48	UCDetector Marker - References
Constant "AbstractMongoDao.UPDATED_EXISTING_STATUS" has 0 references	AbstractMongoDao.java	/xxx/util/mongoDbDao	line 29	UCDetector Marker - References
Class "AbstractRapidTask" has 0 references	AbstractRapidTask.java	/xxx/tasks	line 13	UCDetector Marker - References
```

## 写测试

* 发现漏洞

可以使用[EclEmma](https://www.eclemma.org/)方便的知道哪些代码没有被覆盖，可以针对性的写测试。

* 单元测试

关于Spring环境下的测试，请参考之前写的这篇[总结](https://www.yuanqingfei.com/Spring%20Test/)。

* 集成测试

有时候如果你需要些很多类似的测试代码的时候，你可以借助一个库来帮你写代码。就是[javapoet](https://github.com/square/javapoet)

```java

public class QuattroIntegrateTestGenerator {
	
	public static void main(String args[]) throws IOException {
		Map<Class, String[]> m = new HashMap<>();
		m.put(ManagedSegmentHierarchyLoaderContext.class, new String[] {"date=20190402"});
		m.put(EdealerFxBranchFeedLoaderContext.class, new String[]{"date = 20180524","XXX_HOME=./target/test-classes/"});
		m.put(EdealerTradeConsolidateLoaderContext.class, new String[]{"date = 20180524","XXX_HOME=./target/test-classes/","stage=CUSTOMER"});

        // you can add more class in above map.
		
		for(Class c : m.keySet()) {
			Generator g = new Generator(c, m.get(c));
	 		g.generate();
		}	
		System.out.println("Generate Successfully!");
	}
}

class Generator{
	private Class t;
	private String[] p;
	
	public Generator(Class t, String[] p) {
		this.t = t;
		this.p = p;
	}
	
	public void generate() throws IOException {
		
		MethodSpec testJob = MethodSpec.methodBuilder("testJob")
				.addModifiers(Modifier.PUBLIC)
				.addAnnotation(Test.class)
				.returns(void.class)
				.beginControlFlow("try")
				.addStatement("$T execution = jobLauncher.run(job, new $T())", JobExecution.class, JobParameters.class)
				.addStatement("$T.assertEquals($T.COMPLETED, execution.getStatus())", Assert.class, BatchStatus.class)
				.nextControlFlow("catch ($T e)", Exception.class)
				.addStatement("Assert.fail()")
				.endControlFlow()
				.build();
		
		FieldSpec jobLauncher = FieldSpec.builder(JobLauncher.class, "jobLauncher")
				.addModifiers(Modifier.PRIVATE)
				.addAnnotation(Autowired.class)
				.build();
		
		FieldSpec job = FieldSpec.builder(Job.class, "job")
				.addModifiers(Modifier.PRIVATE)
				.addAnnotation(Autowired.class)
				.build();
		
		AnnotationSpec runnerAnotation = AnnotationSpec.builder(RunWith.class)
				.addMember("value", "$T$L", SpringJUnit4ClassRunner.class, ".class")
				.build();
		
		AnnotationSpec contextAnnotation = AnnotationSpec.builder(ContextConfiguration.class)
				.addMember("classes", "$T$L", t, ".class")
				.build();
		
		AnnotationSpec propertyAnnotation = AnnotationSpec.builder(TestPropertySource.class)
				.addMember("properties", getFormat(p), getObjects(p))
				.build();		
		
		TypeSpec testType = TypeSpec.classBuilder(t.getSimpleName() + "Test")
				.addModifiers(Modifier.PUBLIC)
				.addAnnotation(runnerAnotation)
				.addAnnotation(contextAnnotation)
				.addAnnotation(propertyAnnotation)
				.addField(jobLauncher)
				.addField(job)
				.addMethod(testJob)
				.build();
		
		JavaFile javaFile = JavaFile.builder("com.xxx.integrate", testType)
				.build();
		
		File dest = new File("src/test/java/");
		javaFile.writeTo(dest);
//		javaFile.writeTo(System.out);
	}
	
	private String getFormat(String[] p) {
		StringBuilder sb = new StringBuilder("$L");
		for(int i = 0; i < p.length; i ++) {
			sb.append("$L,");
		}
		String result = sb.toString();
		if(result.endsWith(",")) {
			result = result.substring(0, result.length() - 1);	
		}
		result = result + "$L";
//		System.out.println(result);
		return result;
	}
	
	private String[] getObjects(String[] p) {
		String[] result = new String[p.length + 2];
		result[0] = "{";
		result[p.length + 1] = "}";
		for(int i = 1; i <= p.length; i ++) {
			result[i] = "\"" + p[i -1] + "\"";
		}
//		System.out.println(result);
		return result;
	}
}
```

## Code Smell

最后，当你覆盖率也满意之后，最好还是能侦测下代码的质量，也就是看看是否有不好的气味。可以使用[SonarLint](https://www.sonarlint.org/eclipse/)插件。

## 致谢

* https://stackoverflow.com/questions/35242941/jaxb-maven-plugin-not-generating-classes


---
title: Spring Test
date: "2019-05-06T10:22:00.000Z"
description: "Spring Test 中常用的几个Annotation以及如何应对@PostConstruct"
---

Spring有[spring-test](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html), 加之[Mockito](https://site.mockito.org/)和[Junit](https://junit.org/junit4/)，使得Test变得容易。

1. @RunWith(SpringJUnit4ClassRunner.class) 和 @ContextConfiguration(locations = {...})
------------
使用这个组合来加载springcontext并进行测试。如果你使用的是spring-boot，可以使用 @RunWith(SpringRunner.class)和@SpringBootTest(SpringBootTest.WebEnvironment.None, classes = XXX.class)。 效果是一样的。

2. @MockBean
------------
它的含义是为你创建一个空的Bean，**同时**替换在正式的springcontext中的bean。可以随后基于这个空的bean进行verify。

3. @SpyBean
----------
它的含义是为你创建一个几乎真实的bean，但是你可以**替换**bean里面的方法。同样随后可以基于此进行verify。

4. @Autowired
----------
这个没啥说的，就是你要测试的那个bean。如果有多个同样类型的，使用@Qulifier(xxx)来区分。

5. 例子
--------
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = RapidFacilityLoaderContext.class)
public class RapidFacItemWriterTest {

	@MockBean
	private MongoOperations mockProvider;
	
	@Autowired
	private RapidFacItemWriter rapidWriter;
	
	@Test
	public void testWriteAdd() throws Exception {
		Assert.assertNotNull(rapidWriter);
		
		List<RpdFacility> toWrite = new ArrayList<>();
		RpdFacility rf = createRf();
		
		toWrite.add(rf);
		rf.setActionCode(RpdActionCodeE.ADD);
		rapidWriter.write(toWrite);
		
		verify(mockProvider, times(1)).find(any(Query.class), eq(Facility.class));
		verify(mockProvider, times(1)).updateMulti(any(Query.class), any(Update.class), eq(Facility.class));
		verify(mockProvider, times(1)).insertAll(any(List.class));
	}
```

6. 问题来了，如果你要测试的bean使用了@PostConstruct该怎么办
------------------
我们知道@PostConstruct的方法在Springcontext中是加载完构造体就要执行的，如果这个方法里面的依赖需要mock怎么办，你可能会说，使用@Before阿，可是并没有用，因为@before执行代码在@PostConstruct之后。 方法是基于原有的springcontext创建测试context，在所依赖的bean里mock出想要的行为。如果你的名字不同（下面例子中的altIdDao），就要使用@Primary来区分。如果名字相同，会自动override。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes= {AwesomeTestConfig.class})
@TestPropertySource(properties = {"date = 20180524","propertiesFile=classpath:xxx.properties","./data/input/cds=.","CVA_HOME=.","fileName=emptyfile"})
public class CurveResolutionOrganizationTest {

       @Autowired
       private CreditRatingAltIdDao altIdDao;
       
       @Autowired
       @Qualifier("orgCreditAttributesProcessor")
       CreditAttributesProcessor processor;

       @Test
       public void test() {
              ...
              processor.getCreditRatingAltIdDao();
              Assert.assertTrue(processor.getClassificationOverrideMap()==map);
       }

       @Configuration
       @Import(CurveResolutionOrganizationLoaderContext.class)
       static class AwesomeTestConfig{
              
              @Bean
              @Primary
              public CreditRatingAltIdDao altIdDao() {
                     CreditRatingAltIdDao altIdDao = Mockito.mock(CreditRatingAltIdDao.class);
                     CreditRatingAltId altId = new CreditRatingAltId();
                     when(altIdDao.findAltIdByAgencyAndRating(isA(AgencyE.class), anyString())).thenReturn(altId);
                     return altIdDao;
              }
       }
}

```

7. 致谢
--------
https://www.baeldung.com/spring-boot-testing
https://www.baeldung.com/java-spring-mockito-mock-mockbean
https://stackoverflow.com/questions/31587639/testing-spring-bean-with-post-construct?noredirect=1&lq=1

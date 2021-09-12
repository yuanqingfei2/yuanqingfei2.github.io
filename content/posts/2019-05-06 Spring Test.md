---
title: Spring Test
date: "2019-05-06T10:22:00.000Z"
description: "Spring Test 中常用的几个Annotation以及如何应对@PostConstruct"
---

Spring有[spring-test](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html), 加之[Mockito](https://site.mockito.org/)和[Junit](https://junit.org/junit4/)，使得Test变得容易。

## @RunWith(SpringJUnit4ClassRunner.class) 和 @ContextConfiguration(locations = {...})

使用这个组合来加载springcontext并进行测试。如果你使用的是spring-boot，可以使用 @RunWith(SpringRunner.class)和@SpringBootTest(SpringBootTest.WebEnvironment.None, classes = XXX.class)。 效果是一样的。

## @MockBean

它的含义是为你创建一个空的Bean，**同时**替换在正式的springcontext中的bean。可以随后基于这个空的bean进行verify。

## @SpyBean

它的含义是为你创建一个几乎真实的bean，但是你可以**替换**bean里面的方法。同样随后可以基于此进行verify。

## @Autowired

这个没啥说的，就是你要测试的那个bean。如果有多个同样类型的，使用@Qulifier(xxx)来区分。

## 例子

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
}	
```

## 如何测试@PostConstruct的bean

我们知道@PostConstruct的方法在Springcontext中是加载完构造体就要执行的，如果这个方法里面的依赖需要mock怎么办，你可能会说，使用@Before阿，可是并没有用，因为@Before后运行。 方法是基于原有的springcontext创建测试context，在所依赖的bean里mock出想要的行为。如果你的名字不同（下面例子中的altIdDao），就要使用@Primary来区分。如果名字相同，会自动override。

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

## 如何测试@StepScope的bean

```java
	@Bean
	@StepScope
	public ItemWriter<DirectParents> itemWriter(@Value("#{jobParameters[cobDate]}") Date cobDate) {
		DirectParentsWriter writer = new DirectParentsWriter();
		writer.setCobDate(cobDate);
		return writer;
	}
```
如果想测试上面的Writer，根据[spring-test](https://docs.spring.io/spring-batch/trunk/reference/html/testing.html#testingIndividualSteps), 你需要@TestExecutionListeners({DependencyInjectionTestExecutionListener.class, StepScopeTestExecutionListener.class})，可是如果这样的话，你原来的@MockBean反而不工作了，解决办法就是显式引入Mockito的Listener。如下所示

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = DirectParentTestConfig.class)
@TestExecutionListeners({DependencyInjectionTestExecutionListener.class, StepScopeTestExecutionListener.class, MockitoTestExecutionListener.class})
public class DirectParentsWriterTest {
	
	
	@MockBean
	private MongoOperations mongo;
	
	@Autowired
	private ItemWriter<DirectParents> writer;
	
	@Test
	public void testWriter() {
		assertNotNull(writer);
		assertNotNull(mongo);
		
		List<DirectParents> items = new ArrayList<>();
		DirectParents dp = new DirectParents();
		dp.setGfcId("gfcId");
		dp.setDirectParentGfcId("parentGfcId");
		items.add(dp);
		try {
			writer.write(items);
			verify(mongo, times(1)).insertAll(any(Collection.class));
		} catch (Exception e) {
			e.printStackTrace();
			fail();
		}
	}
	
	public StepExecution getStepExecution() {
		JobParameter datePara = new JobParameter(new Date());
		
//		StepExecution execution = MetaDataInstanceFactory.createStepExecution();
		
		 // doesn't work
//		execution.getJobParameters().getParameters().put("cobDate", datePara);
		// doesn't work
//		execution.getJobExecution().getJobParameters().getParameters().put("cobDate", datePara);  
		
		Map<String,JobParameter> parameters = new HashMap<>();
		parameters.put("cobDate", datePara);
		JobParameters jobParas = new JobParameters(parameters);
		
		JobExecution jobExec = MetaDataInstanceFactory.createJobExecution("job", 333L, 555L, jobParas);
		StepExecution execution = jobExec.createStepExecution("stepName");
		return execution;
	}
	
	@Configuration
	@Import(DirectParentContext.class)
	static class DirectParentTestConfig{
		@Bean 
		public DataSource dataSource() {
			return null;
		}
	}
}
```

关于设置JobParameter有个坑，如果你从execution里面拿Map放参数进去，你永远不会成功，因为它给你是全新的Map。也就是说这个Map只读，不可写。

org.springframework.batch.core.JobParameters.java  

```java
	/**
	 * Get a map of all parameters, including string, long, and date.
	 * 
	 * @return an unmodifiable map containing all parameters.
	 */
	public Map<String, JobParameter> getParameters(){
		return new LinkedHashMap<String, JobParameter>(parameters);
	}
```

## 致谢

https://www.baeldung.com/spring-boot-testing 

https://www.baeldung.com/java-spring-mockito-mock-mockbean 

https://stackoverflow.com/questions/31587639/testing-spring-bean-with-post-construct?noredirect=1&lq=1

https://docs.spring.io/spring-batch/trunk/reference/html/testing.html#d5e3531

https://github.com/spring-projects/spring-boot/issues/9609

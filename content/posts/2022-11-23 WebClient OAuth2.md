---
title: Spring Web Client and OAuth2
date: "2022-11-23T17:32:00.000Z"
description: "最近项目中的一个有价值的东西"
---

项目中，需要调用远程的Rest服务，而且必须通过OAuth2验证，这个过程经历三个演变，一开始就最原始的拿到Token再调用，这种方式最大的缺点是一旦token失效，需要重新拿Token. 
到后来的用RestTemplate, 这种方式的缺点是需要使用一个很丑陋的Interceptor，最后使用WebClient, 代码比较简洁. 先把最后WebClient代码发一下.

```java
public class IsgCloudRestUtil {
	
	@Value("${isgCloud.httpAuthHost}")
	private String httpAuthHost;
	
	@Value("${isgCloud.httpApiHost}")
	private String httpApiHost;
		
	@Value("${isgCloud.authUserName}")
	private String authUserName;
	
	@Value("${isgCloud.authPassword}")
	private String authPassword;

	// use static since WebClient is thread-safe
	private static WebClient webClient;
	
	private static final Logger logger = LoggerFactory.getLogger(IsgCloudRestUtil.class);

	public List<HashMap<?, ?>> callIsgCloudApi(String isgPath, String search, String projection) {
		UriComponentsBuilder builder = UriComponentsBuilder.newInstance()			
			.scheme("https")
			.host(httpApiHost)
			.path(isgPath)
			.queryParam("search", search)
			.queryParam("projection", projection);

		logger.info("url: " + builder.build().toUriString());
		
		// call pagable ISG Cloud API
		Mono<List<HashMap<?, ?>>> result = fetchItems(builder.build().toUri()).expand(response -> {
			if (response.getPaging() == null) {
	            return Mono.empty();
	        }
			String nextUrl = response.getPaging().getPageNext();
	        return fetchItems(URI.create(nextUrl));
		}).flatMap(response -> Flux.fromIterable(response.getResults())).collectList();
		
		return result.block();
	}
	
	private Mono<IsgCloudResponse> fetchItems(URI url){
		return webClient().get().uri(url)
				.accept(MediaType.APPLICATION_JSON)
				.attributes(clientRegistrationId("xxx-cloud"))
				.retrieve()
				.bodyToMono(IsgCloudResponse.class);
	}
	
	private WebClient webClient() {
		if (webClient == null) {
			ServletOAuth2AuthorizedClientExchangeFilterFunction filter;
			try {
				final KeyStore trustStore = createKeyStore(jksFileName, jksPassword);
				final TrustManagerFactory trustManager = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
	            trustManager.init(trustStore);
				final SslContext sslContext = SslContextBuilder.forClient().trustManager(trustManager).build();
				final HttpClient httpClient = HttpClient.create().secure(ssl -> {
	                ssl.sslContext(sslContext);
	            });
				
				filter = new ServletOAuth2AuthorizedClientExchangeFilterFunction(
						isgCloudOauthAuthorizedClientManager(isgCloudOauthClientRegistration()));
				webClient = WebClient.builder()
						.clientConnector(new ReactorClientHttpConnector(httpClient))
						.apply(filter.oauth2Configuration())
						.build();
			} catch (KeyManagementException | NoSuchAlgorithmException | KeyStoreException | CertificateException | IOException e) {
				e.printStackTrace();
				logger.error("failed to get web client", e);
			}
		}
		return webClient;
	}
	
    private ClientRegistration isgCloudOauthClientRegistration() {
        String authTokenEndpoint = UriComponentsBuilder.newInstance()
                .scheme("https")
                .host(httpAuthHost)
                .path("auth/token")
                .build()
                .toUriString();

        return ClientRegistration.withRegistrationId("xxx-cloud")
                .clientId("cloud-api")
                .authorizationGrantType(AuthorizationGrantType.PASSWORD)
                .tokenUri(authTokenEndpoint)
                .build();
    }
    
    private AuthorizedClientServiceOAuth2AuthorizedClientManager isgCloudOauthAuthorizedClientManager(ClientRegistration isgCloudOauthClientRegistration) 
    		throws KeyManagementException, NoSuchAlgorithmException, KeyStoreException, CertificateException, IOException {
        InMemoryClientRegistrationRepository clientRegistrationRepository = new InMemoryClientRegistrationRepository(isgCloudOauthClientRegistration);
        InMemoryOAuth2AuthorizedClientService authorizedClientService = new InMemoryOAuth2AuthorizedClientService(clientRegistrationRepository);
        OAuth2AuthorizedClientProvider oAuth2AuthorizedClientProvider = OAuth2AuthorizedClientProviderBuilder
                .builder()
                .password()
                .refreshToken()
                .build();
        AuthorizedClientServiceOAuth2AuthorizedClientManager authorizedClientManager = 
        		new AuthorizedClientServiceOAuth2AuthorizedClientManager(clientRegistrationRepository, authorizedClientService);
        authorizedClientManager.setAuthorizedClientProvider(oAuth2AuthorizedClientProvider);
        authorizedClientManager.setContextAttributesMapper(
        		oAuth2AuthorizeRequest -> ImmutableMap.of(OAuth2AuthorizationContext.USERNAME_ATTRIBUTE_NAME, authUserName, 
            OAuth2AuthorizationContext.PASSWORD_ATTRIBUTE_NAME, authPassword));
        return authorizedClientManager;
    }    
}
```

本质上和RestTemplate一样构建一个manager和一个registion. WebClient的优点就是它通过使用registrationId`.attributes(clientRegistrationId("xxx-cloud"))`来自动获取token信息。
免去了手工注入的麻烦。

这段代码还有个一个值得一说的功能：调用分页服务，本质上就是通过`expand`方法来递归调用，最后把结果一起发给调用者。

最后一个就是通过Web Client的`clientConnector`使得它具备了可以具备SSL的功能。

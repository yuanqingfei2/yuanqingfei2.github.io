---
title: Spring Web Client and OAuth2
date: "2022-11-23T17:32:00.000Z"
description: "最近项目中的一个有价值的东西"
---

项目中，需要调用远程的Rest服务，而且必须通过OAuth2验证，这个过程经历三个演变，一开始就最原始的拿到Token再调用，这种方式最大的缺点是一旦token失效，需要重新拿Token。到后来的用RestTemplate, 这种方式的缺点是需要使用一个很丑陋的Interceptor，最后使用WebClient, 代码比较简洁. 先把最后WebClient代码发一下.

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

    private static KeyStore createKeyStore(final String keyStoreLocation, final String keyStorePassword) 
    		throws KeyStoreException, NoSuchAlgorithmException, CertificateException, IOException {
        FileInputStream fis = new FileInputStream(keyStoreLocation);
        final KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
        ks.load(fis, keyStorePassword.toCharArray());
        return ks;
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

本质上和RestTemplate一样构建一个manager和一个registion. WebClient的优点就是它通过使用registrationId`.attributes(clientRegistrationId("xxx-cloud"))`来自动获取token信息。另外还可以通过对`ServletOAuth2AuthorizedClientExchangeFilterFunction`的`setDefaultClientRegistrationId()`，都是可以达到一样的效果。


这段代码还有个一个值得一说的功能：调用分页服务，本质上就是通过`expand`方法来递归调用，最后把结果一起发给调用者。

最后一个就是通过Web Client的`clientConnector`使得它具备了可以具备SSL的功能。但是要注意这个SSL只是保证了对resource资源的加密访问，在OAuth2的第一步，如果第一步auth网站也是加密的，那么就必须也要启用SSL功能。假定这两个网站都是使用同样的trustStore，那么有两个方法，第一个方法是全局性的，就是通过给JVM传递参数`-Djavax.net.ssl.trustStore=...`，中间调试的时候也可以通过`-Djavax.net.debug=ssl:handshake`来观察log。另外一个方法是通过在代码中来分别设置，这个方式更灵活，可以override。

下面的三个方法，第一个方法提供了一个配置了SSL信息的HttpClient（注意里面的`keyMananger()`是没有用的，只有`trustManager()`是必须要配置的）。它可以被配置到访问资源和访问Auth的两个网站的WebClient之中。第二个方法所创建的WebClient就是用来访问资源网站的，所以它必须配置`.clientConnector(new ReactorClientHttpConnector(httpClient()))`。第三个方法是配置OAuth2Manager的，所以访问Auth网站的WebClient也必须要配置trustStore。具体就是把定制过TrustStore的webClient配置给`passowrdTokenTokenResponseClient`和`refreshTokenTokenResponseClient`。

另外注意第三个方法创建的是Reactive类型的`ReactiveOAuth2AuthorizedClientManager`，和上面的代码创建的`AuthorizedClientServiceOAuth2AuthorizedClientManager`是不一样的，所以，WebClient就不再使用`ServletOAuth2AuthorizedClientExchangeFilterFunction`了，而是`ServerOAuth2AuthorizedClientExchangeFilterFunction`。

```java
private HttpClient httpClient() {
	final HttpClient httpClient = HttpClient.create().secure(ssl -> {
		try {
		KeyStore trustStore = KeyStore.getInstance("JKS");
		trustStore.load(new FileInputStream(jksFileName), jksPassword.toCharArray());
		TrustManagerFactory trustManager = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
		trustManager.init(trustStore);
		
		KeyStore keyStore = KeyStore.getInstance("JKS");
		keyStore.load(new FileInputStream(jksFileName), jksPassword.toCharArray());
		final KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance("SunX509");
		keyManagerFactory.init(keyStore, jksPassword.toCharArray());
		
		ssl.sslContext(SslContextBuilder
				.forClient()
				.clientAuth(ClientAuth.REQUIRE)
				.trustManager(trustManager)
//		    		.keyManager(keyManagerFactory)
				.build());
		} catch(Exception e) {
			e.printStackTrace();
		}
	});
	return httpClient;
}

private WebClient webClient() {
	if (webClient == null) {
		try {
			ServerOAuth2AuthorizedClientExchangeFilterFunction filter = 
					new ServerOAuth2AuthorizedClientExchangeFilterFunction(isgCloudOauthAuthorizedClientManagerReact(isgCloudOauthClientRegistration()));
			filter.setDefaultClientRegistrationId(REGISTRATION_ID);
			
			// for Buffer
			final int size = 16 * 1024 * 1024;
			final ExchangeStrategies strategies = ExchangeStrategies.builder()
				.codecs(codecs -> codecs.defaultCodecs().maxInMemorySize(size))
				.build();
			
			webClient = WebClient.builder()
					.clientConnector(new ReactorClientHttpConnector(httpClient()))
					.exchangeStrategies(strategies)
					.filter(filter)
					.build();
		} catch (KeyManagementException | NoSuchAlgorithmException | KeyStoreException | CertificateException | IOException  e) {
			e.printStackTrace();
			logger.error("failed to get web client", e);
		}
	}
	return webClient;
}

private ReactiveOAuth2AuthorizedClientManager isgCloudOauthAuthorizedClientManagerReact(ClientRegistration isgCloudOauthClientRegistration) 
		throws KeyManagementException, NoSuchAlgorithmException, KeyStoreException, CertificateException, IOException {
	ReactiveClientRegistrationRepository clientRegistrationRepository = new InMemoryReactiveClientRegistrationRepository(isgCloudOauthClientRegistration);
	ReactiveOAuth2AuthorizedClientService authorizedClientService = new InMemoryReactiveOAuth2AuthorizedClientService(clientRegistrationRepository);
	
	// create SSL enabled webclient for provider thus can make AUTH SSL enabled.
	WebClientReactivePasswordTokenResponseClient passowrdTokenTokenResponseClient = new WebClientReactivePasswordTokenResponseClient();
	WebClientReactiveRefreshTokenTokenResponseClient refreshTokenTokenResponseClient = new WebClientReactiveRefreshTokenTokenResponseClient();
	WebClient webClient = WebClient.builder().clientConnector(new ReactorClientHttpConnector(httpClient())).build();
	passowrdTokenTokenResponseClient.setWebClient(webClient);
	refreshTokenTokenResponseClient.setWebClient(webClient);
	
	ReactiveOAuth2AuthorizedClientProvider authorizedClientProvider = ReactiveOAuth2AuthorizedClientProviderBuilder
			.builder()
			.password(c -> {
				c.accessTokenResponseClient(passowrdTokenTokenResponseClient);
			})
			.refreshToken(c -> {
				c.accessTokenResponseClient(refreshTokenTokenResponseClient);
			})
			.build();
	
	AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager authorizedClientManager = 
			new AuthorizedClientServiceReactiveOAuth2AuthorizedClientManager(clientRegistrationRepository, authorizedClientService);
	
	authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);
	authorizedClientManager.setContextAttributesMapper(
			oAuth2AuthorizeRequest -> Mono.just(ImmutableMap.of(OAuth2AuthorizationContext.USERNAME_ATTRIBUTE_NAME, authUserName, OAuth2AuthorizationContext.PASSWORD_ATTRIBUTE_NAME, authPassword)));
	return authorizedClientManager;
}
```

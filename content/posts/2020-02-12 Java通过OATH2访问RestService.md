---
title: Java通过OATH2访问RestService
date: "2020-02-12T10:49:00.000Z"
description: "简述利用okhttpclient访问过程"
---

主要过程分为两步，第一步去认证服务器拿Token，第二步带着token去资源服务器拿资源。在这片文章中，我利用的是[okhttpclient](https://github.com/square/okhttp/) Version 4.3.1, 也适用于Version 3.14.6.

## 获取Token

这一步非常关键，有几个注意点。

### POST访问

### URL里面要包含grant_type以及username,password

### Request header里面一定要包含"Authorization"

如果没有 `.header("Authorization", "Basic xxx")`，就会出现下面这样的错误

```
{
  "error": "invalid_request",
  "description": "Client credential is not found"
}

```

Basic 后面的字符串应该是[根据“id:password”转BASE64出来的](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication)，但是现在的场景不是，也许是专门设定好的。

完整代码如下

```java
    String authURL = "https://xxx.net/auth/token?grant_type=password&username=xxx&password=xxx";

    final MediaType JSON = MediaType.get("application/json; charset=utf-8");

    RequestBody body = RequestBody.create("{}", JSON);
    Request request = new Request.Builder()
        .header("Authorization", "Basic xxx")
        .url(authURL)
        .post(body)
        .build();

    CertificateFactory certFactory = CertificateFactory.getInstance("X.509");
    File source = new File("C:\\Users\\xxx\\certs\\xxx.pem");

    BufferedInputStream bis = new BufferedInputStream(new FileInputStream(source));
    SSLContext sslContext;
    TrustManager[] trustManagers;
    try {
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        keyStore.load(null, null);

        CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
        Collection<? extends Certificate> certificates = certificateFactory.generateCertificates(bis);
        int i = 0;
        for (Certificate cert : certificates) {
            keyStore.setCertificateEntry("" + i++, cert);
        }

        TrustManagerFactory trustManagerFactory = TrustManagerFactory
                .getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(keyStore);
        trustManagers = trustManagerFactory.getTrustManagers();
        sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, trustManagers, new SecureRandom());
    } catch (Exception e) {
        e.printStackTrace(); // TODO replace with real exception handling tailored to your needs
        return;
    }

    OkHttpClient client = new OkHttpClient.Builder()
            .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustManagers[0]).build();
    try (Response response = client.newCall(request).execute()) {
        System.out.println(response.body().string());
    }
```

如果成功，你会获得如下的结果

```
{
  "access_token": "ZjQ2MTcyOGUtMWFlNC00MjVkLWExYWQtOTY4YTQwZjRjNzA4",
  "expires_in": 86400,
  "scope": null,
  "refresh_token": "OWU5ZTY2YzctYmIzNi00OTNjLTliZWEtYTkxNWIwMzdlZjZi",
  "token_type": "Bearer"
}
```

多执行几次，每次结果都会不一样。

## 利用上面的Token来访问资源

这里面的注意点是

### 由于token的类型是Bearer，所以要在header里面注明

### 这次的访问是GET类型的

### 这次的request可以不用sslSocketFactory

### URL里面不要忘记使用转移符号\

```java
    String dataURL = "https://xxx.net/accounts/account?search={\"$and\":[...]}&projection={\"Mnemonic\":1,\"ShortName\":1,\"xxx.yyy\":1,\"xxx.yyy\":1}";
    request = new Request.Builder()
        .header("Authorization", "Bearer ZjQ2MTcyOGUtMWFlNC00MjVkLWExYWQtOTY4YTQwZjRjNzA4")
        .url(dataURL)
        .get()
        .build();
    client = new OkHttpClient();
    try (Response response = client.newCall(request).execute()) {
        System.out.println(response.body().string());
    }
```

成功后，结果如下：

```
{"code":200,"results":[{"_id":"12376402".....}
```
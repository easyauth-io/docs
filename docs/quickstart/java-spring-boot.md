# Get started with Java Spring Boot

Learn how to authenticate users in a Java Spring Boot application using EasyAuth.

???+ abstract "TLDR: Try the sample Java-Spring-Boot project"

	1. Sign in to [easyauth.io](https://easyauth.io){target=_blank} and create a new 'Registered Client' with redirect URI set to `http://127.0.0.1:8080/login/oauth2/code/easyauth`
	
    2. Clone the sample app from [https://github.com/easyauth-io/easyauth-spring-boot-example](https://github.com/easyauth-io/easyauth-spring-boot-example){target=_blank}
	
	    `git clone https://github.com/easyauth-io/easyauth-spring-boot-example.git`
	
	3. Open the project in your favourite editor.
	
	4. Edit the `src/main/resources/application.properties` file and set the values from your 'Registered Client' that you created in step 1 in place of the curly braces - {}.
	
	5. Run the project and visit [http://127.0.0.1:8080](http://127.0.0.1:8080){target=_blank}

## 1. Create a new Spring Boot Application	

Generate a new spring boot web project from [https://start.spring.io](https://start.spring.io){target=_blank}.

Add the `spring-boot-starter-oauth2-client` starter in `pom.xml` file of your Maven project, it provides all the necessary dependencies required to authenticate your application.

``` xml title="pom.xml"
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-oauth2-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
<dependencies>
```

## 2. Configure application.properties file

It is very easy to configure your application for authentication using spring security with EasyAuth. Edit your application's configuration file i.e `application.properties` file. You can also use `application.yml` file providing required syntax. Configure the Oauth2 client and provider. Use credentials from your `Registered Client` that your created in [EasyAuth](https://easyauth.io){target=_blank}.

### Sample Properties File

```bash title="src/main/resources/application.properties"
server.forward-headers-strategy=FRAMEWORK

spring.security.oauth2.client.registration.easyauth=easyauth
spring.security.oauth2.client.registration.easyauth.client-id={client_id}
spring.security.oauth2.client.registration.easyauth.client-secret={client_secret}
spring.security.oauth2.client.registration.easyauth.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.easyauth.scope=openid
spring.security.oauth2.client.registration.easyauth.client-name={client_name}


spring.security.oauth2.client.provider.easyauth.issuer-uri=https://{your_subdomain}.app.easyauth.io/tenantbackend
spring.security.oauth2.client.provider.easyauth.authorization-uri=https://{your_subdomain}.app.easyauth.io/tenantbackend/oauth2/authorize
spring.security.oauth2.client.provider.easyauth.token-uri=https://{your_subdomain}.app.easyauth.io/tenantbackend/oauth2/token
spring.security.oauth2.client.provider.easyauth.redirect-uri={Redirect Uri such as http://127.0.0.1:8080/login/oauth2/code/easyauth}
spring.security.oauth2.client.provider.easyauth.user-info-uri=https://{your_subdomain}.app.easyauth.io/tenantbackend/userinfo


easyauth.config.baseuri=https://{your_subdomain}.app.easyauth.io

```

---
**NOTE** 
Carefully use your credentials to provide `client-id` and `client-secret`.
 
---

## 3. Adding EasyAuth login

To add login using EasyAuth to your application, create a class to provide an instance of [SecurityFilterChain](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/web/SecurityFilterChain.html) and add the `@EnableWebSecurity` and `@Configuration` annotations.

```java title="src/main/java/com/easyauth/easyAuthExample/config/Oauth2LoginSecurityConfig.java"
package com.easyauth.easyAuthExample.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

import static org.springframework.security.config.Customizer.withDefaults;

@EnableWebSecurity
@Configuration
public class Oauth2LoginSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http.authorizeHttpRequests(authorizeRequests -> authorizeRequests.anyRequest()
                .authenticated()).oauth2Login(withDefaults());
        return http.build();
    }

}
```

---
> **Here, Spring Security is configured to require authentication on all the paths, you can customize it using the [HttpSecurity](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/config/annotation/web/builders/HttpSecurity.html){target=_blank} instance as you wish.**

Learn more about Spring Security Oauth configuration [here](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html){target=_blank}.

---

## 4. Adding a controller to get profile details

Now let's add controller file to provide controllers for index page and profile page to request the authenticated user details from EasyAuth resource server, using the access token.

Here, We're using reactive `WebClient` from [Spring WebFlux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#spring-web-reactive) to send HTTP requests and receive HTTP response. 

### Add the `webflux` dependency in your maven project.

```xml title="pom.xml"
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webflux</artifactId>
		</dependency>
```
### Configure WebClient instance

Create a class to provide instance of `WebClient`. Consider the following sample code.

```java title="src/main/java/com/easyauth/easyAuthExample/config/WebClientConfig.java"
package com.easyauth.easyAuthExample.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientProvider;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientProviderBuilder;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.web.DefaultOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.web.OAuth2AuthorizedClientRepository;
import org.springframework.security.oauth2.client.web.reactive.function.client.ServletOAuth2AuthorizedClientExchangeFilterFunction;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {
    @Bean
    WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2Client =
                new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
        return WebClient.builder()
                .apply(oauth2Client.oauth2Configuration())
                .build();
    }

    @Bean
    OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
                OAuth2AuthorizedClientProviderBuilder.builder()
                        .authorizationCode()
                        .refreshToken()
                        .build();
        DefaultOAuth2AuthorizedClientManager authorizedClientManager = new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

        return authorizedClientManager;
    }

}
```

### Add a controller to fetch user profile 
 
Consider the following sample code which fetches user profile from EasyAuth


```java title="src/main/java/com/easyauth/easyAuthExample/controller/UserRestController.java"
package com.easyauth.easyAuthExample.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClient;
import org.springframework.security.oauth2.client.annotation.RegisteredOAuth2AuthorizedClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;

import static org.springframework.security.oauth2.client.web.reactive.function.client.ServerOAuth2AuthorizedClientExchangeFilterFunction.oauth2AuthorizedClient;

@RestController
public class UserRestController {
    WebClient webClient;

    public UserRestController(WebClient webClient) {
        this.webClient = webClient;
    }

    @GetMapping("/")
    public String index() {
        return "index page";
    }

    @GetMapping("/profile")
    public String profile(@RegisteredOAuth2AuthorizedClient("easyauth") OAuth2AuthorizedClient authorizedClient,
                          @Value("${easyauth.config.baseuri}") String baseUri) {
        String resourceUri = baseUri + "/tenantbackend/api/profile";
        return webClient
                .get()
                .uri(resourceUri)
                .attributes(oauth2AuthorizedClient(authorizedClient))
                .retrieve()
                .bodyToMono(String.class)
                .block();
    }
}
```

---
Here, We've created a `GetMapping` for the path `/profile` that fetches profile details to the EasyAuth Api and returns them as response.

---
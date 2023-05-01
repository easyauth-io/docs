# Get started with Java Spring Boot

Learn how to authenticate users in a java spring boot application using EasyAuth.

???+ abstract "TLDR: Try the sample Java-Spring-Boot project"

	1. Sign in to [easyauth.io](https://easyauth.io) and create a new 'Registered Client' with redirect URI set to `http://127.0.0.1:8080/login/oauth2/code/easyauth`
	
    2. Clone the sample app from [https://github.com/easyauth-io/easyauth-spring-boot-example](https://github.com/easyauth-io/easyauth-spring-boot-example)
	
	    `git clone git@github.com:easyauth-io/easyauth-spring-boot-example.git`
	
	3. Open the project in IntelliJ idea
	
	4. Edit the `src/main/resources/application.properties` file and set the values from your 'Registered Client' that you created in step1 excluding the curly braces-{}.
	
	5. Also edit `line 27 & 28` of `src/main/java/com/easyauth/easyAuthExample/controller/UserRestController.java` providing the correct values excluding curly braces-{}.
	
	6. Run the project and visit [http://127.0.0.1:8080](http://127.0.0.1:8080)

## 1. Create a new Spring Boot Application	

<!-- To create a new spring boot project from [https://start.spring.io](https://start.spring.io).
    
    1. Select Java 17 and maven build with depecdencies- Spring web and spring oauth2 client.
    2. Click generate and the project zip file will be downloaded.

## 2. Spring Security Configuration

Configure a new bean name securityFilterChain  -->

Generate a new spring boot web project from [https://start.spring.io](https://start.spring.io).

Add the `spring-boot-starter-oauth2-client` starter in `pom.xml` file of the your Maven project, it provides all the necessary dependencies required to authenticate your application.

``` xml
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

It is very easy to configure your application for authentication using spring security with EasyAuth. Edit your application's configuration file i.e `application.properties` file. You can also use `application.yml` file providing required syntax. Configure the OAUTH2 client and provider. Use credentials from your `Registered Client` that your created in [EasyAuth](https://easyauth.io).

`Sample Properties File`
```
spring.security.oauth2.client.registration.easyauth=easyauth
spring.security.oauth2.client.registration.easyauth.client-id={client_id}
spring.security.oauth2.client.registration.easyauth.client-secret={client_secret}
spring.security.oauth2.client.registration.easyauth.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.easyauth.scope=openid
spring.security.oauth2.client.registration.easyauth.client-name={client_name}


spring.security.oauth2.client.provider.easyauth.issuer-uri=https://{your_subdomain}.{app}.easyauth.io/tenantbackend
spring.security.oauth2.client.provider.easyauth.authorization-uri=https://{your_subdomain}.{app}.easyauth.io/tenantbackend/oauth2/authorize
spring.security.oauth2.client.provider.easyauth.token-uri=https://{your_subdomain}.{app}.easyauth.io/tenantbackend/oauth2/token
spring.security.oauth2.client.provider.easyauth.redirect-uri={Redirect Uri such as http://127.0.0.1:8080/login/oauth2/code/easyauth}
spring.security.oauth2.client.provider.easyauth.user-info-uri=https://{your_subdomain}.{app}.easyauth.io/tenantbackend/userinfo

```
---
**NOTE** 
Carefully use your credentials to provide `Client_Id` and `Client_Secret`. `Registration_Id` & `Provider_Id` can be anything as you wish, Or, You can put the name of OAUTH2 Api provider you're going to use there, in this case EasyAuth that is.
 
---

## 3. Adding EasyAuth login

To add login using EasyAuth to your application, create a class to provide an instance of [SecurityFilterChain](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/web/SecurityFilterChain.html) and add the `@EnableWebSecurity` and `@Configuration` annotations.

``` java
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
> **Here, Spring security is configured to require authentication on all the paths, you can customize it using the [HttpSecurity](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/config/annotation/web/builders/HttpSecurity.html) instance as you wish.**

---

## 4. Adding controller to get profile details

Now let's add controller file to provide controllers for index page and profile page to request the authenticated user details from EasyAuth resource server, using the access token.

Here, We're using reactive `WebClient` from [Spring WebFlux](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#spring-web-reactive) to send HTTP requests and receive HTTP response. 

> Add the `spring-boot-starter-webflux` starter dependency in your maven project.

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webflux</artifactId>
		</dependency>
```
Now, We first need to configure webClient instance.
> Create a class to provide instance of `webClient`. Consider the following sample code.
 ``` java
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

 > Now, adding a Controller Class to configure REST API paths. Consider the following sample code.

 ```java
 package com.easyauth.easyAuthExample.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
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
    public String profile(@RegisteredOAuth2AuthorizedClient("easyauth") OAuth2AuthorizedClient authorizedClient) {
        String resourceUri = "https://{your_subdomain}.{app}.easyauth.io/tenantbackend/api/profile";		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webflux</artifactId>
		</dependency>

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
Here, We've created a `GetMapping` path for the index page `/` and a `GetMapping` path `/profile` that sends HTTP request for the profile details to the EasyAuth Api and returns the response.

---
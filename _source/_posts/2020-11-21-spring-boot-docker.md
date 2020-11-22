---
layout: blog_post
title: "Spring Boot Application in Docker"
author:
by: advocate|contractor
communities: [devops,security,mobile,.net,java,javascript,go,php,python,ruby]
description: ""
tags: []
tweets:
- ""
- ""
- ""
image:
type: awareness|conversion
---



Most certainly, you have heard about Docker. After years of hype, it has become the somewhat standard technology for everyday DevOps operations. It's a deal-breaker for cloud applications and that’s why it got so much attention in recent years.

Docker has enabled a new unified way of application deployment. The basic idea is simple: instead of preparing a target environment on each machine bring it as a part of your application in the form of a container. Indeed, that means no conflicting library versions or overlapping network ports. Built images are immutable - your application works the same way locally, on your teammate's computer, or in the cloud. Also, it's possible to run multiple instances of the container on the same machine, and that helps to increase the density of deployment, bringing down costs.

In this tutorial, you build and run a simple web application into the Docker-compatible image using [buildpacks support][buildpacks-in-2.3.0], introduced in Spring Boot 2.3.0 

**Prerequisites**

* [Java 11+][java11]
* Unix-like shell
* [Docker][install-docker] installed
* [Okta CLI][okta-cli] installed


## Bootstrap Secure Spring Boot Application

Start with creating a Spring Boot application using [Spring Boot Initializr](https://start.spring.io/). 
That could be done via the web interface or using handy curl command:


```sh
curl https://start.spring.io/starter.tgz -d dependencies=web,okta \
    -d groupId=com.okta \
    -d artifactId=demospringboot \
    -d type=gradle-project \
    -d language=kotlin \
    -d baseDir=springboot-docker-demo | tar -xzvf -
```

This command requests Spring Boot Initializer to generate an application which is using the Gradle build system and Kotlin programming language.
It also configures dependencies on Spring Web and Okta. Created project is automatically unpacked to the `springboot-docker-demo` directory.


Update your generated application file to allow unauthenticated access configured in `WebSecurityConfigurerAdapter` and introduce your controller `WebController` which welcomes user. In Kotlin, it's safe put everything in single file `src/main/kotlin/com/okta/demospringboot/DemoApplication.kt`:

```kotlin
package com.okta.demospringboot

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter
import org.springframework.context.annotation.Configuration
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.ResponseBody
import org.springframework.web.bind.annotation.RestController
import java.security.Principal

@SpringBootApplication
class DemoApplication

fun main(args: Array<String>) {
    runApplication<DemoApplication>(*args)
}

@Configuration
class OktaOAuth2WebSecurityConfigurerAdapter: WebSecurityConfigurerAdapter() {
    override fun configure(http: HttpSecurity) {
        http.authorizeRequests().anyRequest().permitAll()
    }
}

@RestController
class WebController {
    @RequestMapping("/")
    fun home(user: Principal?) = "Welcome, ${user?.name ?: "guest"}!"
}
```

### Run Spring Boot Application

Start your application in the project folder via command line:

```
./gradlew bootRun
```

Then, open a browser at `http://localhost:8080`. Your web application greets the guest user:

{% img blog/spring-boot-docker/welcome-guest.png alt:"Webpage displaying welcome guest" width:"710" %}{: .center-image }


## Build Spring Boot Docker Image and Run Application

Since version 2.3.0 Spring Boot supports [Cloud Native Buildpacks][buildpacks]. It has become straightforward to deploy a web service to the popular clouds using Buildpacks due to the mass adoption of this technology. 



Build your application into the local docker daemon:

```sh
./gradlew bootBuildImage --imageName=springbootdemo
```

Next, start your containerized web application with docker:

```sh
docker run -it -p8080:8080 springbootdemo
```

As expected, your web application will be available on `http://localhost:8080` 

## Secure Your Spring Boot Application in Docker

User management is never an easy task, and most certainly is not the main objective of your application. Okta is a SaaS identity management provider that helps you to take care of routine work such as the implementation of OAuth 2.0, social login and SSO(Single Sign-On). It's very developer-friendly, and it has excellent integration with different frameworks, including Spring Boot.

Start with [installing the Okta CLI tool][okta-cli] - it's a real time saver for developers. 

1. **Create your free developer [account with Okta][okta-signup]** , no credit card required:

    ```sh
    okta register

    ...(provide registration details)...
    ...
    To set your password open this link:
    https://dev-xxxxxxxxxxxx.okta.com/welcome/ABCDEFG

    ```
    Don't forget to set your password using the link above!

2. Create an Okta application. CLI will ask several questions, don't worry about them; it's safe to accept defaults. You can change the application parameters later. In the project directory, execute:
    ```sh
    okta apps create
    
    ...(accept defaults)...
    ```
3. When an application successfully created, its configuration is saved in the current folder `.okta.env` file.
    ```sh
    cat .okta.env
    export OKTA_OAUTH2_ISSUER="https://dev-xxxxxx.okta.com/oauth2/default"
    export OKTA_OAUTH2_CLIENT_SECRET="viVC58i6MzQHzAz9BeXjzhWpSz8qbg6U5B4RXnre"
    export OKTA_OAUTH2_CLIENT_ID="zoa1bzlj7DWmzSI8o5d6"
    ```
    These parameters need to be injected into the application to enable authorization.

    **⚠️ NOTE: make sure you _never_ checkin credentials to the VCS**


### Configure Spring Boot Security

Previously, your webpage was accessible for everyone. To allow access for authorized users only update Spring Boot Security configuration in `src/main/kotlin/com/okta/demospringboot/DemoApplication.kt`:

```kotlin
@Configuration
class OktaOAuth2WebSecurityConfigurerAdapter: WebSecurityConfigurerAdapter() {
    override fun configure(http: HttpSecurity) {
        http.authorizeRequests().anyRequest().authenticated();
    }
}
```

That's it. Okta Spring Boot Starter takes care of the rest configuration.

Rebuild the application again:

```sh
./gradlew bootBuildImage --imageName=springbootdemo
```

The `--imageName` parameter allows specifying an image name. Without it, the name would look like `appName:0.0.1-SNAPSHOT`


### Start Spring Boot Application in Docker

When your application starts Okta module reads environment variables to configure security in your application. Start your application with additional configuration:

```
docker run -it -p8080:8080 \
    -e OKTA_OAUTH2_ISSUER="https://dev-xxxxxx.okta.com/oauth2/default" \
    -e OKTA_OAUTH2_CLIENT_SECRET="yyyyyyyyyyyyyyyyyyy" \
    -e OKTA_OAUTH2_CLIENT_ID="zzzzzzzzzzzzzzzz" \
    springbootdemo
```

The argument `-e` allows to set an environment variable for the application running _inside_ your container and `-p` maps container's point to the `localhost`.

Now, head over to `http://localhost:8080`, and you'll be asked to log in using Okta standard form. Put your login credentials, and upon successful sign-in, your web browser will be redirected to the main page, displaying a welcoming message.

{% img blog/spring-boot-docker/browser-welcomes-logged-in-user-via-okta.png alt:"Webpage displaying 'welcome username'" width:"744" %}{: .center-image }

Congratulations, you have created a simple Spring Boot application contextualized with Docker and secured with Docker.


**Bonus - use dotenv file**

While providing a few environment variables in the command-line might be acceptable, it's not very convenient and can leave unwanted traces of the secrets in your terminal history. Docker supports dotenv file format, which makes it easier to set multiple environment parameters. 

1. Create `.env` file in the root of the project and set desirable environment variables:
    ```sh
    OKTA_OAUTH2_ISSUER=https://dev-xxxxxx.okta.com/oauth2/default
    OKTA_OAUTH2_CLIENT_SECRET=viVC58i6MzQHzAz9BeXjzhWpSz8qbg6U5B4RXnre
    OKTA_OAUTH2_CLIENT_ID=zoa1bzlj7DWmzSI8o5d6
    ```

2. Always be extra careful with credentials - avoid leaking them to the version control system even for the pet project. Add `.env` to `.gitignore`.
    ```sh
    echo ".env" >> .gitignore
    ```

3. Run Docker providing your `.env` file via `--env-file` argument
    ```sh
    docker run -it -p8080:8080 --env-file .env springbootdemo 
    ```
    Looks much cleaner, isn't it?



## Learn More About Docker, Spring Boot, Buildpacks and Okta


Checkout [source code on GitHub](https://github.com/ruXlab/spring-boot-docker-buildpacks).

In this brief tutorial, you created a simple Spring Boot application welcoming user, configured Okta OAuth 2.0 provider, built buildpack into local docker daemon and run created docker image providing runtime configuration. This bootstrap project is a great starting point for your next cloud-native project.

See other relevant tutorials:
* [OAuth 2.0 Java Guide: Secure Your App in 5 Minutes][java-oauth2]
* [Angular + Docker with a Big Hug from Spring Boot][angular-spring-docker]
* [A Quick Guide to OAuth 2.0 with Spring Security][oauth2-spring-security-guide]

Follow us for more great content and updates from our team! You can find us on [Twitter](https://twitter.com/oktadev), [Facebook](https://www.facebook.com/oktadevelopers), subscribe to our [YouTube Channel](https://youtube.com/c/oktadev) or start the conversation below!

[install-docker]: https://docs.docker.com/get-docker/
[java11]: https://adoptopenjdk.net/
[okta-cli]: https://github.com/okta/okta-cli
[okta-signup]: https://developer.okta.com/signup/
[oci]: https://opencontainers.org/
[buildpacks]: https://buildpacks.io/ 
[buildpacks-in-2.3.0]: https://spring.io/blog/2020/01/27/creating-docker-images-with-spring-boot-2-3-0-m1
[java-oauth2]: /blog/2019/10/30/java-oauth2
[angular-spring-docker]: /blog/2020/06/17/angular-docker-spring-boot
[oauth2-spring-security-guide]: /blog/2019/03/12/oauth2-spring-security-guide
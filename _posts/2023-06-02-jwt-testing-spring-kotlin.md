---
layout: single
classes: wide
title:  "Testing JWT in Spring Boot with Kotlin"
date:   2023-06-02
categories: spring-boot kotlin jwt testing
---

I _love_ automated tests. While I'm not really a TDD fan in most cases (I prefer to write tests after the code is written), I still find them invaluable. They help me to understand the code better, and they help me to refactor it with confidence. But the main point is that I'm usually too lazy to test things manually, and on my personal projects I prefer to dedicate the time I have to writing new features, not testing the old ones. While working on one of my recent personal projects I decided to use JWT for authentication, and I wanted to test it properly in Web layer. So, here's how I did it.

<!--more-->

## JWT in Spring Boot

JWT Authentication in SpringBoot usually works out of the box. You just need to add `spring-boot-starter-security` dependency to your project, and Spring will take care of the rest. You can find more details in [Spring Security documentation](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-jwt). In my case I used Firebase Authentication, but really nothing else then setting up the project in Firebase and configuring Spring Security was required. Any other JWT-compatible provider should work just as well (e.g. Auth0, Okta, etc.)

OAuth2 Resource Server config:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://securetoken.google.com/your_project_id
```

Also, we need to configure SpringSecurity itself:

```kotlin
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(
    prePostEnabled = true,
    securedEnabled = true,
    jsr250Enabled = true
)
class SecurityConfiguration {
    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http.invoke {
            cors { }
            csrf { disable() }

            authorizeRequests {
                authorize("/actuator", permitAll)
                authorize("/actuator/**", permitAll)
                authorize(anyRequest, authenticated)
            }
            oauth2ResourceServer {
                jwt {}
            }
            sessionManagement {
                sessionCreationPolicy = SessionCreationPolicy.STATELESS
            }
        }

        return http.build()
    }
}
```

This will make sure all requests to `/actuator` endpoints are not authenticated, and all other requests are authenticated using JWT. SpringSecurity will take care of the rest and you can extract the user details from the `Authentication` or `Principal` objects in your controllers.

## Testing JWT

Great thing about testing in Spring is that you can decide how much of the application you want to test. You can test just the controller, or you can test the whole application. I my case I just wanted to test the controller, so I used `@WebMvcTest` annotation. For start, here's the simple sample test:

```kotlin
@WebMvcTest(TestController::class)
@Import(SecurityConfiguration::class)
@WithMockJwtUser
class DatabaseControllerTest {
    @Autowired
    lateinit var mockMvc: MockMvc
}
```

Looks pretty standard. We just define a `@WebMvcTest` for our controller, and we import our `SecurityConfiguration` which we defined earlier. 

The new component here is `@WithMockJwtUser` annotation to mock the JWT user. Here's the code for it:

```kotlin
@Retention(AnnotationRetention.RUNTIME)
@WithSecurityContext(factory = WithMockJwtUserSecurityContextFactory::class)
annotation class WithMockJwtUser(
    val username: String = "user",
    val email: String = "user@example.com",
    val roles: Array<String> = ["USER"]
)
```

It's simple enough - we can define the username, email and roles for our user. But how does it work? Let's look at the `WithMockJwtUserSecurityContextFactory`:

```kotlin
@TestComponent
class WithMockJwtUserSecurityContextFactory : WithSecurityContextFactory<WithMockJwtUser> {
    override fun createSecurityContext(annotation: WithMockJwtUser): SecurityContext {
        val context: SecurityContext = SecurityContextHolder.createEmptyContext()
        // Set headers
        val headers = mapOf(
            "alg" to "HS256",
            "typ" to "JWT"
        )

        // Set JWT claims for authentication, e.g. "sub", "roles", etc.
        val claims: Map<String, Any> = mutableMapOf(
            "iss" to "https://mockissuer.com",
            "aud" to "audience",
            "sub" to annotation.username, // this is an external User ID from Firebase
            "name" to annotation.username,
            "email" to annotation.email,
            "picture" to "https://placekitten.com/96/96",
            "roles" to annotation.roles.joinToString(",") { "ROLE_${it.trim()}" }
        )

        val issuedAt = Instant.now()
        val expiresAt = issuedAt.plus(Duration.ofDays(1))
        val jwt = Jwt("mockToken", issuedAt, expiresAt, headers, claims)
        val authentication = JwtAuthenticationToken(jwt)

        authentication.isAuthenticated = true
        authentication.details = WebAuthenticationDetails(MockHttpServletRequest())
        context.authentication = authentication

        return context
    }
}
```

The above implementation is pretty simple. We just create a JWT token with the claims we need, and then we create a `JwtAuthenticationToken` with the token we created. The `JwtAuthenticationToken` is a standard Spring Security class, and it's used to authenticate the user. The `JwtAuthenticationToken` is then set as the authentication object in the `SecurityContext` and returned.

When our controller is called, the `JwtAuthenticationToken` will be used to authenticate the user, and we can extract the user details from the `Authentication` object.

As we also have properties like `roles` in our JWT token, we can use them to test the authorization part of our controller. For example, if we have a method like this:

```kotlin
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/admin")
fun admin(): String {
    return "admin"
}
```

We can test it like this:

```kotlin
@Test
fun `admin endpoint should return 200`() {
    mockMvc.get("/admin") {
        accept = MediaType.APPLICATION_JSON
    }.andExpect {
        status { isOk }
    }
}
```

And that's it. We can now easily test our JWT authentication and authorization in our controller tests.

Until next time :peace_symbol:

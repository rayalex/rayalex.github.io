---
layout: single
classes: wide
title:  "Using OSO Authorization Framework with SpringBoot & SpringSecurity"
date:   2023-06-02
categories: spring-boot spring-security kotlin oso polar
header:
    overlay_image: /assets/images/spring-security-with-oso.jpg
    overlay_filter: 0.5
---

```
Why couldn't the polar bear access the fish database?
It kept getting the message: "You do not have the bear-er token!"
```

Now that we have that out of the way, let's get to the real stuff. In this post, we will see how to use the [OSO Authorization Framework](https://www.osohq.com/) with SpringBoot and SpringSecurity. We will be using Kotlin for this example, but the same can be done with Java as well.

**Note**
As of the end of 2023 OsoHQ has decided to [deprecate the open-source version](https://github.com/osohq/oso?tab=readme-ov-file#deprecated) of the Oso Authorization Framework. Parts of this post were written before this announcement and hence I've decided to publish it, since it still somewhat applies to the commercial (cloud) version of the framework.

In the future I might either write my own simple authorization framework or leverage the Oso in its commercial form. Currently my project is a Modular Monolith and doesn't really require anything complex.

## What is OSO?

Oso (in its open-source form) is a library that allows you to define authorization rules in a simple, declarative language called [Polar](https://www.osohq.com/docs/reference/polar/syntax). It is a powerful language that allows you to define complex authorization rules in a very simple way. It is also very easy to integrate with your application. We can use this language to model authorization rules in RBAC, ReBAC, ABAC and other models.

Here's a short example of a Polar rule:

```polar
actor User {}

resource Organization {
  roles = ["owner"];
}


resource Repository {
  permissions = ["read", "push"];
  roles = ["contributor", "maintainer"];
  relations = { parent: Organization };

  "read" if "contributor";
  "push" if "maintainer";

  "contributor" if "maintainer";
}

...
```

Here we have a rule that says that a `User` can `read` a `Repository` if they have the `contributor` role. Similarly, they can `push` to a `Repository` if they have the `maintainer` role.

The public interface is also very easy to use. Here's an example of how to use the above rule:

```kotlin
oso.authorize(user, "read", repo);
```

Here, we are asking Oso to check if the `user` has the `read` permission on the `repo`. If the user has the permission, `true` is returned, otherwise we get a `false` back.

Oso has a great getting started guide that you can follow to get a better understanding of how it works. You can find it [here](https://www.osohq.com/docs/oss/java/guides/rbac.html). We'll be skipping the basics and focus on integrating it with SpringSecurity.

## SpringSecurity with OSO

There's no out of the box way to integrate Oso with SpringSecurity, but we can leverage the extensibility of Spring's MethodSecurity to integrate both. 

### Prerequisites

Let's start by setting up a simple landscape application model. In our case we'll have `User`s, `Organization`s and `Repository`s. A `User` can be a member of multiple `Organization`s and a `Repository` can belong to only one `Organization`. A `User` can have different roles in different `Organization`s. 

### Registering Java classes with Oso

Oso requires us to register the Java classes that we want to use in the Polar rules. We can do this by using the `registerClass` method on the `Oso` object. Let's create a simple configuration class to do this and also return an instance of `Oso` when needed:

```kotlin
@Configuration
class SecurityConfiguration {
    @Bean
    fun oso(): Oso {
        val oso = Oso()
        oso.registerClass(UserPrincipal::class.java, "User")

        return oso
    }
}
```

We can register all the classes manually by calling the `registerClass` method for each class. But this can get tedious if we have a lot of classes. If you want to be fancier, we can define a custom and use it to discover and register all the classes automatically. 

```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.CLASS)
annotation class Securable(val name: String)
```

And to register them:

```kotlin
private fun registerSecurables(oso: Oso) {
    // scan all the classes with @Securable annotation and register them
    val scanner = ClassPathScanningCandidateComponentProvider(false)
    scanner.addIncludeFilter(AnnotationTypeFilter(Securable::class.java))

    for (beanDefinition in scanner.findCandidateComponents("com.example")) {
        val clazz = Class.forName(beanDefinition.beanClassName)
        if (clazz.isAnnotationPresent(Securable::class.java)) {
            oso.registerClass(clazz, clazz.getAnnotation(Securable::class.java).name)
        } else {
            oso.registerClass(clazz, clazz.simpleName)
        }
    }
}
```
Oso will use these bindings in order to resolve the classes in the Polar rules as well as call the methods on them for role, permission adn relation resolution.

### Defining the models

We'll start by defining the models for our application. We'll just use simple classes here, but in the real app you would probably use JPA entities as your base.

If you're subscribing to the [DDD](https://en.wikipedia.org/wiki/Domain-driven_design) philosophy, you can separate the models into the domain and the data models.

#### Organization

```kotlin
@Securable("Organization")
data class Organization(val id: UUID)
```

#### User

We model the user as a collection of `OrganizationMembership`s. Each `OrganizationMembership` contains the `Organization` and the `Role` of the user in that `Organization`.

```kotlin
@Securable("User")
data class UserPrincipal(val memberships: List<OrganizationMembership>) : UserDetails
```

#### Repository

```kotlin
@Securable("Repository")
data class Repository(val id: UUID, val organization: Organization)
```

### SpringSecurity Configuration

By convention, `Oso` follow the simple model of `actor`, `action` and `resource`. We can use this to define a custom `PermissionEvaluator` that can be used by SpringSecurity to evaluate permissions. 

```kotlin
class OsoPermissionEvaluator(private val oso: Oso) : PermissionEvaluator {
    override fun hasPermission(authentication: Authentication?, targetDomainObject: Any?, permission: Any?): Boolean {
        if (authentication == null) return false
        if (targetDomainObject == null) return false
        if (permission == null || permission !is String) return false
        return oso.isAllowed(authentication.principal, permission, targetDomainObject)
    }
}
```

The above implementation uses `targetDomainObject` which allows us to write the rules that use the resources we already have as objects. In case you want to use the `id` of the resource, you can use the `hasPermission(authentication: Authentication?, targetId: Serializable?, targetType: String?, permission: Any?): Boolean` method instead. This one would be useful when securing the controllers.

```kotlin
override fun hasPermission(
    authentication: Authentication?,
    targetId: Serializable?,
    targetType: String?,
    permission: Any?
): Boolean {
    if (authentication == null) return false
    if (targetId == null) return false
    if (targetType == null) return false
    if (permission == null || permission !is String) return false

    when (targetType) {
        "YourType" -> {
            val loadedType = yourRepository.findById(targetId)
            return hasPermission(authentication, database, permission)
        }
        else -> {
            logger.warn("SECURITY: Unknown target type: {}", targetType)
            return false
        }
    }
}

```


As a last step before we can start using Oso, we need to configure SpringSecurity to use our custom `PermissionEvaluator`. We can do this by extending the `MethodSecurityExpressionHandler`:

```kotlin
@Configuration
@EnableMethodSecurity
internal class MethodSecurityConfiguration(private val oso: Oso) {
    @Bean
    fun expressionHandler(): MethodSecurityExpressionHandler {
        val handler = DefaultMethodSecurityExpressionHandler()
        handler.setPermissionEvaluator(OsoPermissionEvaluator(oso))

        return handler
    }
}
```

### Securing the endpoints with SpringSecurity

Now that we have everything in place, we can start securing our endpoints. We can either secure the Controllers or the Service methods. In this example, we'll secure the Service methods first using the `@PreAuthorize` annotation. 

```kotlin
@Service
class RepositoryService {
    @PreAuthorize("hasPermission(#repo, 'push')")
    fun push(repo: Repository, content: Content): Repository {
        return repo
    }
}
```
Or, if you want to use the `id` of the resource to secure reading the repository:

```kotlin
@Controler
class RepositoryController(private val repositoryService: RepositoryService) {
    @GetMapping("/{id}")
    @PreAuthorize("hasPermission(#id, 'Repository', 'read')")
    fun getRepository(@PathVariable id: UUID): Repository {
        return repositoryService.getRepository(id)
    }
}
```

### Conclusion
In this post, we saw how to integrate the OSO Authorization Framework with SpringSecurity. We used Kotlin for this example, but the same can be done with Java as well. It's also possible to use the same approach to integrate other authorization frameworks with SpringSecurity due to its extensibility.

The complete code for this example will be published later on my GitHub. I'll update this post with the link once it's published.
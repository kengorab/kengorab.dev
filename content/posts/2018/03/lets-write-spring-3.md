---
title: "Let's Write Spring! (Part 3)"
date: 2018-03-25T20:07:56-05:00
tags: [java, spring, diy]
---

_This is a copy of a post, [originally written on Medium](https://medium.com/@ken.gorab/lets-write-spring-part-3-352802226417). I've since rehosted it here._

---

If you haven’t read [Part 1](/posts/2018/03/lets-write-spring-1) or [Part 2](/posts/2018/03/lets-write-spring-2) you should definitely do that first.

As I mentioned at the end of Part 2, we’re pretty much done with building out Spring’s DI. We took care of `@Qualifier` annotations in the previous part, but so far it’s only been implemented on `@Autowired` fields. Let’s explore other places in which the `@Qualifier` annotation can be used.

## Step 8: @Qualifier and @Bean methods

As we implemented last time, methods annotated with the `@Bean` annotation will be invoked as the dependency graph gets resolved. Any parameters to these methods will come from the application context as it’s invoked. In the previous part, we discussed the need for the `@Qualifier` annotation on `@Autowired` fields, since it’s unclear to the graph resolver which instance of the type to provide; the `@Qualifier` annotation allows us to specify by name which instance of the type should be provided. Let’s see what it takes to add this to `@Bean` methods (spoiler alert: it’s very simple).

First, let’s make a couple modifications to the `@Qualifier` annotation. The current code specifies a `@Target(ElementType.FIELD)` which limits the annotation’s usage to fields only.

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface Qualifier {
    String value();
}
```
<div class="caption">src/main/java/co/kenrg/hooke/annotations/Qualifier.java</div>

Now, we’ve added the `ElementType.PARAMETER` to the list so it’s enabled on parameters to methods as well. I’ve also made another small modification; previously the `value()` method had a default value of `""`, but I’ve realized that that makes no sense. Specifying a `@Qualifier` annotation without specifying the name should be considered invalid, and by not having a default value here in the `@interface` it will throw a compile-time error if no name is passed into the annotation.

Now that our annotation is modified, let’s see what else needs to be changed. Right now in the `getDependencyUnitsForBeanMethods` within `DependencyCollectors`, we call into a helper method `getDependenciesFromParameters` to return a `List<Pair<String, Class>>` to us. Previously, we had hard-coded null as the left-hand side of the `Pair` because at that time we didn’t care about the exact name of the dependency. Let’s see what changes now that we have the ability to add `@Qualifier` annotations on parameters:
```java
private static List<Pair<String, Class>> getDependenciesFromParameters(Parameter[] parameters) {
    List<Pair<String, Class>> dependencies = Lists.newArrayList();
    for (Parameter parameter : parameters) {
        Annotations annotations = getAnnotations(parameter::getAnnotations);
        Qualifier qAnn = annotations.get(Qualifier.class);
        String name = (qAnn != null && !qAnn.value().isEmpty())
            ? qAnn.value()
            : null;
        dependencies.add(Pair.of(name, parameter.getType()));
    }
    return dependencies;
}
```
<div class="caption">getDependenciesFromParameters method in src/main/java/co/kenrg/hooke/application/graph/DependencyCollectors.java</div>

This is similar to code we’ve seen in the `getDependencyUnitForComponent` method, where we have a left-hand side of `null` if we don’t care about the name of the dependency and we only care about the type; if we do care about the name, then the name will be the left-hand side. With this, let’s modify our `Configuration1.java` class within our example application and verify that our code works!
```java
// Configuration1.java
@Configuration
public class Configuration1 {

    @Bean
    public String superSecretApiKey() {
        return "asoiudhfjkn";
    }

    @Bean
    public String exclamation() {
        return "!";
    }

    @Bean
    public String greeting(@Qualifier("exclamation") String exclamation) {
        return "Howdy" + exclamation;
    }

    @Bean
    public Component2 component2(Component3 component3) {
        return new Component2(component3);
    }
}
```

The differences here are the added `exclamation()` method and the dependency added to the `greeting()` method. Note that without that `@Qualifier` on the parameter, this would fail since there are many instances of `String` within the dependency graph. Making this change and then running the application will print out the message “Component1 says ‘Hello’; Component2 says ‘Howdy!’” which is similar to what we’ve been seeing, only now `Component2` is a little more emphatic.

See, I told you that wouldn’t be too difficult. The code so far has been pushed to the Github repo under the tag [`step-8-qualifier-and-bean-methods`](https://github.com/kengorab/hooke/tree/step-8-qualifier-and-bean-methods).

## Step 9: Constructor-based Injection

Right now, the only form of dependency injection that we’ve built out is field-based dependency injection; we declare a class annotated with `@Component` and add the `@Autowired` annotation on its fields to “ask” for instances of those fields’ types from the application context. Constructor-based injection doesn’t make use of the `@Autowired` annotation at all and in some ways feels more natural and more java-like, since it doesn’t require the injection container to reach into the class’s (potentially private!) fields in order to establish the dependency graph. I’m not going to go too much into the differences and benefits of either approach (you can look at what [the official Spring documentation has to say about that](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/beans.html#beans-constructor-injection), if you scroll down to the **"Constructor-based or setter-based DI?”** section), but suffice it to say that constructor-based injection is both very common and easy to implement.

In the `getDependencyUnitForComponent` method within `DependencyCollectors` we are currently just invoking the default constructor for the component class passed in, when instantiating the dependency within the `DependencyUnit`'s lambda function:
```java
Constructor constructor = componentClass.getConstructors()[0];
Object instance = constructor.newInstance();
```

(Side note, this code will actually fail at runtime if the component class in question had a constructor defined that wasn’t a no-args constructor.) Now that we’re supporting constructor-based injection though, the constructor to use needs to be determined earlier on, when we’re figuring out the list of dependencies this particular class has. Luckily we already have a helper method `getDependenciesFromParameters`, so the addition here is really pretty minimal. The only other thing that needs to change is the code within the `DependencyUnit`'s lambda; now we just need to invoke the constructor that we’ve determined ahead of time, passing in instances from the application context, to obtain our instance. Here’s the entirety of the `getDependencyUnitForComponent` method now, though only a few lines have changed:
```java
public static DependencyUnit getDependencyUnitForComponent(Class componentClass, DependencyInstanceGetter dependencyInstanceGetter) {
    List<Pair<String, Class>> dependencies = Lists.newArrayList();

    Constructor[] constructors = componentClass.getConstructors();
    if (constructors.length > 1) {
        throw new IllegalStateException("Cannot instantiate class with no default constructor: " + componentClass);
    }
    Constructor constructor = constructors[0];
    List<Pair<String, Class>> constructorDependencies = getDependenciesFromParameters(constructor.getParameters());

    Map<Field, Pair<String, Class>> fields = Maps.newHashMap();
    for (Field field : componentClass.getDeclaredFields()) {
        Annotations annotations = getAnnotations(field::getAnnotations);
        Autowired autowiredAnn = annotations.get(Autowired.class);
        Qualifier qualifierAnn = annotations.get(Qualifier.class);

        if (autowiredAnn != null) {
            String depName = qualifierAnn != null && !qualifierAnn.value().isEmpty()
                ? qualifierAnn.value()
                : null;
            fields.put(field, Pair.of(depName, field.getType()));
        }
    }

    dependencies.addAll(fields.values());
    dependencies.addAll(constructorDependencies);

    return new DependencyUnit(
        null,
        componentClass,
        dependencies,
        () -> {
            try {
                List<Object> constructorParameters = dependencyInstanceGetter.getDependencyInstances(constructorDependencies);
                Object instance = constructor.newInstance(constructorParameters.toArray());

                for (Map.Entry<Field, Pair<String, Class>> entry : fields.entrySet()) {
                    Field field = entry.getKey();
                    Pair<String, Class> dependency = entry.getValue();

                    Object value = dependencyInstanceGetter.getDependencyInstances(Lists.newArrayList(dependency)).get(0);
                    if (!field.isAccessible()) field.setAccessible(true);
                    field.set(instance, value);
                }

                return instance;
            } catch (IllegalAccessException | InvocationTargetException | InstantiationException e) {
                throw new IllegalStateException("Could not invoke constructor for class" + componentClass, e);
            }
        }
    );
}
```
<div class="caption">getDependencyUnitForComponent src/main/java/co/kenrg/hooke/application/graph/DependencyCollectors.java</div>

One cool thing here is that we actually get support for `@Qualifier` annotations on constructor parameters for free, since we’re using the `getDependenciesFromParameters` method. To test this, let’s remove the `@Bean` method from `Configuration1.java` which had been creating an instance of `Component2` and instead add the `@Component` annotation to `Component2`. When we run our example application now, we should see the same output as in Step 8, only this time the `Component2` instance is being instantiated via constructor injection.

Like I said, this was pretty easy to implement! As always, the code so far has been pushed to the Github repo, and is available at the [`step-9-constructor-based-injection`](https://github.com/kengorab/hooke/tree/step-9-constructor-based-injection) tag.

## Step 10: @Autowired methods

We only have a couple of things left to do, and one of them is `@Autowired` methods. Using the `@Autowired` annotation on methods has a very similar effect to using the `@Bean` annotation on methods, the only difference being that the `@Bean` methods’ return value will be entered into the application context (and therefore cannot have a return type of `void`) and `@Autowired` methods’ return value will be ignored (if it’s non-`void`). One of the main purposes of `@Autowired` methods is to perform Setter-based injection (which is mentioned in the Spring documentation link I put above discussing constructor-based injection); the idea is that for each dependency your class has, you have a setter method annotated with `@Autowired` which Spring will invoke and pass in the requested dependency instance so you can assign it to a field within your class. This is discouraged in favor of constructor-based injection. Another use case is if your class needs an instance of some dependency from the application context for some other reason, like passing it into a constructor for one of your class’s fields; this is commonly seen when using `JdbcTemplate`:
```java
class SomeDao {
    private JdbcTemplate jdbcTemplate;
    @Autowired
    public void setupJdbc(DataSource dataSource) {
        jdbcTemplate = new JdbcTemplate(dataSource);
    }
}
```
Let’s get into this and start adding some new code. I created a new class to house this bit of functionality (and the functionality of the next section, too) called `DependencyPostSetupHandlers`, and a method in there called `invokeAutowiredMethodsForBeans`. It should accept a list of instances from our application context, as well as a `DependencyInstanceGetter` (which we’d seen before when handling `@Bean` methods on classes). The main strategy here is very simple: find each method on each bean’s class that has the `@Autowired` method, find any dependencies it needs according to its parameters, and invoke it. Here’s what it looks like:
```java
public static void invokeAutowiredMethodsForBeans(List<Object> beanInstances, DependencyInstanceGetter dependencyInstanceGetter) {
    for (Object instance : beanInstances) {
        for (Method method : instance.getClass().getMethods()) {
            boolean hasAutowired = getAnnotations(method::getAnnotations).contains(Autowired.class);

            if (hasAutowired) {
                final List<Pair<String, Class>> dependencies = getDependenciesFromParameters(method.getParameters());
                List<Object> args = dependencyInstanceGetter.getDependencyInstances(dependencies);
                try {
                    method.invoke(instance, args.toArray());
                } catch (IllegalAccessException | InvocationTargetException e) {
                    throw new IllegalStateException("Could not invoke @Autowired method " + method, e);
                }
            }
        }
    }
}
```
<div class="caption">invokeAutowiredMethodsForBeans method in src/main/java/co/kenrg/hooke/application/DependencyPostSetupHandlers.java</div>

Note that the `getDependenciesFromParameters` method is the same as in `DependencyCollectors`. I’ve pulled that out into a shared utility class called `Parameters.java` so it can be accessed from both places. Before we can test this out though, we need to make a couple of changes to `Application.java`. In order to invoke the `@Autowired` methods on all of our instances, we need to get all instances out of the application context. We currently have this method
```java
public List<Object> getBeans(List<Pair<String, Class>> beans)
```

but it seems excessive to create a `Pair` to pass in when we know we won’t be caring about the name of the parameter (since all we care about is the type). So let’s make a new method in there really quickly:
```java
public List<Object> getBeansOfTypes(Collection<Class> types) {
    List<Object> values = Lists.newArrayList();
    for (Class type : types) {
        Object instance = allBeans.get(type);
        if (instance == null) {
            throw new IllegalStateException("No bean available of type " + type);
        }
        values.add(instance);
    }

    return values;
}
```
<div class="caption">getBeansOfTypes method in src/main/java/co/kenrg/hooke/application/ApplicationContext.java</div>

So now, we can add these 2 lines to the end of the `run` method in `Application`:
```java
List<Object> allInstances = 
    applicationContext.getBeansOfTypes(componentClasses);
invokeAutowiredMethodsForBeans(
  allInstances,
  applicationContext::getBeans
);
```

Before we can test this out, we need to modify the annotation to support being applied to a method since right now it’s only valid on fields. This is very similar to what we had done with the `@Qualifier` annotation back in Step 8, except now it’s adding `ElementType.METHOD` instead of `ElementType.FIELD`. Finally, let’s test this out by adding an `@Autowired` method to `Component2`:

```java
@Autowired
public void setup(@Qualifier("superSecretApiKey") String apiKey) {
    System.out.printf("API key: %s\n", apiKey);
}
```

Now, when we run our application we should see the “API key: …” message print out right before our normal output gets printed. Awesome! We now have `@Autowired` support on methods! The code so far has been pushed to the Github repo, and is available at the [`step-10-autowired-methods`](https://github.com/kengorab/hooke/tree/step-10-autowired-methods) tag.

There’s just one final thing to do now before we’re done…

## Step 11: @PostConstruct methods

Methods annotated with `@PostConstruct` function in a very similar way to those annotated with `@Autowired`. So much so, in fact, that I debated whether to even include this section at all. Methods annotated with `@PostConstruct` will be invoked by Spring after the dependency graph has been resolved, and after all `@Autowired` methods have been invoked. Basically, it’s the last thing that will happen during the setup phase of the application. These methods cannot have parameters, and can have a return type of `void` (its return value will be thrown away so it doesn’t really matter). Let’s add this method to `DependencyPostSetupHandlers`:

```java
public static void invokePostConstructMethodsForBeans(List<Object> beanInstances) {
    for (Object instance : beanInstances) {
        for (Method method : instance.getClass().getMethods()) {
            boolean hasPostConstruct = getAnnotations(method::getAnnotations).contains(PostConstruct.class);

            if (hasPostConstruct) {
                if (method.getParameterCount() != 0) {
                    throw new IllegalStateException("Lifecycle method annotation @PostConstruct requires a no-arg method");
                }
                try {
                    method.invoke(instance);
                } catch (IllegalAccessException | InvocationTargetException e) {
                    throw new IllegalStateException("Could not invoke @Autowired method " + method, e);
                }
            }
        }
    }
}
```
<div class="caption">invokePostConstructMethodForBeans method in src/main/java/co/kenrg/hooke/application/DependencyPostSetupHandlers.java</div>

Then, let’s call it within the `run` method in `Application`, passing in the `allInstances` `Set` (same as with the `invokeAutowiredMethodsForBeans` method).

Now let’s add the `@PostConstruct` annotation to the `printMessage` method in `Component1`. Note that the `@PostConstruct` annotation is something that’s included in the `javax.annotation` package, and isn’t something we have to define ourselves. If we were to run our application now, we’d see the ‘Hello’…’Howdy!’ message print out twice, once for the `@PostConstruct` invocation of the `printMessage` method, and once for the code we added to `ExampleHookeApplication`'s main method. Now that we have the `@PostConstruct` functionality though, we can remove those 3 lines from the main method in our example application, so it should now just be
```java
public class ExampleHookeApplication {
    public static void main(String[] args) {
        HookeApplication.start(ExampleHookeApplication.class);
    }
}
```
So, that’s pretty much it for this step. The code is available at the project’s Github repo, and at the [`step-11-postconstruct-methods`](https://github.com/kengorab/hooke/tree/step-11-postconstruct-methods) tag.

## Conclusion

There have been a lot of topics covered in this 3-part series. We’ve built out an implementation of Spring’s dependency injection container from scratch! We’ve talked a lot about how Spring works internally, about reflection, and about lots of other things too.

I have 2 real goals with these “Let’s Write X” posts/series. The first one is that I just really enjoy reverse-engineering something and figuring out how it works. Considering all of the cases and design decisions that the original engineers had made all the way back when a project was first started is really interesting to me. The second goal is to help demystify how commonly-used tools/libraries work. I’ve always had at least a working knowledge of Spring, but after researching and building it out myself I feel like I’ve learned a lot. My hope is that you have too, and the next time you see a cryptic and pages-long stack trace from Spring you’ll maybe have some clearer picture of the error’s cause. If these types of posts interest you, you should stay tuned. I have more in the works as I attempt to rebuild pretty much all of the tools and libraries that I use on a daily basis.

That’s not to say that they will be exactly as performant or feature-complete. There is _so much_ that I’ve left out here in terms of functionality and performance (for example, I could probably be more optimized in the calls to the reflection API that I use), but that’s not the point. As it says in the README for this project, the goal was not to write a drop-in replacement for Spring, but merely to learn a bit about how it works. I hope you’ve learned something too!

Thank you for reading!


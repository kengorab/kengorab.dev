---
title: "Let's Write Spring! (Part 2)"
date: 2018-03-18T20:07:56-05:00
draft: true
tags: [java, spring, diy]
---

_This is a copy of a post, [originally written on Medium](https://medium.com/@ken.gorab/lets-write-spring-part-2-5837c48e3405). I've since rehosted it here._

---

If you haven’t read [Part 1](/posts/2018/03/lets-write-spring-1) you should definitely do that first.

Last time we got a basic implementation of Spring’s DI working, with `@Component` and `@Autowired` annotations working. In this post, we’re going to handle `@Configuration` classes and `@Bean` annotations on methods, as well as any issues that will now be able to arise around duplicate beans. Let’s get into it, picking up where we left off after Step 4.

## Step 5: Refactoring and @Configuration classes

In Step 2, I mentioned that the idea of iterating over something’s annotations (whether it’s a class, a field, or a method) is going to become a common enough pattern that it might merit being pulled out into some kind of helper function. In case you’re unclear on the code that I’m talking about, it’s the part in the `run` method in `Application` where we iterate over `clazz.getAnnotations()`, and the part in `getDependencyUnitForComponent` method in `DependencyCollectors` where we iterate over `field.getAnnotations()`. Let’s see if we can consolidate this a little bit into a utility.

```java
package co.kenrg.winter.util;

import com.google.common.collect.Maps;

import java.lang.annotation.Annotation;
import java.util.Map;
import java.util.function.Supplier;

public class Annotations {
    public static Annotations getAnnotations(Supplier<Annotation[]> annotationsSupplier) {
        Annotations annotations = new Annotations();
        for (Annotation annotation : annotationsSupplier.get()) {
            annotations.put(annotation.annotationType(), annotation);
        }
        return annotations;
    }

    private Map<Class, Annotation> map = Maps.newHashMap();

    private void put(Class c, Annotation a) {
        this.map.put(c, a);
    }

    public boolean contains(Class c) {
        return this.map.containsKey(c);
    }

    public void ifContains(Class c, Runnable fn) {
        if (this.contains(c)) {
            fn.run();
        }
    }

    public <T> T get(Class<T> clazz) {
        return (T) this.map.get(clazz);
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/util/Annotations.java</div>

This code enables us to transform this snippet of code from the `getDependencyUnitForComponent` method:
```java
Autowired autowiredAnn = null;

for (Annotation annotation : field.getAnnotations()) {
    if (annotation.annotationType().equals(Autowired.class)) {
        autowiredAnn = (Autowired) annotation;
        break;
    }
}
```
into this:
```java
Annotations annotations = getAnnotations(field::getAnnotations);
Autowired autowiredAnn = annotations.get(Autowired.class);
```
which (I think) encapsulates away a lot of stuff that we don’t need to care about. Similarly, this code in the `run` method in `Application`:
```java
for (Annotation annotation : clazz.getAnnotations()) {
    if (annotation.annotationType().equals(Component.class)) {
        componentClasses.add(clazz);
    }
}
```
becomes:
```java
Annotations annotations = getAnnotations(clazz::getAnnotations);
annotations.ifContains(
  Component.class,
  () -> componentClasses.add(clazz)
);
```
which I think also simplifies things a bit. Let’s run this to ensure that we didn’t somehow break something, and we should see the same message printed to the console that we saw at the end of Step 4: “Component1 says ‘Hello’; Component2 says ‘Howdy’”. Cool, onto `@Configuration` annotations!

```java
package co.kenrg.winter.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Configuration {
}
```
<div class="caption">src/main/java/co/kenrg/hooke/annotations/Configuration.java</div>

This shouldn’t be anything new; it’s pretty much exactly the same as the `@Component` annotation. And, much like the `@Component` annotation, we want to find all classes with the `@Configuration` annotation and store them in a `Set`. So, in `Application`, in addition to the `componentClasses` `Set`, we should now also have
```java
private final Set<Class> configClasses = Sets.newHashSet();
```
and within the for-loop iterating over all classes we should add the class to this set if it has the `@Configuration` annotation:
```java
Annotations annotations = getAnnotations(clazz::getAnnotations);
annotations.ifContains(
  Component.class,
  () -> componentClasses.add(clazz)
);
annotations.ifContains(
  Configuration.class,
  () -> configClasses.add(clazz)
);
```

Just for now, to gauge whether we’re correctly picking up on these configuration classes, let’s print out the `configClasses` `Set` at the end of the `run` method; let’s also add a configuration class to the example application in our tests:

```java
package co.kenrg.hooke.config;

import co.kenrg.hooke.annotations.Configuration;

@Configuration
public class Configuration1 {

}
```
<div class="caption">src/test/java/co/kenrg/hooke/config/Configuration1.java</div>

Running our application should print out a list containing the `Configuration1` class name, and then the same ‘Hello’…’Howdy’ output from earlier.

Okay, no huge changes here, but we’re headed towards something really cool. The code up until this point is available under the [`step-5-refactoring-and-config-classes`](https://github.com/kengorab/hooke/tree/step-5-refactoring-and-config-classes) tag on the project’s Github repo.

## Step 6: @Bean methods

One of the main use cases of `@Configuration` classes is to provide beans via methods annotated with `@Bean`. If you’ve never seen `@Bean` methods or were never exactly sure what they did, here’s an example:
```java
@Configuration
public class Configuration1 {

    @Bean
    public PrintStream printStream() {
        return System.out;
    }
}
```

When Spring starts up and finds the `@Configuration` classes, it will look for methods annotated with `@Bean`. It will then invoke those methods, and the return values will be inserted into the application context. A `@Bean` method is very similar to a class annotated with `@Component`; in fact, if there were a class `ABC` that _didn’t_ have a `@Component` annotation but you still wanted an instance of it in the application context, you could have a method like this:
```java
@Bean
public void ABC() {
    return new ABC();
}
```

and then another class can `@Autowire private ABC abc`. Therefore, it makes a lot of sense to think of a `@Bean` method as another thing that we can transform into a `DependencyUnit`. Let’s revisit our `DependencyCollectors` class and add a new method `getDependencyUnitsForBeanMethods` which will accept a list of configuration classes and return a list of `DependencyUnits`:
```java
public static List<DependencyUnit> getDependencyUnitsForBeanMethods(Set<Class> configClasses, DependencyInstanceGetter dependencyInstanceGetter) {
    List<DependencyUnit> units = Lists.newArrayList();
    for (Class configClass : configClasses) {
        final Object instance = instantiateClass(configClass);

        for (Method method : configClass.getMethods()) {
            boolean hasBean = getAnnotations(method::getAnnotations).contains(Bean.class);

            if (hasBean) {
                if (method.getReturnType() == Void.TYPE) {
                    throw new IllegalStateException("Method " + method + " given @Bean annotation, but returns void!");
                }

                DependencyUnit dependencyUnit = new DependencyUnit(
                    method.getName(),
                    method.getReturnType(),
                    Lists.newArrayList(),
                    () -> {
                        try {
                            List<Object> args = dependencyInstanceGetter.getDependencyInstances(Lists.newArrayList());
                            return method.invoke(instance, args.toArray());
                        } catch (IllegalAccessException | InvocationTargetException e) {
                            throw new IllegalStateException("Could not invoke method " + method, e);
                        }
                    }
                );
                units.add(dependencyUnit);
            }
        }
    }
    return units;
}

private static Object instantiateClass(Class configClass) {
    for (Constructor constructor : configClass.getConstructors()) {
        if (constructor.getParameterCount() == 0) {
            try {
                return constructor.newInstance();
            } catch (IllegalAccessException | InstantiationException | InvocationTargetException e) {
                throw new IllegalStateException("Could not invoke constructor " + constructor, e);
            }
        }
    }

    throw new IllegalStateException("No default no-args constructor for class: " + configClass);
}
```
<div class="caption">getDependencyUnitsForBeanMethods method in src/main/java/co/kenrg/hooke/application/DependencyCollectors.java</div>

Okay, let’s go over what we have here. First let’s start with the second method in the above snippet, `instantiateClass`. This is just a helper method to instantiate a class (as the name would suggest) which attempts to find a no-args constructor and invoke it; if there is no no-args constructor defined, we throw an exception (we can revisit this later on, but Spring makes a lot of assumptions about classes having no-args constructors, so I can justify this temporary decision with a minor appeal to authority).

So, back within `getDependencyUnitsForBeanMethods`, we iterate over all of the configuration classes passed in and obtain an instance of each one (it needs to be `final` because it’s referenced in the lambda later on). Then, we iterate over all of the methods in that class, and find the ones with the `@Bean` annotation. If it has the `@Bean` annotation but has a return type of `void`, then we throw an exception (since the return value of a method annotated with `@Bean` will be entered into the application context, this should be considered an invalid state). Then we construct a `DependencyUnit` instance to represent that `@Bean` method, passing in a lambda which returns the result of invoking the method, as well as an empty list of dependencies. What would it mean for a `@Bean` method to have dependencies? We’ll come back to that, don’t worry.

The last thing to do before we can test that this works is to ensure that these `DependencyUnit`s are getting added to the dependency graph that we build. So, our `run` method in `Application` should now look like this:
```java
public void run() throws IOException {
    String packageName = this.appClass.getPackage().getName();
    ClassLoader classLoader = this.appClass.getClassLoader();
    ImmutableSet<ClassInfo> allClasses = ClassPath.from(classLoader).getTopLevelClassesRecursive(packageName);
    for (ClassPath.ClassInfo classInfo : allClasses) {
        Class<?> clazz = classInfo.load();

        Annotations annotations = getAnnotations(clazz::getAnnotations);
        annotations.ifContains(Component.class, () -> componentClasses.add(clazz));
        annotations.ifContains(Configuration.class, () -> configClasses.add(clazz));
    }

    List<DependencyUnit> dependencies = Lists.newArrayList();

    dependencies.addAll(getDependencyUnitsForBeanMethods(configClasses, applicationContext::getBeansOfTypes));

    for (Class componentClass : componentClasses) {
        dependencies.add(getDependencyUnitForComponent(componentClass, applicationContext::getBeansOfTypes));
    }

    Graph<DependencyUnit> graph = buildDependencyGraph(dependencies);
    resolveDependencyGraph(graph, applicationContext::insert);

    System.out.println(configClasses);
}
```
<div class="caption">run method in src/main/java/co/kenrg/hooke/application/Application.java</div>

So how can we verify that this works? Well, if we change our example application to remove the `@Component` annotation on our `Component2` class and add a `@Bean` method into our configuration class which returns an instance of `Component2` we should see the same output we’ve been seeing when we run it.
```java
// src/test/java/co/kenrg/hooke/components/Component1.java
@Component
public class Component1 {
    @Autowired private Component2 component2;

    public void printMessage() {
        System.out.printf("Component1 says 'Hello'; Component2 says '%s'\n", component2.getMessage());
    }
}


// src/test/java/co/kenrg/hooke/components/Component2.java
public class Component2 {
    public String getMessage() {
        return "Howdy";
    }
}


// src/test/java/co/kenrg/hooke/config/Configuration1.java
@Configuration
public class Configuration1 {

    @Bean
    public Component2 component2() {
        return new Component2();
    }
}


// src/test/java/co/kenrg/hooke/ExampleHookeApplication.java
public class ExampleHookeApplication {
    public static void main(String[] args) {
        HookeApplication.start(ExampleHookeApplication.class);

        ApplicationContext ctx = HookeApplication.getApplicationContext();
        Component1 component1 = ctx.getBeanOfType(Component1.class);
        component1.printMessage();
    }
}
```

This should print out our familiar ‘Hello’…’Howdy’ message, confirming that everything is in working order! So, let’s come back to that question from a moment ago — “What would it mean for a `@Bean` method to have dependencies?”. Well, let’s say that `Component2` needed to have something passed into its constructor, let’s say an instance of `Component3`. So we have our modified `Component2` class, and our new `Component3` class as such:
```java
// src/test/java/co/kenrg/hooke/components/Component2.java
public class Component2 {
    private Component3 component3;

    public Component2(Component3 component3) {
        this.component3 = component3;
    }

    public String getMessage() {
        return component3.getGreeting();
    }
}


// src/test/java/co/kenrg/hooke/components/Component3.java
@Component
public class Component3 {
    public String getGreeting() {
        return "Howdy";
    }
}
```

As contrived and ridiculous as this example is, it illustrates the situation pretty well. I’m also aware that we should just make `Component2` a `@Component`, that way it would be able to `@Autowire private Component3 component3` instead of receiving it as a constructor argument, but let’s say `Component2` is from a library that we don’t have access to and therefore can’t modify (this is a common use case for `@Bean` methods).

So our application context will have an instance of `Component3` but how does the `@Bean` method obtain that instance when it’s constructing the `new Component2()`? Well, `@Bean` methods can have arguments, the values of which will come from the application context. This is what it means for a `@Bean` method to have dependencies. And thankfully, there is very little that we need to change in order to pull this off. The first thing we’ll need is a little helper method in `DependencyCollectors`:
```java
private static List<Pair<String, Class>> getDependenciesFromParameters(Parameter[] parameters) {
    List<Pair<String, Class>> dependencies = Lists.newArrayList();
    for (Parameter parameter : parameters) {
        dependencies.add(Pair.of(null, parameter.getType()));
    }
    return dependencies;
}
```
<div class="caption">getDependenciesFromParameters method in src/main/java/co/kenrg/hooke/application/DependencyCollectors.java</div>

This looks simple for now, but it’ll get complex enough later that it makes sense to have in its own method. One thing of note is that the `Pair` getting returned always has `null` as its left-hand value. Why? This is where I come forward and say that I jumped the gun a bit. It was totally not necessary to attach names to dependencies just yet, but I was thinking too far ahead. I apologize, but just ignore it for now and I promise we’ll come back to it when we handle `@Qualifier` annotations. Anyway, if we modify the `getDependencyUnitsFromBeanMethod` method to have this code:
```java
if (hasBean) {
    if (method.getReturnType() == Void.TYPE) {
        throw new IllegalStateException("Method " + method + " given @Bean annotation, but returns void!");
    }

    final List<Pair<String, Class>> dependencies = getDependenciesFromParameters(method.getParameters());
    List<Class> dependencyClasses = dependencies.stream().map(Pair::getRight).collect(toList());

    DependencyUnit dependencyUnit = new DependencyUnit(
        method.getName(),
        method.getReturnType(),
        dependencies,
        () -> {
            try {
                List<Object> args = dependencyInstanceGetter.getDependencyInstances(dependencyClasses);
                return method.invoke(instance, args.toArray());
            } catch (IllegalAccessException | InvocationTargetException e) {
                throw new IllegalStateException("Could not invoke method " + method, e);
            }
        }
    );
    units.add(dependencyUnit);
}
```
<div class="caption">getDependencyUnitsFromBeanMethod method in src/main/java/co/kenrg/hooke/application/DependencyCollectors.java</div>

we should have a working solution. The only difference here is that we obtain our `@Bean` method’s dependencies from the method’s parameters now (the `List<Pair<String, Class>>`). Due to a decision I made earlier on to have the `DependencyUnit` accept a `List<Pair<String, Class>>` as its `dependencies` field and have the `DependencyInstanceGetter::getDependencyInstances` method accept a `List<Class>`, we need to do an additional stream here. This will be rectified later on, again when we handle the `@Qualifier` annotation.

So if we modify the configuration class’s `@Bean` method to look like this:
```java
// src/test/java/co/kenrg/hooke/config/Configuration1.java
@Configuration
public class Configuration1 {

    @Bean
    public Component2 component2(Component3 component3) {
        return new Component2(component3);
    }
}
```
and run our example application, we should see again that same ‘Hello’…’Howdy’ message we’re so used to at this point. Excellent! Everything is working and we have implemented `@Bean` methods and `@Configuration` classes! The code thus far is available at the [`step-6-bean-methods`](https://github.com/kengorab/hooke/tree/step-6-bean-methods) from the project’s Github repo.

## Step 7: @Qualifier and Duplicate Beans

So I’ve mentioned `@Qualifiers` a few times now, and I think it’s time to tackle them. For those of you that don’t know the use case for `@Qualifiers`, here’s a very small example which illustrates the problem that they solve:
```java
// Config.java
@Configuration
public class Config {

    @Bean
    public String apiKey() {
        return "asodifhijqkbewfxcv";
    }

    @Bean
    public String databaseUrl() {
        return "http://localhost:5432";
    }
}


// SomeDao.java
@Component
public class SomeDao {
    @Autowired private String databaseUrl; // Which String will be @Autowired?
    
    /*
      ... dao logic here
    */
}
```

In the `Config` class, we declare 2 `@Bean` methods which produce the same type (`String`). Both of those instances will be put in the application context without any issue, but the `@Autowired` annotation in our `SomeDao` class is essentially asking the application context, “can I please have an instance of `String`?", a question that the application context cannot answer with just 1 single instance. To solve this, the `@Qualifier` annotation can be used to specify the intended name of the Bean that you want:
```java
@Component
public class SomeDao {
    @Autowired @Qualifier("databaseUrl") private String databaseUrl;
    
    /*
      ... dao logic here
    */
}
```

A `@Bean` method will cause an instance of its return type to be inserted into the application context along with its name (the name of the method). A `@Component` annotation will cause an instance of that class to be inserted along with its name (the name of the class). Using `@Qualifier` alongside `@Autowired` is like asking the application context, “can I please have an instance of String whose name is "databaseUrl"?” This is what we’ll implement now, and it’s why (as some of you may have been wondering) I’ve been using `Pair<String, Class>` in cases where simple `Class` would have sufficed. Right now, running our ongoing example with the following changes will break due to a duplicate key exception; after we’re done it will work perfectly:
```java
// Component3.java
@Component
public class Component3 {
    @Autowired private String greeting;

    public String getGreeting() {
        return greeting;
    }
}


// Configuration1.java
@Configuration
public class Configuration1 {

    @Bean
    public String superSecretApiKey() {
        return "asoiudhfjkn";
    }

    @Bean
    public String greeting() {
        return "Howdy";
    }

    @Bean
    public Component2 component2(Component3 component3) {
        return new Component2(component3);
    }
}
```
The first offender is in the `buildDependencyGraph` method of `DependencyGraph`, but before we dive in there let’s talk about our strategy a bit. Right now the `dependencies` field on `DependencyUnit` is a `List<Pair<String, Class>>` where the left-hand value of the `Pair` is the dependency’s name and the right-hand value is the dependency’s class. This represents all of the entries in the application context that this unit will “ask for” when it builds itself. Let’s make the decision that if the left-hand value is null that means that we don’t necessarily care what the name of the entry is — we just want the entry that matches the right-hand side’s class. If the left-hand value is not null, then we will ask by name and by class.

So first let’s fix our earlier implementation of the `getDependencyUnitForComponentClass` method within `DependencyCollectors`, which doesn’t really abide by that strategy (my bad, but mistakes happen). Let’s change that for-loop to be:
```java
for (Field field : componentClass.getDeclaredFields()) {
    Annotations annotations = getAnnotations(field::getAnnotations);
    Autowired autowiredAnn = annotations.get(Autowired.class);

    if (autowiredAnn != null) {
        fields.put(field, Pair.of(null, field.getType()));
    }
}
```
<div class="caption">loop in the getDependencyUnitForComponentClass method</div>

Note the `null` in the left-hand value of the `Pair`. This should cause nothing to break. Good. That was the only thing that needed to change in order to follow our strategy.

Now let’s revisit the `buildDependencyGraph` method of `DependencyGraph`. What we want to do now is, as we iterate over each `DependencyUnit`'s dependencies, if the dependency’s `Pair` has a non-null left-hand value, we want to see if there exists a `DependencyUnit` that matches the name; if not, then that’s an invalid state since the graph cannot be resolved successfully. If the dependency’s `Pair` has a null left-hand value, then we simply want to find the `DependencyUnit` that matches the type; however there’s a new twist here. We don’t simply have a `Map<Class, DependencyUnit>` anymore, since there could be multiple `DependencyUnit` instances that produce the same `Class`. We _could_ construct a `Map<Class, Collection<DependencyUnit>>` ourselves, but there are wonderful `Collectors` for `Multimaps` in guava, so I’ll just use that instead. Here’s our modified `buildDependencyGraph` method in full:
```java
public static Graph<DependencyUnit> buildDependencyGraph(List<DependencyUnit> dependencyUnits) {
    Map<String, DependencyUnit> namedDependencyUnitMap = dependencyUnits.stream()
        .filter(unit -> unit.name != null)
        .collect(toMap(unit -> unit.name, Function.identity()));
    Map<Class, Collection<DependencyUnit>> dependencyUnitMap = dependencyUnits.stream()
        .collect(toImmutableListMultimap(unit -> unit.providesClass, Function.identity()))
        .asMap();

    MutableGraph<DependencyUnit> graph = GraphBuilder.directed().build();
    for (DependencyUnit unit : dependencyUnits) {
        graph.addNode(unit);
        for (Pair<String, Class> dependency : unit.dependencies) {
            String name = dependency.getLeft();
            Class type = dependency.getRight();

            DependencyUnit depUnit;
            if (name != null) {
                if (namedDependencyUnitMap.containsKey(name)) {
                    depUnit = namedDependencyUnitMap.get(name);
                } else {
                    throw new IllegalStateException("No beans available with name " + name);
                }
            } else {
                Collection<DependencyUnit> possibleDepsByType = dependencyUnitMap.get(type);
                if (possibleDepsByType.isEmpty()) {
                    throw new IllegalStateException("No beans available of type " + type);
                } else if (possibleDepsByType.size() != 1) {
                    throw new IllegalStateException("Multiple beans available of type " + type + "; try using a @Qualifier");
                } else {
                    depUnit = possibleDepsByType.iterator().next();
                }
            }

            graph.putEdge(unit, depUnit);
        }
    }

    return graph;
}
```
<div class="caption">buildDependencyGraph method in src/main/java/co/kenrg/hooke/application/graph/DependencyGraph.java</div>

It’s only slightly different — the main idea is the same. Now though, there’s additional logic around name-based dependencies as well as additional error handling. Running this code against our example application (not the sample application that I had included above where `Component3` gets its greeting via `@Autowire`) should produce the same old result. Good, we’re not breaking stuff; there’s no way our new code path would be traveled since the left-hand value of those Pairs are always `null`. Let’s change that. Let’s add a `@Qualifier` annotation:
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Qualifier {
    String value() default "";
}
```
<div class="caption">src/main/java/co/kenrg/hooke/annotations/Qualifier.java</div>

This one is only slightly different from the other annotations we’ve made thus far, the main difference being the `String value() default "";` bit. When using this annotation, we can pass the name of the `Bean` that we want into it by doing `@Qualifier("beanNameHere")`. That `"beanNameHere"` string is accessible via the annotation’s `value()` method. We’ll look at this in greater detail below. Let’s take a look at that for-loop in the `getDependencyUnitForComponentClass` method in `DependencyCollectors`, and make a small change:
```java
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
```
<div class="caption">getDependencyUnitForComponentClass method in src/main/java/co/kenrg/hooke/application/DependencyCollectors.java</div>

If the field has `@Autowired` annotation and it has a `@Qualifier` annotation with a non-empty value, its name comes from that annotation’s value; otherwise its name is null, like it had been before. If we change `Component3` in our working example to have
```java
@Autowired @Qualifier("greeting") String greeting;
```
now (adding the `@Qualifier`), we don’t get that duplicate key exception we saw earlier, but our output is not what we expect:
```
Component1 says 'Hello'; Component2 says 'asoiudhfjkn'
```
_(If it is correct the first time you run it, try running it again. Eventually it will look like this; it’s non-deterministic)_

It looks like nothing is respecting our name field. But I thought we already fixed that? Didn’t we solve this in our `DependencyGraph` class? Actually no — the code we changed in the `buildDependencyGraph` method only ensured that the graph was capable of resolving correctly, in order to ensure that the `DependencyUnits` correctly instantiate the class that they provide we need to make another change. Within the lambdas that we pass to the `DependencyUnit` constructor, we are calling upon the `DependencyInstanceGetter` passed as an argument to `getDependencyUnitForComponent` and `getDependencyUnitsForBeanMethods`, which right now has a signature of:
```java
List<Object> getDependencyInstances(List<Class> dependencies);
```
when it really should be
```java
List<Object> getDependencyInstances(
  List<Pair<String, Class>> dependencies
);
```

to account for our possibly-named dependencies. Let’s do that quick refactor in `DependencyInstanceGetter` and `DependencyCollectors`, making sure to remove the now-unneeded variables that we had previously had in the two `getDependencyUnitFor\*` methods. Then we get to the error within `Application`, since the method from `ApplicationContext` that we’re currently passing in to satisfy the `DependencyInstanceGetter` parameter no longer matches the signature. Let’s change up our `ApplicationContext` class a little bit:
```java
public class ApplicationContext {
    private final Map<Class, Object> allBeans = Maps.newHashMap();
    private final Map<String, Object> namedBeans = Maps.newHashMap();

    public void insert(Class clazz, @Nullable String name, Object instance) {
        allBeans.put(clazz, instance);
        if (name != null) {
            namedBeans.put(name, instance);
        }
    }

    public List<Object> getBeans(List<Pair<String, Class>> beans) {
        List<Object> values = Lists.newArrayList();
        for (Pair<String, Class> pair : beans) {
            Object instance;
            String errMsg;
            if (pair.getLeft() == null) {
                instance = allBeans.get(pair.getRight());
                errMsg = "No bean available of type " + pair.getRight();
            } else {
                instance = namedBeans.get(pair.getLeft());
                errMsg = "No bean available for name " + pair.getLeft();
            }

            if (instance == null) {
                throw new IllegalStateException(errMsg);
            }

            values.add(instance);
        }

        return values;
    }

    public <T> T getBeanOfType(Class<T> clazz) {
        return (T) allBeans.get(clazz);
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/ApplicationContext.java</div>

There are a couple of new things here. The first is the `getBeans` method, and the second is the switch from having a field `Map<Class, Object> applicationContext` to having 2 maps, one which represents all of the beans and one which represents only the named ones. When getting instances of beans out of the application context based on the `Pair` the `getBeans` method returns, we need to check whether the name (the left-hand value of the `Pair`) is `null` or not. If it’s not, attempt to get the named bean instance out of the `namedBeans` map; otherwise, get the instance from the `allBeans` map. Running our application now should give us the same ‘Hello’…’Howdy’ message we’ve come to expect, and this time it’s deterministic.

If we wanted to, we could also add a `getNamedBean` method to our `ApplicationContext` (I omitted that here though, since it’d only be used within our main method and I didn’t feel like implementing it. It’d be super simple though if you wanted to give it a shot!). Just to verify that everything is still working, let’s try to remove that `@Qualifier(“greeting”)` from the greeting field in `Component3`; we should see this error message:
```
Exception in thread "main" java.lang.IllegalStateException: Multiple beans available of type class java.lang.String; try using a @Qualifier
```

As always, the code up until now has been pushed to the Github repo, and is available under the tag [`step-7-qualifier-and-duplicate-beans`](https://github.com/kengorab/hooke/tree/step-7-qualifier-and-duplicate-beans).

## To Be Continued…

There was a good amount of stuff we did in part 2; prior to starting this we only had the `@Component` and `@Autowired` annotations and now we have `@Configuration`, `@Bean`, and `@Qualifier`. Previously we could only have one bean in our application context per type and now we can have many. Things are coming along pretty well!

I don’t think there’s a whole lot more that needs to be added to this for the series to be considered “complete”. In the next (and probably final) post, I’ll add support to Hooke for `@Qualifier` annotations on `@Bean` methods, constructor-based DI, `@Autowired` on methods, and the `@PostConstruct` annotation. If that interests you, continue on to [Part 3](/posts/2018/03/lets-write-spring-3). After that, I think we’ll have built out all the basic functionality that Spring provides for dependency injection. If you haven’t already, check out the project’s Github page, and thank you so much for reading!


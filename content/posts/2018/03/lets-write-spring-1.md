---
title: "Let's Write Spring! (Part 1)"
date: 2018-03-12T20:07:56-05:00
draft: true
tags: [java, spring, diy]
---

_This is a copy of a post, [originally written on Medium](https://medium.com/@ken.gorab/lets-write-spring-part-1-1b03d04b2cee). I've since rehosted it here._

---

In my previous article ([Let’s Write Redux!](/posts/2018/03/lets-write-redux)), I walked through most of the API of the javascript state management library Redux. I started with the API as it’s laid out in the documentation and guides, and then built out all of those functions. My goal there was to demystify some of the things that Redux does, in an effort to make it feel less like magic. This time, I’ll try to do the same with Spring, a Dependency Injection library for JVM languages like Java or Kotlin.

## What is Spring?

You’ve maybe seen Spring used in the context of web applications, where it acts as the framework around which a service/backend is built. You’ve probably seen the `@Component` and `@Autowired` annotations before, sprinkled throughout various classes and methods. But what exactly _is_ Spring?

Spring is a Dependency Injection library. You write all of your classes and their dependencies (read: other classes that your class makes use of in its methods), then use Spring’s annotations to declaratively describe how they’re all related together. Then, in the `main` method of your application, you start the application and Spring sets everything up for you. As Spring has gained in popularity, many additional libraries have been written to simplify things like responding to web requests, connecting to a Postgres or Mongo database, implementing security rules, and much more. All of those libraries `make use` of Spring and build upon it, but are not Spring itself; it is, at its heart, _just_ a Dependency Injection library.

Let’s take a look at what a simple, three-file Spring application might look like:

```java
// co/kenrg/example/TestApplication.java
package co.kenrg.example;

public class TestApplication {
  public static void main(String[] args) {
    SpringApplication.start(TestApplication.class);
  }
}

// co/kenrg/example/components/Component1.java
package co.kenrg.example.components;

@Component
public class Component1 {
  @Autowired private Component2 component2;
  
  public void printMessage() {
    System.out.printf("Component1 says 'hello'; Component2 says '%s'\n", component2.getMessage());
  }
}

// co/kenrg/example/components/Component2.java
package co.kenrg.example.components;

@Component
public class Component2 {
  public String getMessage() {
    return "greetings!";
  }
}
```

In this example, there are 3 files/classes: `TestApplication.java`, `Component1.java`, and `Component2.java`. The `TestApplication` class houses the `main` method, which kicks off the Spring application by passing in the `TestApplication.class`. From there, Spring will find all the classes in the package that are annotated with `@Component` (there are others, but let’s just leave it at that one for now) and instantiate it. For each of those component classes, it looks at any instance variables annotated with `@Autowired` and instantiates _those_, and so on and so on until all classes have all of their dependencies initialized.
This is just the tip of the iceberg, and there will be a lot to talk about and a _lot_ to add to this, but let’s dive in with step 1.

## Step 1: Gathering @Components

Let’s get this rolling with the first step, gathering all of the classes annotated with `@Component`. I’m going to be naming my Spring rewrite [Hooke](https://en.wikipedia.org/wiki/Hooke%27s_law), so instead of the entrypoint being `SpringApplication` we have `HookeApplication`:

```java
public class HookeApplication {
    private static Application INSTANCE;

    public static void start(Class appClass) {
        try {
            HookeApplication.INSTANCE = new Application(appClass);
            INSTANCE.run();
        } catch (IOException e) {
            throw new IllegalStateException(e);
        }
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/HookeApplication.java</div>

This doesn’t really do anything except instantiate the underlying `Application` class (which is where all of the real dirty work will happen). I wanted to keep the entrypoint pretty shallow, and defer nearly all of the setup logic to the `Application` class; exceptions will be caught here too at the highest level.

Before we look into the `Application` class though, let’s take a look at this project’s `build.gradle` file:

```java
group 'co.kenrg'
version '1.0'

apply plugin: 'java'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    compile 'com.google.guava:guava:23.6-jre'
    compile 'org.apache.commons:commons-lang3:3.7'

    testCompile 'junit:junit:4.12'
}
```

Nothing too complex going on here: applying the `java` plugin (which is what gives the project its `src/main/java` structure), setting the source compatibility to Java 8 (since we’ll be using some features like streams and lambdas), and pulling in the apache commons and google guava libraries. These libraries are pretty ubiquitous in java projects I find, and complement the Collections API very nicely. I’ll mention when I’m using classes from there, so you can see the breadth of usefulness the library contains.

Let’s take a moment to think about what needs to happen in the `Application` class’s `run` method now. In `HookeApplication` we invoke the `Application` constructor and pass in the class that represents the app. Using that class, we need to find all classes within that class’s package. Then, we can find the ones that have the `@Component` annotation. I guess let’s make that annotation first, since there’s really nothing to it:

```java
package co.kenrg.hooke.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {

}
```
<div class="caption">src/main/java/co/kenrg/hooke/annotations/Component.java</div>

The `@Target(ElementType.TYPE)` annotation means that this annotation can only be applied to classes, interfaces, and enums; trying to add the `@Component` annotation to a method parameter, for example, will cause a compiler error. The `@Retention(RetentionPolicy.RUNTIME)` annotation means that the annotation information will be output in the `.class` file, but will also be available at runtime (this is different than `RetentionPolicy.CLASS`, which is the default behavior).

So, the `run` method of `Application` must look at all classes in the package, and find those annotated with the `@Component` annotation, and keep track of them somehow. Since we’re not ready to do anything more with them other than simply collect them, let’s print them out at the end of the run method:

```java
package co.kenrg.hooke.application;

import java.io.IOException;
import java.lang.annotation.Annotation;
import java.util.Set;

import co.kenrg.hooke.annotations.Component;
import com.google.common.collect.ImmutableSet;
import com.google.common.collect.Sets;
import com.google.common.reflect.ClassPath;
import com.google.common.reflect.ClassPath.ClassInfo;

public class Application {
    private final Class appClass;
    private final Set<Class> componentClasses = Sets.newHashSet();

    public Application(Class appClass) {
        this.appClass = appClass;
    }

    public void run() throws IOException {
        String packageName = this.appClass.getPackage().getName();
        ClassLoader classLoader = this.appClass.getClassLoader();
        ImmutableSet<ClassInfo> allClasses = ClassPath.from(classLoader).getTopLevelClassesRecursive(packageName);
        for (ClassPath.ClassInfo classInfo : allClasses) {
            Class<?> clazz = classInfo.load();

            for (Annotation annotation : clazz.getAnnotations()) {
                if (annotation.annotationType().equals(Component.class)) {
                    componentClasses.add(clazz);
                }
            }
        }

        System.out.println(componentClasses);
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/Application.java</div>

Note that everything whose import begins with `com.google.common` is from the google-guava library we pulled in via our `build.gradle` file; it provides a very nice way to create collections (like `Sets` and `Maps`) and also provides much cleaner ways of interacting with `ClassLoaders`.

You probably have noticed that we’re using reflection here. In case you’re not familiar with what that is, the reflection APIs in Java allow you to interact with and obtain data about your code itself; in the `run` method so far, we’re loading the information about a class and searching through all of the annotations that the class has. Reflection is pretty cool and super powerful, and we will be using it quite often as we build Hooke, but it’s also pretty dangerous and not very performant. This is fine though, because although Hooke (much like Spring) _may_ take a little bit on the longer side to start up due to all of these reflection API calls, it will not impact the actual runtime of the code at all, since the reflection only happens at the very beginning of the application’s runtime.

I think the only thing remaining is some test code to see if this is actually working. I will not be going over unit tests for this (though I _will_ be writing it in such a way that it’s easily unit-testable), so the only code in the `src/test` directory will be an example Hooke application, and it will grow in complexity as we continue to add features to it.

```java
// src/test/java/co/kenrg/hooke/components/Component1.java
package co.kenrg.hooke.components;

import co.kenrg.hooke.annotations.Component;

@Component
public class Component1 {
}


// src/test/java/co/kenrg/hooke/components/Component1.java
package co.kenrg.hooke.components;

import co.kenrg.hooke.annotations.Component;

@Component
public class Component2 {
}


// src/test/java/co/kenrg/hooke/components/ExampleHookeApplication.java
package co.kenrg.hooke;

public class ExampleHookeApplication {
    public static void main(String[] args) {
        HookeApplication.start(ExampleHookeApplication.class);
    }
}
```

Running the main method in `ExampleHookeApplication` causes the names of `Component1` and `Component2` to be output to the console, proving that step 1 is complete! [Here is the github repo](https://github.com/kengorab/hooke/tree/step-1-gathering-components) for this project, to which I’ll be committing and tagging after every step. You can look through the state of the project so far by checking out the `step-1-gathering-components` tag.

## Step 2: @Autowired instance variables

Here’s when we start thinking about how the `@Component` classes relate to each other. A `@Component` class that has `@Autowired` annotations on its instance variables is said to have a dependency on the type of that instance variable (again, it will get more involved than that later on, but let’s start with the basics). It becomes necessary then to build some kind of data structure to keep track of which components are dependencies of other components. This data structure is a `Graph`, the nodes of which are what I’ll call `DependencyUnit`s.

```java
package co.kenrg.hooke.application.graph;

import java.util.List;

import org.apache.commons.lang3.tuple.Pair;

public class DependencyUnit {
    public final String name;
    public final Class providesClass;
    public final List<Pair<String, Class>> dependencies;

    public DependencyUnit(String name, Class providesClass, List<Pair<String, Class>> dependencies) {
        this.name = name;
        this.providesClass = providesClass;
        this.dependencies = dependencies;
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/graph/DependencyUnit.java</div>

This class contains 3 fields: `providesClass` which represents the class that this `DependencyUnit` will provide once all its dependencies are met; `dependencies` which is a `Pair` of the dependency’s name and its `Class` (note that `Pair` comes from apache commons); and `name` which, for now, will just be the name of the class (but that’ll change later on when we add things like `@Qualifier`).

The other thing we’ll need of course, is the `@Autowired` annotation itself:

```java
package co.kenrg.hooke.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {

}
```
<div class="caption">src/main/java/co/kenrg/hooke/annotations/Autowired.java</div>

This annotation is very similar to the `@Component` annotation, the only difference being that the `@Target(ElementType.FIELD)` means that this annotation is only valid on fields whereas the `@Component` annotation was only valid on classes/interfaces/enums.

Next, it’s time to write a method that, given a class annotated with `@Component`, it will return the `DependencyUnit` for that component. We will need to look through all fields of that component class, find any that have the `@Autowired` annotation on it, and store it. Then we can return the `DependencyUnit`. Let’s see what that looks like:

```java
package co.kenrg.hooke.application;

import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.util.List;
import java.util.Map;

import co.kenrg.hooke.annotations.Autowired;
import co.kenrg.hooke.application.graph.DependencyUnit;
import com.google.common.collect.Lists;
import com.google.common.collect.Maps;
import org.apache.commons.lang3.tuple.Pair;

public class DependencyCollectors {

    public static DependencyUnit getDependencyUnitForComponent(Class componentClass) {
        List<Pair<String, Class>> dependencies = Lists.newArrayList();

        Map<Field, Pair<String, Class>> fields = Maps.newHashMap();
        for (Field field : componentClass.getDeclaredFields()) {
            Autowired autowiredAnn = null;

            for (Annotation annotation : componentClass.getAnnotations()) {
                if (annotation.annotationType().equals(Autowired.class)) {
                    autowiredAnn = (Autowired) annotation;
                    break;
                }
            }

            if (autowiredAnn != null) {
                Class<?> fieldType = field.getType();
                String fieldName = field.getName();

                Pair<String, Class> depPair = Pair.of(fieldName, fieldType);
                fields.put(field, depPair);
            }
        }

        dependencies.addAll(fields.values());

        return new DependencyUnit(componentClass.getName(), componentClass, dependencies);
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/DependencyCollectors.java</div>

Again, we use reflection to get the fields on a particular class, and iterate over its annotations to see if there’s an `@Autowired` present. This business of iterating over an array of annotations is going to be pretty common and will probably merit being pulled out into a helper method, but that can happen a bit later on. Aside from some calls to the reflection API though, this should be pretty straightforward. The `Pair` that gets created based on the each field’s name and type becomes an entry in the dependencies for the `DependencyUnit`. It might not be obvious why `fields` is a `Map` instead of simply a `Set`, or why it’s necessary to preserve the `Field` in that `Map`, but it’ll become more apparent later on.

So now, the only thing left is to get a `DependencyUnit` for each class in `componentClasses` within the `run` method in 1Application`:

```java 
public void run() throws IOException {
    String packageName = this.appClass.getPackage().getName();
    ClassLoader classLoader = this.appClass.getClassLoader();
    ImmutableSet<ClassInfo> allClasses = ClassPath.from(classLoader).getTopLevelClassesRecursive(packageName);
    for (ClassPath.ClassInfo classInfo : allClasses) {
        Class<?> clazz = classInfo.load();

        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().equals(Component.class)) {
                componentClasses.add(clazz);
            }
        }
    }

    for (Class componentClass : componentClasses) {
        DependencyUnit dependencyUnit = getDependencyUnitForComponent(componentClass);
        System.out.println(dependencyUnit);
    }
}
```

<div class="caption">run method in src/main/java/co/kenrg/hooke/application/Application.java</div>

I omitted the `equals`, `hashCode`, and `toString` overrides in the `DependencyUnit` class for brevity, but once we modify our example application so that `Component1` has an `@Autowired private Component2 component2` field, we should see the expected output!

The work up until this point is available under [`step-2-autowired-instance-variables`](https://github.com/kengorab/hooke/tree/step-2-autowired-instance-variables) tag on the project’s Github page.

## Step 3: Building & Resolving the Dependency Graph

So, what do we have so far? For every class annotated with `@Component` we have a `DependencyUnit` that stores all of that class’s dependencies, as declared via the `@Autowired` annotation. That might sounds like a lot, but we don’t yet have any instances of those classes, and the instance variables have not been assigned to anything! Now is when we start thinking about the dependency graph.

There are 2 parts to this: building the dependency graph, and then resolving that graph. Neither is very difficult, especially when you use tools provided by the google-guava library. Let’s write a method that, given a list of `DependencyUnits` returns a `Graph`, where the nodes are `DependencyUnits`, and nodes A and B are connected if B depends on A (aka, class B has an `@Autowired` instance variable of type A).

```java
package co.kenrg.hooke.application.graph;

import static java.util.stream.Collectors.toMap;

import java.util.List;
import java.util.Map;
import java.util.function.Function;

import com.google.common.graph.Graph;
import com.google.common.graph.GraphBuilder;
import com.google.common.graph.MutableGraph;
import org.apache.commons.lang3.tuple.Pair;

public class DependencyGraph {
    public static Graph<DependencyUnit> buildDependencyGraph(List<DependencyUnit> dependencyUnits) {
        Map<Class, DependencyUnit> dependencyUnitMap = dependencyUnits.stream()
            .collect(toMap(unit -> unit.providesClass, Function.identity()));

        MutableGraph<DependencyUnit> graph = GraphBuilder.directed().build();
        for (DependencyUnit unit : dependencyUnits) {
            graph.addNode(unit);
            for (Pair<String, Class> dependency : unit.dependencies) {
                DependencyUnit depUnit = dependencyUnitMap.get(dependency.getValue());
                graph.putEdge(unit, depUnit);
            }
        }

        return graph;
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/graph/DependencyGraph.java</div>

The first thing we do is extrapolate the input list of dependencies into a `Map`, to facilitate easier (and constant-time) lookups in the loops further on. We then build our `MutableGraph<DependencyUnit>`, which comes from the google-guava library. For each `DependencyUnit` we add a node into the graph, and iterate over that `DependencyUnit`'s dependencies and add an edge for each one. The `MutableGraph` class takes care of accidental duplicates, and really just makes this whole thing a lot easier.

So now we have our graph, a relationship of how all of these classes fit together. But that doesn’t really do us any good until we can finally start instantiating them! Next comes resolving the graph.

But first we need to change `DependencyUnit` a little bit. What I’d really like to be able to do is for each `DependencyUnit` to have not only the information about the class it provides and the dependencies needed to instantiate it fully, but also the means by which to do it. It’d be a really good separation of concerns if the code that’s resolving the graph didn’t actually know anything about what each `DependencyUnit` was, but instead just had to call some method on each one to obtain the necessary instance. Let’s add a field to the `DependencyUnit` class:

```java
package co.kenrg.winter.application.graph;

import org.apache.commons.lang3.tuple.Pair;

import java.util.List;
import java.util.function.Supplier;

public class DependencyUnit {
    public final String name;
    public final Class providesClass;
    public final List<Pair<String, Class>> dependencies;
    private final Supplier<Object> instanceProvider;

    public DependencyUnit(String name, Class providesClass, List<Pair<String, Class>> dependencies, Supplier<Object> provider) {
        this.name = name;
        this.providesClass = providesClass;
        this.dependencies = dependencies;
        this.instanceProvider = provider;
    }
    
    public Object getInstance() {
        return instanceProvider.get();
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/graph/DependencyUnit.java</div>

The only difference here is the addition of the last parameter to the constructor, a `Supplier<Object>` provider and a helper method which invokes that function. So, if we have an instance unit of `DependencyUnit`, all that the downstream code will have to do is say `unit.getInstance()`, to obtain an instance. I think this is a pretty cool pattern, and a good application of Java 8 closures. What does this look like in our `getDependencyUnitForComponent` method in `DependencyCollectors` though? Let’s take a look at the return statement:

```java
return new DependencyUnit(
    componentClass.getName(),
    componentClass,
    dependencies,
    () -> {
        try {
            Constructor constructor = componentClass.getConstructors()[0];
            Object instance = constructor.newInstance();

            for (Map.Entry<Field, Pair<String, Class>> entry : fields.entrySet()) {
                Field field = entry.getKey();
                Pair<String, Class> dependency = entry.getValue();
                Class depType = dependency.getValue();

                Object value = null; // TODO: get dependency's value
                if (!field.isAccessible()) field.setAccessible(true);
                field.set(instance, value);
            }

            return instance;
        } catch (IllegalAccessException | InvocationTargetException | InstantiationException e) {
            throw new IllegalStateException("Could not invoke constructor for class " + componentClass, e);
        }
    }
);
```
<div class="caption">return statement in DependencyCollectors::getDependencyUnitForComponent</div>

Previously this was just
```java
return new DependencyUnit(
  componentClass.getName(),
  componentClass,
  dependencies
);
```

but we’ve added the lambda function as the last argument, the `Supplier<Object>`. Let’s talk about what’s going on here. The goal is for this function to return an instance of the class that the `DependencyUnit` provides, with all of the `@Autowired` dependencies populated. So, we again use reflection to invoke the constructor for the component class and obtain an instance of that class. Note that its type is `Object` since we don’t really know anything about that class at runtime. Then, for each field that we had identified above as having an `@Autowired` annotation, we use reflection to set the field’s value on that instance to… something. What should that be though?

We have the field’s name and its type, which should be enough to find any dependencies that have already been declared in our dependency graph, but the problem is that we currently have no place to store the instances of our `@Component`s. This is what’s known as an **application context**, which you can essentially think of as a `Map<Class, Object>`, a data structure which maps a `Class` to an instance of that class. The value that we assign to each `@Autowired` field will come from entries that have already been inserted into the application context. We don’t have any class to represent the application context, and we won’t need one just yet, but let’s imagine that such a thing exists. We’d want a way, from within our `getDependencyUnitForComponent` method, to get an instance of a dependency based on its name and its type. Rather than having our method accept a parameter of `Function` though, let’s define an interface for it (we will make it a `@FunctionalInterface` since it will only have one method and be essentially the same as passing in a function as far as we’re concerned, but we have a much better declarative name for it instead of some gross, generic `Function` type).

```java
package co.kenrg.winter.application;

import org.apache.commons.lang3.tuple.Pair;

import java.util.List;

@FunctionalInterface
public interface DependencyInstanceGetter {
    List<Object> getDependencyInstances(List<Class> dependencies);
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/iface/DependencyInstanceGetter.java</div>

You may be wondering why this signature isn’t simply

```java
Object getDependencyInstance(Class dependency);
```

and that’s because eventually we will want to do this operation in bulk, and I’m anticipating that. So now if we make our `getDependencyUnitForComponent` method accept a `DependencyInstanceGetter` as its second parameter, we can replace that `//TODO` statement with:
```java
Object value = dependencyInstanceGetter
    .getDependencyInstances(Lists.newArrayList(depType))
    .get(0);
```

Okay so this makes sense, except now we need to revisit our `run` method in `Application`, because we’ll need to pass in a `DependencyInstanceGetter`. And in order to do that, I think we need to start working on an `ApplicationContext` class.
```java
package co.kenrg.hooke.application;

import java.util.List;
import java.util.Map;

import com.google.common.collect.Lists;
import com.google.common.collect.Maps;

public class ApplicationContext {
    private final Map<Class, Object> applicationContext = Maps.newHashMap();

    public List<Object> getBeansOfTypes(List<Class> classes) {
        List<Object> beans = Lists.newArrayList();
        for (Class clazz : classes) {
            if (applicationContext.containsKey(clazz)) {
                beans.add(applicationContext.get(clazz));
            } else {
                throw new IllegalStateException("No bean available of type " + clazz);
            }
        }
        return beans;
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/ApplicationContext.java</div>

It may be fairly simple now, but it _will_ get more complex soon. Now, let’s create a new instance variable in our `Application` class, and add the necessary second parameter to `getDependencyUnitForComponent`:
```java
package co.kenrg.hooke.application;

import static co.kenrg.hooke.application.DependencyCollectors.getDependencyUnitForComponent;

import java.io.IOException;
import java.lang.annotation.Annotation;
import java.util.Set;

import co.kenrg.hooke.annotations.Component;
import co.kenrg.hooke.application.graph.DependencyUnit;
import com.google.common.collect.ImmutableSet;
import com.google.common.collect.Sets;
import com.google.common.reflect.ClassPath;
import com.google.common.reflect.ClassPath.ClassInfo;

public class Application {
    private final Class appClass;
    private final Set<Class> componentClasses = Sets.newHashSet();
    private final ApplicationContext applicationContext = new ApplicationContext();

    public Application(Class appClass) {
        this.appClass = appClass;
    }

    public void run() throws IOException {
        String packageName = this.appClass.getPackage().getName();
        ClassLoader classLoader = this.appClass.getClassLoader();
        ImmutableSet<ClassInfo> allClasses = ClassPath.from(classLoader).getTopLevelClassesRecursive(packageName);
        for (ClassPath.ClassInfo classInfo : allClasses) {
            Class<?> clazz = classInfo.load();

            for (Annotation annotation : clazz.getAnnotations()) {
                if (annotation.annotationType().equals(Component.class)) {
                    componentClasses.add(clazz);
                }
            }
        }

        for (Class componentClass : componentClasses) {
            DependencyUnit dependencyUnit = getDependencyUnitForComponent(componentClass, applicationContext::getBeansOfTypes);
            System.out.println(dependencyUnit);
        }
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/Application.java</div>

The only difference here really is the function reference passed to `getDependencyUnitForComponent`. The `getBeansOfTypes` method will obviously never return anything, since we’re currently never inserting into the `ApplicationContext`'s underlying `Map`, but this code should still produce the same output when run as it did at the end of Step 2. This is because the `instanceProvider` function that we initialize our `DependencyUnit`s with is never called. Let’s tackle both of these issues at once as we work on resolving the dependency graph!

Let’s think about how we want to do this. A node in our graph cannot be thought of as fully instantiated until all of its dependencies have been satisfied; only then can it be inserted into the `ApplicationContext`. Luckily though, we have a graph as our data structure. In our graph, how can we determine then which classes have no dependencies? That’s right, look at the leaves of that graph. The leaf nodes of the graph are ones that have no dependencies, and can be instantiated even with an empty `ApplicationContext`. Once we have all of the leaves, we instantiate them and then we insert them into the `ApplicationContext`; then we can move inwards a layer and instantiate all of the nodes in our graph that only depend on the leaf nodes and insert those into the `ApplicationContext`. Repeat this until there are no more nodes left to instantiate. Here’s what this might look like, as another static method in the `DependencyGraph` class:
```java
public static void resolveDependencyGraph(Graph<DependencyUnit> graph) {
    Set<DependencyUnit> nodesToProcess = graph.nodes().stream()
        .filter(node -> graph.outDegree(node) == 0) // Start at leaves
        .collect(toSet());

    Set<DependencyUnit> processedNodes = Sets.newHashSet();
    while (!nodesToProcess.isEmpty()) {
        for (DependencyUnit node : nodesToProcess) {
            // TODO: Insert into application context
        }

        processedNodes.addAll(nodesToProcess);

        Set<DependencyUnit> nextLevel = Sets.newHashSet();
        for (DependencyUnit node : nodesToProcess) {
            Set<DependencyUnit> dependants = graph.predecessors(node).stream()
                .filter(dep -> !processedNodes.contains(dep))
                .collect(toSet());
            nextLevel.addAll(dependants);
        }
        nodesToProcess.clear();
        nodesToProcess.addAll(nextLevel);
    }
}
```
<div class="caption">resolveDependencyGraph method in src/main/java/co/kenrg/hooke/application/graph/DependencyGraph.java</div>

The code above should be pretty straightforward and, with the exception of that `//TODO` statement, everything should be in working order. We have a `DependencyUnit` in that loop, and we know we can call `node.getInstance()` on it to obtain the instance of the class that the `DependencyUnit` provides. But, what should we do with that instance once we have it? We need some way to insert that value into the `ApplicationContext`; let’s pass in a function to the `resolveDependencyGraph` method which allows us to do that from the outside. Much like `DependencyInstanceGetter`, I’ll define it as a `@FunctionalInterface`:
```java
package co.kenrg.hooke.application.iface;

@FunctionalInterface
public interface DependencyInserter {
    void insert(Class clazz, Object instance);
}
```
<div class="caption">src/main/java/co/kenrg/hooke/application/iface/DependencyInserter.java</div>

So now our `resolveDependencyGraph` method can accept a `DependencyInserter` as a second parameter, and the `//TODO` can be replaced with
```java
dependencyInserter.insert(
  node.providesClass,
  node.name,
  node.getInstance()
);
```

We’re almost there! Let’s modify our `ApplicationContext` class and add an incredibly easy insert method:

```java
public class ApplicationContext {
    private final Map<Class, Object> applicationContext = Maps.newHashMap();

    public List<Object> getBeansOfTypes(List<Class> classes) {
        List<Object> beans = Lists.newArrayList();
        for (Class clazz : classes) {
            if (applicationContext.containsKey(clazz)) {
                beans.add(applicationContext.get(clazz));
            } else {
                throw new IllegalStateException("No bean available of type " + clazz);
            }
        }
        return beans;
    }

    public void insert(Class clazz, Object instance) {
        applicationContext.put(clazz, instance);
    }
```
<div class="caption">src/main/java/co/kenrg/hooke/application/ApplicationContext.java</div>

and then we’re ready to make our final change to the `run` method in `Application`. We now gather all of the dependencies that we create from our `componentClasses`, build a dependency graph, and resolve the graph:
```java
public void run() throws IOException {
    String packageName = this.appClass.getPackage().getName();
    ClassLoader classLoader = this.appClass.getClassLoader();
    ImmutableSet<ClassInfo> allClasses = ClassPath.from(classLoader).getTopLevelClassesRecursive(packageName);
    for (ClassPath.ClassInfo classInfo : allClasses) {
        Class<?> clazz = classInfo.load();

        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().equals(Component.class)) {
                componentClasses.add(clazz);
            }
        }
    }

    List<DependencyUnit> dependencies = Lists.newArrayList();
    for (Class componentClass : componentClasses) {
        dependencies.add(getDependencyUnitForComponent(componentClass, applicationContext::getBeansOfTypes));
    }

    Graph<DependencyUnit> graph = buildDependencyGraph(dependencies);
    resolveDependencyGraph(graph, applicationContext::insert);
}
```
<div class="caption">run method in src/main/java/co/kenrg/hooke/application/Application.java</div>

If we run this now, we should see… nothing. Yeah, we currently don’t have any way to verify that this is working. Let’s fix that in what will be the last step for this article. But first, the code so far is available at the [`step-3-building-and-resolving-the-dependency-graph`](https://github.com/kengorab/hooke/tree/step-3-building-and-resolving-the-dependency-graph) tag on the project’s Github repo.

## Step 4: Reaching into the ApplicationContext

As Step 3 ended, we were left with no way of knowing that our code is actually working. Let’s add some temporary code to `HookeApplication` to facilitate this (I say temporary because there are better ways of doing what we’re about to do, and I will not keep this code around for long).
```java
public class HookeApplication {
    private static Application INSTANCE;

    public static void start(Class appClass) {
        try {
            HookeApplication.INSTANCE = new Application(appClass);
            INSTANCE.run();
        } catch (IOException e) {
            throw new IllegalStateException(e);
        }
    }

    public static ApplicationContext getApplicationContext() {
        return INSTANCE.applicationContext;
    }
}
```
<div class="caption">src/main/java/co/kenrg/hooke/HookeApplication.java</div>

The only new bit here is the `getApplicationContext` method, which returns the `applicationContext` on the instance of our `Application` (the `applicationContext` field on the `Application` class will need to be made public for this). I also added a `getBeanOfType` method to the `ApplicationContext` class, which does some fancy stuff with generics but is otherwise pretty simple:

```java
public <T> T getBeanOfType(Class<T> clazz) {
    return (T) getBeansOfTypes(Lists.newArrayList(clazz)).get(0);
}
```
<div class="caption">getBeanOfType method in src/main/java/co/kenrg/hooke/application/ApplicationContext.java</div>

Now, in our example application within our tests, let’s make a couple of changes starting with `Component1` and `Component2`, then followed by code in the main method which makes use of the two methods we’ve added above:
```java
// Component1.java
@Component
public class Component1 {
    @Autowired private Component2 component2;

    public void printMessage() {
        System.out.printf("Component1 says 'Hello'; Component2 says '%s'\n", component2.getMessage());
    }
}


// Component2.java
@Component
public class Component2 {
    public String getMessage() {
        return "Howdy";
    }
}


// ExampleHookeApplication.java
public class ExampleHookeApplication {
    public static void main(String[] args) {
        HookeApplication.start(ExampleHookeApplication.class);

        ApplicationContext ctx = HookeApplication.getApplicationContext();
        Component1 component1 = ctx.getBeanOfType(Component1.class);
        component1.printMessage();
    }
}
```

Running this code will print out “Component1 says ‘Hello’; Component2 says ‘Howdy’”, which lets us know that our code is all working correctly!

So let’s recap everything that’s happening here. We start with `HookeApplication.start(ExampleHookeApplication.class)`. This kickstarts the application, and Hooke will find all classes in the example project that are annotated with `@Component`. For each of those classes, it finds any fields annotated with `@Autowired`. Then, with all of those dependency relationships defined we build a graph and resolve it, instantiating classes as all of their dependencies become available and inserting them into the `ApplicationContext` to be used as dependencies of other classes. Once everything is resolved, we’re left with an instance of `Component2` and an instance of `Component1` in the `ApplicationContext`; the `component2` instance variable of `Component1` is assigned to the instance of `Component2`. The next line in our main method, which accesses the internals of our application to get a bean from the `ApplicationContext`. With our instance of `Component1` in hand, we call `component1.printMessage()` which calls a method on its `component2` instance variable, and causes the expected message to be output. If this didn’t work and something broke along the way, we’d likely have a `NullPointerException` trying to call `component2.getMessage()` within the `printMessage` method in `Component1`.

Not that a whole lot has changed from Step 3 to Step 4, but the code is available under the [`step-4-reaching-into-the-application-context`](https://github.com/kengorab/hooke/tree/step-4-reaching-into-the-application-context) tag at the project’s Github repo.

## To Be Continued…

Okay, this was a _lot_. But, we’re left with a really satisfying result, and a working rewrite of some of the basic core functionality of Spring. Hopefully you’ve learned a bit about how Spring works and what those previously-mysterious annotations do, and hopefully it feels like a little less like dark magic.

However, there is much _much_ more left to do. We still need to handle the case where there are multiple beans of the same type, we need to handle the `@Qualifier` annotation, constructor-based dependency injection (as opposed to field-based dependency injection), `@PostConstruct` methods, `@Configuration` classes, methods annotated with `@Bean`, etc. Trust me when I say that, although there is a lot left to do, there are not a lot of changes we need to make in order to make it all happen. In fact, we’ve taken care of most of the difficult part — building and resolving the dependency graph. Really, all the rest just falls into place around that.

So, thank you for taking the time to read this, and I hope you’ve learned something! Continue on to [Part 2](/posts/2018/03/lets-write-spring-2) to build out support for `@Configuration`, `@Bean`, and `@Qualifier` annotations.


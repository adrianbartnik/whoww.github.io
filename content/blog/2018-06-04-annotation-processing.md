---
title: "Java Annotation Processing"
date: "2018-06-04"
draft: false
tags: ["java", "annotation-processing"]
slug: "annotation-processing"
---

Have you ever wondered what the `@Override`-annotation on all your methods really means and how it is used under the hood?
Or do you have nice use-case that would benefit from automatic code generation?
Or maybe you simply want to learn more about the annotations and the java compilation process, then this post is for you.
We are going to talk about *Annotation Processing* in Java, what it is all about, how it integrates with the Java compiler and how you can write your own annotation processor.

**TLDR**: This post implements a custom annotation processor that generates convenience functions for missing features.
You can find the code on [Github](https://github.com/whoww/annotation-processor).


# Java Annotations

Annotations simply act as *markers* and provide additional information to the Java compiler **during compilation**, but have not effect on the application at runtime.
They stand in contrast to Reflection, which is all about retrieving, examining & manipulating properties of objects at runtime but comes with certain drawbacks.
Most importantly, reflection imposes impose a performance penalty when accessing the object properties.
Additionally, it may introduce security issues and expose the inner workings of a class.
You can read more about that in the [Java documentation](https://docs.oracle.com/javase/tutorial/reflect/index.html).

## Annotation Overview

Annotation have been introduced with Java 6, which was released in December 2006, and provide an easy mechanism for adding further information for your Java source code.
Java 6 came with 3 standard annotations, such as `@Override` in order to check if annotated method actually overrides a parent method or implements a method from an implemented interface.
`@Deprecated` results in a compiler warning if a method is used that has been marked as deprecated by the developer.
`@SuppressWarnings` may be used to suppress these or other compiler warnings.
Java 8 introduced some new annotations such as the [`FunctionalInterface`](https://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html), which is used to marked functions, which can be used in Lambda expressions.
Additionally, before Java 8, only declarations of classes, fields, methods, and other program elements such as parameters could be annotated.
With the release of Java 8, annotations can also be applied to the use of types, such as in:

```java
// Class instance creation expression
new @Interned MyObject();

// Type cast
myString = (@NonNull String) str;

// implements clause
class UnmodifiableList<T> implements @Readonly List<@Readonly T> {...}

// Thrown exception declaration
void monitorTemperature() throws @Critical TemperatureException {...}
```

## Compilation Process

As already mentioned, annotations are processed during the compilation and not during runtime, as shown in the figure below ([Source](http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html)).
All Java sources files are feed into the parser, which generates [Abstract Syntax Trees](https://en.wikipedia.org/wiki/Abstract_syntax_tree) and fills the compiler's symbol table.
Then, for each annotation, the corresponding Annotation Processor is called, which may generate new source files.
All newly generated source files are then feed back to the parser as they also need to be parsed and may include further annotations.
Finally, when no new source files have been generated, the syntax trees are translated into class files.
This last step includes resolving references to external libraries, whose classes also need to be compiled.
However, they will not be subject to annotation processing.

{{< figure src="/img/blog/annotation-processor/CompilationProcess.svg" title="Java Compilation Process with Annotation Processing" >}}

It is important to note that an annotation processor can **only generate new source files** but not modify existing files.
It is also important to know that `javac` runs the annotation processor inside its *own JVM* during compilation.
This means that you can use arbitrary Java libraries in order to generate your source files and also write tests.
We will see an example in the upcoming sections.

## Custom Annotation Processor

The general structure for writing an annotation processor is as follows:
First, have one dedicated project or module, which only contains the annotations themselves.
Both the annotation processor and the client depend on this project.
However, the client does not need a compile-time dependency on the annotation processor, but only need to run it before the real code.
By doing this, we prevent copying all the annotation processor code into the final jar-file.
You can see the final structure in the figure below.

{{< figure src="/img/blog/annotation-processor/Dependencies.svg" title="Annotation Processing Dependencies" >}}

### Annotations

In the following, we will create our own annotation processor.
It will allow us to document pending features on classes we want to implement in the future.
The features are described by a *severity* and a *description*.
The annotation processor will collect those annotations and create a new file, `MissingFeatureManager`, which provides convenience functionality, such as checking that no feature above a certain severity is still present.
We could also think about generating some documentation based on those pending features.
You can find the code on [Github](https://github.com/whoww/annotation-processor).
Anyway, it's simply for learning purposes, so let's start.

In our case, the annotation project contains two annotations: `@MissingFeature` and `@MissingFeatures`.
`@Target(ElementType.TYPE)` means that only interfaces and classes can be annotated and `@Retention(RetentionPolicy.CLASS)` that the annotations will not be available at runtime, e.g. for reflection.
One special case is the `@Repeatable(MissingFeatures.class)`, which is needed in order to be able to annotate the same annotation to the same class multiple times.
This feature has been introduce in Java 8 and requires that we also add the additional interface `@MissingFeatures`.
This interface groups multiple `MissingFeature` for a single class in an array and makes it available to the annotation processor.

```java
@Repeatable(MissingFeatures.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface MissingFeature {

    String featureDescription();

    SeverityLevel severityLevel() default SeverityLevel.MEDIUM;

    enum SeverityLevel {
        CRITICAL(4),
        HIGH(3),
        MEDIUM(2),
        LOW(1),
        NONE(0);
    }
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface MissingFeatures {

    MissingFeature[] value();
}
```

### Annotation Processor

Generally, all annotation processor must be registered before they can be run.
This happens by creating a file named `javax.annotation.processing.Processor` in the `META-INF/services` directory.
Each line in the file may specify an annotation processor that is automatically detected and run by `Javac`.
However, you we also simply use Google's [`AutoService`](https://github.com/google/auto/tree/master/service), which automatically generates this file for us (it is also an annotation processor).
We only have to annotate each annotation processor in our project with `@AutoService(Processor.class)`.
You can find the dependencies as an extract from the `build.gradle` file below.

```java
dependencies {
    compile project(':annotation')
    compile 'com.squareup:javapoet:1.9.0' // Easy creation of java classes
    compile 'com.google.auto.service:auto-service:1.0-rc1'
}
```

Each annotation processor either has to implement the `Processor` interface or inherit from `AbstractProcessor`.
The initialization of the annotation processor happens in the `init`-method, where we can get references to important classes.
The most important ones are:

* `Messager` for logging errors
* `Filer` for writing new classes
* `Elements` to work with `Element` classes (more information later)
* `Types` to work with `TypeMirror` (more information later)

One of the first steps when writing your own annotation processor should be overriding `getSupportedAnnotationTypes` which signal in which annotation you processor is interested in.
In our case, these are `@MissingFeature` and `@MissingFeatures`.

```java
@AutoService(Processor.class)
public class MissingFeatureProcessor extends AbstractProcessor {

    private Filer filer;
    private Messager messager;
    private MissingFeatureManagerGenerator missingFeatureManagerGenerator =
      new MissingFeatureManagerGenerator();

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        filer = processingEnv.getFiler();
        messager = processingEnv.getMessager();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> set = new HashSet<>();
        set.add(MissingFeature.class.getCanonicalName());
        set.add(MissingFeatures.class.getCanonicalName());
        return set;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}
```

The `Messager` object is used for logging progress or notifying the developer from exceptions.
Since the annotation processor is run in its own JVM, it needs a way to notify the compiler about compilation errors.
It is important to note that every `error` log done via the `Messager` object will stop the compilation process.

As we have seen previously, the Java compiler will scan through our source code and look for annotations.
We have seen that there are various valid locations where we can use annotations.
Our processor will then receive those elements in the `process`-method corresponding to the annotations he registered for.

[Hannes Dorfman](http://hannesdorfmann.com/annotation-processing/annotationprocessing101) has this nice overview of different types of elements.

```java
package com.example;	// PackageElement

public class Foo {		// TypeElement

	private int a;		// VariableElement
	private Foo other; 	// VariableElement

	public Foo () {} 	// ExecuteableElement

	public void setA ( 	// ExecuteableElement
	                 int newA	// TypeElement
	                 ) {}
}
```

Each element represents some part of the source code and provides information such as the class name.
However, for retrieving more information about the class itself, such as superclass or its implemented interface, you have to use a `TypeMirror`.
The `TypeMirror` of each element is accessible by calling `element.asType()`.
Since in our use-case we are not so much concerned with the different element types, I leave it like that and forward you to his post in case you want to learn more.
We retrieve all classes that annotated with one of our annotations from the `RoundEnviroment` via `getElementsAnnotatedWith(MissingFeature.class)`.

```java
@AutoService(Processor.class)
public class MissingFeatureProcessor extends AbstractProcessor {

    ...

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

        try {

            processMissingFeatureAnnotations(roundEnv);
            processMissingFeaturesAnnotations(roundEnv);

            missingFeatureManagerGenerator.generateCode(filer);

        } catch (ProcessingException exception) {
            // Error handling
        }

        return true;
    }
}
```

In this method, we check whether the element that has been annotated is actually a class.
Otherwise we throw an exception and stop processing further elements.
In case we have found a valid element, we create a convenience object `MissingFeatureAnnotatedClass` that holds all necessary information.
The example below shows the code for a class that has only one `MissingFeature` annotation.
However, since repeated annotations has only been added in Java 8, we need to handle this case separately.
It works basically the same, the only difference is that we can retrieve an array of annotations from the `Element`.
As last step, we pass the objects to our generator, who will eventually generate our `MissingAnnotationManager`.

```java
private void processMissingFeatureAnnotations(RoundEnvironment roundEnv)
  throws ProcessingException {
    for (Element annotatedElement :
        roundEnv.getElementsAnnotatedWith(MissingFeature.class)) {

        // Check if a class has been annotated with @MissingFeature
        checkForClassType(annotatedElement);

        // We can cast it, because we know that it of ElementKind.CLASS
        TypeElement typeElement = (TypeElement) annotatedElement;

        MissingFeatureAnnotatedClass missingFeatureAnnotatedClass =
          new MissingFeatureAnnotatedClass(typeElement);

        missingFeatureManagerGenerator.add(missingFeatureAnnotatedClass);
    }
}

private void checkForClassType(Element annotatedElement)
  throws ProcessingException {
    // Check if a class has been annotated with @MissingFeature
    if (annotatedElement.getKind() != ElementKind.CLASS) {
        throw new ProcessingException(annotatedElement,
            "Only classes can be annotated with @%s",
                MissingFeature.class.getSimpleName());
    }
}
```

The only thing that is missing is the source code for the `MissingFeatureManagerGenerator`.
Once all classes with `MissingFeature` annotations have been added, we are ready to generate its corresponding source file.
The class simply maintains a list of `MissingFeatureHolder`, which in turn simply store the fully qualified class name, the feature description and the severity.
Once we have processed all annotations, our annotation processor writes a simple java class, which makes this list accessible and adds convenience functionality.
We will see an example output in just a second.
For generating the Java source file, we use the [Javapoet](https://github.com/square/javapoet) library by Square.

```java
public class MissingFeatureManagerGenerator {

  private List<MissingFeatureHolder> missingFeatures = new ArrayList<>();

  void add(MissingFeatureAnnotatedClass annotatedClass) {
      missingFeatures.add(annotatedClass.getMissingFeatureHolder());
  }

  void generateCode(Filer filer) throws IOException {

      ClassName missingFeature = ClassName.get(MissingFeatureHolder.class);
      ClassName list = ClassName.get("java.util", "List");
      ClassName arrayList = ClassName.get("java.util", "ArrayList");
      TypeName listOfFeatures =
        ParameterizedTypeName.get(list, missingFeature);

      // Java Poet Code
      // ...

      // Write file
      JavaFile.builder(PACKAGE, typeSpec)
              .addStaticImport(MissingFeature.SeverityLevel.NONE)
              .addStaticImport(MissingFeature.SeverityLevel.LOW)
              .addStaticImport(MissingFeature.SeverityLevel.MEDIUM)
              .addStaticImport(MissingFeature.SeverityLevel.HIGH)
              .addStaticImport(MissingFeature.SeverityLevel.CRITICAL)
              .build()
              .writeTo(filer);
  }
}
```

As mentioned previously, all newly generated files, in this case our `MissingFeatureManager`, will also be subject to annotation processing.
However, in the next pass, only all newly created files will be considered.
Although, we do not except to see any newly class files with a `MissingFeature` annotation, we have to guard against overwriting the file again since we currently use a static file name.
This can be done by using a simple `boolean` toggle.

### Client

Lastly, our client needs to depend on both projects and to see some output annotate some classes with our annotations.
In order to enable annotation processing in Java with Gradle, we use the [`gradle-apt-plugin`](https://github.com/tbroyer/gradle-apt-plugin).
The example below is for using the IntelliJ IDE, to use it in Eclipse, simple replace `id "net.ltgt.apt-idea" version "0.15"` by `id "net.ltgt.apt-eclipse" version "0.15"`.

```java
plugins {
    id 'java'
    id "net.ltgt.apt-idea" version "0.15"
}

dependencies {
    compile project(":annotation")
    annotationProcessor project(':processor')
}
```

This is an example of our annotation in action.
We can annotation each class with multiple `@MissingFeature` annotation and specify a severity as well as a description.


```java
@MissingFeature(
        featureDescription = "Missing Feature Two",
        severityLevel = MissingFeature.SeverityLevel.HIGH)
class DummyClassTwo {
}

@MissingFeature(featureDescription = "Implement Logic",
        severityLevel = MissingFeature.SeverityLevel.HIGH)
@MissingFeature(featureDescription = "Write Blogpost",
        severityLevel = MissingFeature.SeverityLevel.CRITICAL)
@MissingFeature(featureDescription = "Add Authorization")
class DummyClassOne {
}
```

The corresponding generated java files looks as follows:
It contains a convenience method for checking if all features above a certain severity have been resolved.
Otherwise, it will throw a runtime exception.
But it should be clear by now that we are also free to generate other kinds of documents, send email notifications or generally do whatever we what since the annotation processor is simply its own Java application.

```java
public final class MissingFeatureManager {
  private static final List<MissingFeatureHolder> missingFeatures =
    new ArrayList<>();

  static {
    missingFeatures.add(
      new MissingFeatureHolder("DummyClassTwo", "Missing Feature Two", HIGH));
    missingFeatures.add(
      new MissingFeatureHolder("DummyClassOne", "Implement Logic", HIGH));
    missingFeatures.add(
      new MissingFeatureHolder("DummyClassOne", "Write Blogpost", CRITICAL));
    missingFeatures.add(
      new MissingFeatureHolder("DummyClassOne", "Add Authorization", MEDIUM));
  }

  public List<MissingFeatureHolder> getAllMissingFeatures() {
    return missingFeatures;
  }

  public void CheckNoBelow(MissingFeature.SeverityLevel severityLevel) {
    for (MissingFeatureHolder tmp : missingFeatures) {
      if (tmp.getSeverityLevel().isMoreSevereThan(severityLevel)) {
        throw new IllegalStateException(
          "There are more severe open features!");
      }
    }
  }
}
```

# Java 9

Java 9 further extends the annotation mechanism by introducing the [`@Generated`-Annotation](https://docs.oracle.com/javase/8/docs/api/javax/annotation/Generated.html), which marks generated source files and allows to add further information, such as the annotation processor name, the author and date.
Java 9 also adds further functionality to the [`RoundEnvironment`](https://docs.oracle.com/javase/9/docs/api/javax/annotation/processing/RoundEnvironment.html) for easier handling of multiple annotation: `getElementsAnnotatedWithAny​(Set<Class<? extends Annotation>> annotations)` and `getElementsAnnotatedWithAny​(TypeElement... annotations)`.

# Outlook and Further Information

Although this guide was rather lengthy, there are still a lot of things that have not been covered.
Most importantly, processing annotation at runtime.
The Spring framework makes heavy use of adding functionality to your Java application at runtime using aspect oriented programming.
But that is different topic for a separate blog post.

For further information, please see the following posts:

* Hannes Dorfman - http://hannesdorfmann.com/annotation-processing/annotationprocessing101
* Jorge Hidalgo - https://www.slideshare.net/deors/javaone-2017-con3282-code-generation-with-annotation-processors-state-of-the-art-in-java-9
* Mert Şimşek - https://medium.com/@iammert/annotation-processing-dont-repeat-yourself-generate-your-code-8425e60c6657
* Java compilation process - http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html


# Summary

This post described the magic behind the Java annotation processor that drives many popular libraries, such as Dagger, Data Binding or Room.
We started by understanding what annotations in Java actually are and how they integrate with the compilation process.
We continued by defining a simple example use-case separated into `annotation`, `annotation-processor` and `client`.
We then implemented all three parts and saw the final result.
Lastly, we saw a brief summary of Java 9 features and an outlook.

I hope you have learned something.
Please let me know in case you have any further questions.

You can find the code on [Github](https://github.com/whoww/annotation-processor).

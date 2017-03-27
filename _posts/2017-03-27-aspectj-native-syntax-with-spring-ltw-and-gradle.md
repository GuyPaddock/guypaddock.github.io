---
layout: article
title: "Using Native AspectJ (.aj) Syntax in Spring with Load-Time Weaving and a Gradle-based Build Process"
categories: posts
modified: 2017-03-27T13:05:00-04:00
tags: [aspectj, java, gradle, spring, aop, load-time-weaving]
comments: true
ads: false
---

## Introduction
There are many resources out there about how to use AspectJ in Spring, but most fit into the following categories:
- They assume you are using Maven instead of Gradle; or
- They assume you want to use AspectJ's annotation-based syntax (_@AspectJ style_), and don't acknowledge native AspectJ (.aj) syntax at all; or
- They assume you want to use compile-time weaving instead of load-time weaving; or
- They assume you want to use Spring's built-in AspectJ support instead of the native AspectJ weaver; or
- They haven't been updated for the latest versions of Spring and Spring Boot

**This post takes a different approach, to satisfy the following goals:**
- We want to use **Gradle**, because its syntax is more succinct and Gradle projects ideally are easier to maintain.
- We want to have support for both the annotation-based and **native AspectJ syntax**, because the full-blown syntax reads more cleanly and is less awkward for more advanced aspects.
- We need to be able to use the native **AspectJ weaver** to support the full AspectJ syntax.
- We want to be able to use **load-time weaving**, so that we can decide whether we want certain aspects on or off (trace logging, performance tracing, etc) without any hits to application performance from code that's turned off &ndash; at run-time, without having to recompile our entire project.
- We need to be able to work with the **current versions of Spring and Spring Boot** (4.3.7 and 1.5.2, respectively, as of the time of this writing).

## Differences Between AspectJ and Spring's Flavor of AspectJ
If you're familiar with the [Spring framework](https://spring.io/) and Spring AOP, you know that it borrowed AspectJ's annotation-based syntax in Spring 2.0. What you may not know is that AspectJ also provides a first-class Java-like language for declaring aspects that is more powerful and provides some options that Spring's implementation of AOP does not.

The reason that AspectJ's capabilities and Spring AOP's capabilities seem disparate, even though one is seemingly based on the other, is that Spring is actually using a different approach under the hood. AspectJ uses _bytecode weaving_ which actually changes the code that gets executed to accommodate the call flow changes of the aspects, while Spring AOP uses _AOP proxies_, which modifies the calls from one component (_bean_) to another in response to pointcuts in the aspects. For basic aspects, the difference is imperceptible.

An in-depth look at how Spring AOP compares to AspectJ is outside the scope of this post. Luckily, this post doesn't have to! I recommend these resources if this is something you'd like to read more about:

 - [Justin Wilson](https://www.credera.com/blog/technology-insights/open-source-technology-insights/aspect-oriented-programming-in-spring-boot-part-3-setting-up-aspectj-load-time-weaving/) has an excellent three-part blog post on how to use Spring AOP, how it works under the hood, and how to switch on Load-time Weaving. A large portion of my blog post is based on information from Justin's post, but not everything Justin provided worked out of the box for our use case.
 - [Spring's Documentation](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html) provides extensive documentation on AOP in general, with copious examples of how to work with it in Spring.
 - [_AspectJ in Action: Second Edition_](https://www.amazon.com/gp/product/1933988053/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1933988053&linkCode=as2&tag=guypaddock-20&linkId=833eb5fb6499989a0daae8ac6989b380) remains the definitive guide for AspectJ. The main difference between the first and second edition is that the second edition specifically discusses how to use AspectJ with Spring.
 - [Emmanouil Gkatziouras](https://egkatzioura.wordpress.com/2016/05/29/aspect-oriented-programming-with-spring-boot/) provides a quick tutorial on how to get a simple Spring Boot app with Aspects using Gradle. A large portion of the sample for this blog post was originally based on Emmanouil's sample project.

## Step #1: Create a Gradle-based Spring Boot Project (Something We Can Augment with Aspects)
Although not strictly required to follow along with this tutorial, you'll want to have a Spring Boot project that is small enough to make tweaks and quickly see the result, but that has some functionality to play with.

You can feel free to use the "New Project" wizard / "Spring Initializr" that comes with your IDE (Eclipse for Java EE, IntelliJ, etc) to create said project, as long as you make sure to select "Gradle Project" as the build system / project type.

Or, check out [the sample project](https://github.com/GuyPaddock/aspectj-ltw-gradle-sample) I've created to accompany this blog post. If you want to start as minimal as possible, you can roll your copy of the repo back to commit [33a1036](https://github.com/GuyPaddock/aspectj-ltw-gradle-sample/commit/33a1036ef44aeaaa7850653718d3489c31577c03).

You can also start from [Emmanouil Gkatziouras's sample project](https://egkatzioura.wordpress.com/2016/05/29/aspect-oriented-programming-with-spring-boot/), upon which my sample project was originally based.

## Step #2: Create Some Aspects (using Annotation-based Syntax)
If you're starting from a basic Spring Boot app, you'll want to create one or more aspects to alter the behavior of the project. For now, we're going to use the Annotation-based AspectJ syntax that Spring uses by default, just so we can get things working with Spring's default configuration. Then, I'll show you what these aspects look like in the native syntax after we get that working.

If you checked-out my sample project, you'll already have one aspect &ndash; `SampleServiceAspect` &ndash; courtesy of Emmanouil Gkatziouras. To make things interesting, here is a second aspect you can add to your project, taken from [Spring's official docs](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html#aop-aj-ltw-first-example): 
```Java
package com.rosieapp.aop.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.util.StopWatch;

@Aspect
@Component
public class ProfilingAspect {
  private static final Logger LOGGER = LoggerFactory.getLogger(ProfilingAspect.class);

  @Around("methodsToBeProfiled()")
  public Object profile(ProceedingJoinPoint pjp) throws Throwable {
      StopWatch sw = new StopWatch(getClass().getSimpleName());
      try {
          sw.start(pjp.getSignature().getName());
          return pjp.proceed();
      } finally {
          sw.stop();
          LOGGER.info(sw.prettyPrint());
      }
  }

  @Pointcut("execution(* com.rosieapp.aop.service.SampleService.createSample(java.lang.String))")
  public void methodsToBeProfiled(){}
}
```

What this aspect does is _profile_ the methods targeted by the `methodsToBeProfiled` pointcut, so that you can measure how long each method takes to execute. In this case, we're targeting a method called `createSample` in the `SampleService` Spring service in the sample project. 

If you're using your own sample project instead of starting from mine, you'll need to adjust classes and packages referenced in this aspect (e.g. `com.rosieapp.aop`, `SampleService`, etc) so they match the class and package names you're using in your sample project. Otherwise, this aspect may not build, or may build but not match anything in your project.

_If you get stuck at step #2, feel free to reference my sample project at commit [4c5887b](https://github.com/GuyPaddock/aspectj-ltw-gradle-sample/commit/4c5887b55f5f27776868f0f88dbb2b3a33242f44) ._

## Step #3: Enable AspectJ-based Load-time Weaving
As previously mentioned in this post, the default implementation of AOP you get with Spring is proxy-based and does not rely on the _AspectJ weaver_ at run-time. To use that default implementation, all you'd need to do is include the `org.springframework.boot:spring-boot-starter-aop` package as a dependency in your Gradle file, and then ensure that each of your aspects is a Java class in your project, denoted with both the `@Aspect` and `@Component` annotations (as shown in the sample aspect in step #2 above). Spring should do the rest.

But, if we're on the road to full-blown AspectJ support, this isn't enough! We need to do the following:
1. Reconfigure Gradle to pull in the dependencies for AspectJ.
2. Ensure Spring knows we want AspectJ-based load-time weaving turned on.
3. Ensure the AspectJ Weaver &ndash; and the dependencies needed for it to integrate with Spring &ndash; are loaded as agents at run-time.
4. Ensure our aspects are no longer auto-loaded as Spring Components (since we don't want to use Spring AOP for them).
5. Ensure AspectJ & Spring can find the aspects and apply them to the desired parts of our application, now that we're no longer having Spring load and weave them for us.

This blog post will cover the mechanics of how to do this, but will gloss over some subtle, yet important, details. For the full story, I will once again refer you to [Justin Wilson's article](https://www.credera.com/blog/technology-insights/open-source-technology-insights/aspect-oriented-programming-in-spring-boot-part-3-setting-up-aspectj-load-time-weaving/).

_If you get stuck at step #3, feel free to reference my sample project at commit [b204825](https://github.com/GuyPaddock/aspectj-ltw-gradle-sample/commit/b204825e2fa65cfff388c809cced8457865ffb9c)._

### Step #3.1 Reconfigure Gradle to pull in the dependencies for AspectJ
You'll want to make sure that you're pulling-in Spring's AOP starter package in your `build.gradle` file, as follows:

```Groovy
buildscript {
    ext {
        springBootVersion = '1.5.2.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}"
    }
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    // This is the important line
    compile 'org.springframework.boot:spring-boot-starter-aop'
}
```
**NOTE:** You'll want to make sure you're running at least Spring 1.5.0 to avoid [a known issue](https://github.com/spring-projects/spring-boot/issues/7587) in which Spring Boot attempts to load AspectJ aspects too early.

### Step #3.2 Ensure Spring Knows We Want AspectJ-based Load-time Weaving (LTW)
This is an easy part. We need to define a `Configuration` that Spring will automatically load for us, in which we politely demand that AspectJ LTW should be switched on, instead of turning it on only if an `aop.xml` file is found. _(Unfortunately, as described in more detail under step #3.5 below, this doesn't buy as much as we expect.)_

Here's a sample configuration class (the annotations above the class are the only important parts here):
```Java
package com.rosieapp.aop.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableLoadTimeWeaving;
import org.springframework.context.annotation.EnableLoadTimeWeaving.AspectJWeaving;

@Configuration
@EnableLoadTimeWeaving(aspectjWeaving = AspectJWeaving.ENABLED)
public class AspectJConfig {
}
```

### Step #3.3 Ensure AspectJ Is Loaded as an Agent at Run-time
This is where we encounter our first challenge. You might think that all we need to do is ensure that AspectJ is in the classpath for our application, for it to be loaded at run-time. But, this isn't the case &ndash; that's actually too late in the process. The Weaver needs to load as a _JVM Agent_ to give it early, lower-level access to classes before they're loaded.

If it were as easy as changing the classpath, simply adding the Spring AOP starter package as a dependency would be enough. We need some way to define the agents as JVM args. Thankfully, standing on the shoulders of giants (i.e. Justin Wilson), this is actually quite manageable if we leverage Gradle's configurations feature.

Define a new configuration in your `build.gradle` file, as follows:
```Groovy
configurations {
    // Configuration that tracks the agents added to the JVM at run-time
    runtimeAgent
}
```

This allows us to attach dependencies to a specific configuration, which we can use to notify the JVM at start-up time which JARs contain our agents. The goal is for us to pass the `-javaagent` argument to the JVM once for each agent JAR. _This will make sense in a moment._

Now, let's actually attach the agent dependencies further down in your `build.gradle` file:
```Groovy
dependencies {
    // ... Your other dependencies would be here ... //
    
    // These are the two important lines
    runtimeAgent "org.springframework:spring-instrument"
    runtimeAgent "org.aspectj:aspectjweaver"
}
```

As noted, the two new lines go inside your existing `dependencies` section, at the bottom. So, your full dependencies section might look like this now (your other dependencies will vary based on your project):
```Groovy
dependencies {
    compile 'org.springframework.boot:spring-boot-starter-aop'
    
    runtimeAgent "org.springframework:spring-instrument"
    runtimeAgent "org.aspectj:aspectjweaver"
}
```

What we're doing here is ensuring we have both the `spring-instrument` and `aspectjweaver` packages, and that they're associated with the `runtimeAgent` configuration. Before we added these two lines, Gradle was already downloading these packages as part of sorting out dependendencies for the `org.springframework.boot:spring-boot-starter-aop` package, but we didn't have a way to refer to where the JARs for these components lived on the system. (If you've referenced Justin Wilson's post, you may notice that this isn't the approach he took. In his case, he just bundled the JARs directly with the project since he was not sure what the best approach to referencing JARs managed by Gradle would be.)

As for why you need these two packages, the AspectJ Weaver package is self-explanatory, but why do we need the Spring Instrument package? As it turns out, AspectJ doesn't natively know how to interact with Spring's way of loading classes at run-time; the Spring Instrument package provides the bridge code needed for the two to co-exist. Since the AspectJ weaver has to load as a JVM agent, so too must we load the agent from the Spring Instrument package, so both are available together. Justin Wilson's article explains what happens when you don't load both.

Okay, we're almost in the home stretch for this step. Now, we need to leverage the configuration to actually ensure that the JARs get loaded by the JVM at start-up.

Add the following sections to the end of your  `build.gradle` file:
```Groovy
test.doFirst {
    // Ensure that all of the agents we need to load at run-time happen for tests
    configurations.runtimeAgent.each {
        File jarFile ->
            jvmArgs "-javaagent:${jarFile.absolutePath}"
    }
}

bootRun.doFirst {
    // Ensure that all of the agents we need to load at run-time happen when running the app through
    // Gradle
    configurations.runtimeAgent.each {
        File jarFile ->
            jvmArgs "-javaagent:${jarFile.absolutePath}"
    }
}
```

This is the secret sauce we needed! What we're doing here is exactly what we said earlier &ndash; grabbing the path to each of the JARs Gradle was nice enough to pull in for us as part of the `runtimeAgent` configuration, and then passing each one as a `-javaagent` argument. We're doing it twice because there are two different Gradle _tasks_ we want to make sure we do this for (`test` for when we run tests, and `bootRun` when we run our application using Gradle).

_There may be a way to DRY up my approach so we only have the loop over the configurations once, but I have not yet come across it. Pull requests and feedback are both welcome. :)_

### Step #3.4 Ensure Aspects Are No longer Auto-loaded by Spring
This is one of the easiest steps. All we need to do is remove the `@Component` annotation from each of our aspects (be sure to leave this annotation where it appears in other classes in your project; we're only interested in making this change on the _aspects_).

So, for example, this:
```Java
@Aspect
@Component
public class SampleServiceAspect {
  // ... omitted ...
}
```
Should now become:
```Java
@Aspect
public class SampleServiceAspect {
  // ... omitted ...
}
```

Boom. On to the next part!

### Step #3.5 Ensure AspectJ Finds and Applies the Aspects
This part is straightforward, but surprisingly easy to mess up because it deals with creating a file with a very specific name in a very specific place in your project. To make it worse, and contrary to what Justin Wilson's post says, Spring doesn't really help you identify when the file is missing or in the wrong place.

To complete this part, "all you should need to do&trade;" is create a file called `aop.xml` in `src/main/resources/META-INF/aop.xml`.
- If you don't store your resources under `src/main/resources`, you need to make sure you put `aop.xml` under `META-INF` in whichever folder you do use for resources.
- If you didn't create a `resources` folder, and are using Gradle's defaults, then `src/main/resources/META-INF/aop.xml` is typically still the default location for resources in your project.

**Whatever you do, make sure the file is named `aop.xml` under the `META-INF` folder &ndash; it's very easy to accidentally put it one folder up, or somehow have a typo when naming the file.**

The contents of `aop.xml` need to accomplish two things:
1. Specify what packages are safe to weave-in aspects (protecting, for example, packages provided by Spring or Java itself from unintended side effects in poorly-written aspects).
2. Indicate which aspects we want to load.

Once again, refer to Justin Wilson's article, or any of the resources I posted at the top of this blog post, for more detail about the purpose of `aop.xml` and other options of interest.

For now, here's a sample `aop.xml` (adjust classes and packages as necessary for your project):
```XML
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "http://www.eclipse.org/aspectj/dtd/aspectj.dtd">
<aspectj>
  <weaver options="-verbose -showWeaveInfo">
    <!-- We only want to weave classes that exist in our application-specific packages.

         IMPORTANT: The packages defined here MUST include the package that the aspects are
         defined in. The load-time aspect weaver must be able to weave-in additional methods each
         Spring requires each aspect to have at run-time. If you exclude the aspects, you'll get
         an error about aspectOf() not being defined.
      -->
    <include within="com.rosieapp.aop..*" />
  </weaver>
  <aspects>
    <!-- These are the two aspects we want to switch on for now. -->
    <aspect name="com.rosieapp.aop.aspect.ProfilingAspect"/>
    <aspect name="com.rosieapp.aop.aspect.SampleServiceAspect"/>
  </aspects>
</aspectj>
```
Per the note in the `weaver` section, a subtle point here is that you want to make sure that the weaver is given a list of packages that include the aspects. This is because Spring actually adds some extra methods to the aspects so that they can be wired into the Spring system. Without this, you'll get a [`NoSuchMethodError` when your appliaction loads](http://forum.spring.io/forum/spring-projects/aop/72229-nosuchmethoderror-aspect-aspectof).

## Step #4: Test Your Project
_Whew... take a breath._ I know -- that was a lot of configuration to change.

Now for something I think you'll really like. _Eh, okay. At least something a bit more fun._

Although we're not done yet accomplishing all of our goals, now is a good time to do a sanity check of our project and make sure that everything builds and runs. The best way to do this is to start our project through Gradle's `bootRun` task.

_If you're following along with my sample project, you'll want to checkout the project at commit [b204825](https://github.com/GuyPaddock/aspectj-ltw-gradle-sample/commit/b204825e2fa65cfff388c809cced8457865ffb9c)._

Assuming your project is structured like mine, you should be able to run the task from the command-line as follows:
```
gradle buildRun
```

If you're using IntelliJ and have imported your Gradle project with auto-import turned on, it's even easier &ndash; just open the "Gradle" panel, then double-click on "Tasks -> application -> bootRun" under your project in the list.

This should launch your Spring project, resulting in a log that looks something like this:
```
12:06:40 AM: Executing external task 'bootRun'...
:compileJava UP-TO-DATE
:processResources
:classes
:findMainClass
[AppClassLoader@58644d46] info AspectJ Weaver Version 1.8.10 built on Monday Dec 12, 2016 at 19:07:48 GMT
[AppClassLoader@58644d46] info register classloader sun.misc.Launcher$AppClassLoader@58644d46
[AppClassLoader@58644d46] info using configuration aspectj_sample/build/resources/main/META-INF/aop.xml
[AppClassLoader@58644d46] info register aspect com.rosieapp.aop.aspect.ProfilingAspect
[AppClassLoader@58644d46] info register aspect com.rosieapp.aop.aspect.SampleServiceAspect
[AppClassLoader@58644d46] warning javax.* types are not being woven because the weaver option '-Xset:weaveJavaxPackages=true' has not been specified
:bootRun

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.2.RELEASE)

2017-03-27 00:06:49.426  INFO 16696 --- [           main] c.rosieapp.aop.AspectjSampleApplication  : Starting AspectjSampleApplication on RL-WIN7-04254D with PID 16696 (aspectj_sample\build\classes\main started by guy.paddock in aspectj_sample)
2017-03-27 00:06:49.441  INFO 16696 --- [           main] c.rosieapp.aop.AspectjSampleApplication  : No active profile set, falling back to default profiles: default
2017-03-27 00:06:49.695  INFO 16696 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@74fe5c40: startup date [Mon Mar 27 00:06:49 EDT 2017]; root of context hierarchy
[AppClassLoader@58644d46] weaveinfo Join point 'method-execution(com.rosieapp.aop.model.Sample com.rosieapp.aop.service.SampleService.createSample(java.lang.String))' in Type 'com.rosieapp.aop.service.SampleService' (SampleService.java:16) advised by around advice from 'com.rosieapp.aop.aspect.ProfilingAspect' (ProfilingAspect.aj:13)
[AppClassLoader@58644d46] weaveinfo Join point 'method-execution(com.rosieapp.aop.model.Sample com.rosieapp.aop.service.SampleService.createSample(java.lang.String))' in Type 'com.rosieapp.aop.service.SampleService' (SampleService.java:16) advised by before advice from 'com.rosieapp.aop.aspect.SampleServiceAspect' (SampleServiceAspect.aj:9)
2017-03-27 00:06:56.620  INFO 16696 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8080 (http)
2017-03-27 00:06:56.675  INFO 16696 --- [           main] o.apache.catalina.core.StandardService   : Starting service Tomcat
2017-03-27 00:06:56.680  INFO 16696 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.11
2017-03-27 00:06:57.133  INFO 16696 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2017-03-27 00:06:57.134  INFO 16696 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 7449 ms
2017-03-27 00:06:57.559  INFO 16696 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
2017-03-27 00:06:57.570  INFO 16696 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
2017-03-27 00:06:57.572  INFO 16696 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2017-03-27 00:06:57.573  INFO 16696 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
2017-03-27 00:06:57.573  INFO 16696 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
2017-03-27 00:06:57.781  INFO 16696 --- [           main] o.s.c.w.DefaultContextLoadTimeWeaver     : Found Spring's JVM agent for instrumentation
2017-03-27 00:06:57.800  INFO 16696 --- [           main] o.s.c.w.DefaultContextLoadTimeWeaver     : Found Spring's JVM agent for instrumentation
2017-03-27 00:06:58.738  INFO 16696 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@74fe5c40: startup date [Mon Mar 27 00:06:49 EDT 2017]; root of context hierarchy
2017-03-27 00:06:59.074  INFO 16696 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/sample]}" onto public com.rosieapp.aop.model.Sample com.rosieapp.aop.controller.SampleController.sample(java.lang.String)
2017-03-27 00:06:59.091  INFO 16696 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
2017-03-27 00:06:59.093  INFO 16696 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
2017-03-27 00:06:59.271  INFO 16696 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-03-27 00:06:59.272  INFO 16696 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-03-27 00:06:59.517  INFO 16696 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-03-27 00:07:00.162  INFO 16696 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2017-03-27 00:07:00.507  INFO 16696 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080 (http)
2017-03-27 00:07:00.533  INFO 16696 --- [           main] c.rosieapp.aop.AspectjSampleApplication  : Started AspectjSampleApplication in 12.438 seconds (JVM running for 15.111)
2017-03-27 00:07:01.012  INFO 16696 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring FrameworkServlet 'dispatcherServlet'
2017-03-27 00:07:01.012  INFO 16696 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization started
2017-03-27 00:07:01.048  INFO 16696 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : FrameworkServlet 'dispatcherServlet': initialization completed in 36 ms
```

The lines you are looking for most are the `AppClassLoader` lines that say `info register aspect` and `weaveinfo Join point`. If you see both of these types of lines, with no fatal exceptions, great! This means your aspects are loading.

On the other hand:
- If you don't see any output at all from the weaver, this usually means your `aop.xml` file was not found / is not loading. Review steps 3.2 and 3.5 again, closely.
- If you see output from the weaver, but nothing about the aspects, then make sure you provided the correct, fully-qualified package and class of each aspect in the `aop.xml` file. You also need to make sure that your aspect classes are _in_ a namespace &ndash; the default package won't do!
- If you see output from the weaver about the aspects, but not the weave info, then make sure the packages listed in `<include>` tags in the `<weaver>` section are correct and include all of the classes you expect your aspects to affect. Inside the `within` attribute, you can use `*` as a wildcard to stand-in for a single package, and `..*` as a wildcard for all sub-packages.
- If you see a `NoSuchMethodError` about `aspectOf()`, review step 3.5 again.
- If you see warnings about `Xlint:nonReweavableTypeEncountered`, you can safely disregard them.

If all looks good, try actually requesting something from your application to trigger your aspects. With my sample application, this means opening up your browser to a URL like `http://localhost:8080/sample?sampleName=abc`. You should see the desired output in the logs:
```
2017-03-27 00:07:01.156  INFO 16696 --- [nio-8080-exec-1] c.r.aop.aspect.SampleServiceAspect       : A request was issued for a sample name: abc
2017-03-27 00:07:01.171  INFO 16696 --- [nio-8080-exec-1] com.rosieapp.aop.service.SampleService   : Sample is being fetched.
2017-03-27 00:07:01.181  INFO 16696 --- [nio-8080-exec-1] com.rosieapp.aop.aspect.ProfilingAspect  : StopWatch 'ProfilingAspect': running time (millis) = 3
-----------------------------------------
ms     %     Task name
-----------------------------------------
00003  100%  createSample
```

## Step #5: Switch over to Native AspectJ Syntax
This is the last step, and, similar to step #3, there are a few parts to it. This is also the only step that is not covered by any of the other blog articles or resources I've linked to earlier in this post, so you'll want to pay close attention.

To accomplish the switch over to using the full AspectJ syntax, we'll need to do the following:
1. Create a new folder structure for us to put our native `.aj` aspects.
2. Define a source set in Gradle for our new folder structure.
3. Port our aspects from the @AspectJ annotation-based syntax to the native syntax.
4. Configure Gradle to execute the AspectJ compiler on our new source set. 

Let's get to it!

## Step #5.1: Create a New Folder Structure for Aspects
Previously, since aspects in our project were just Plain Old Java Objects (POJOs) that just happened to have AspectJ annotations, they could live along-side our other classes in the same project. But, now that we're starting to write aspects in the native AspectJ `.aj` format, we'll want to move them to their own portion of our project so that we can compile them separately. If, instead, we were to simply keep all the files together and pass our project into the AspectJ compiler to be processed, we'd actually end up with Compile-Time Weaving (CTW) of our aspects, which isn't what we want.

To accomplish this step, all you need to do is change your project layout from this:
```
+-- src
|   +-- main
|   |   +-- java
|   |   +-- resources
|
|-- build.gradle
```

To this:
```
+-- src
|   +-- aspects
|   |   +-- aspectj
|
|   +-- main
|   |   +-- java
|   |   +-- resources
|
|-- build.gradle
```

Step #5.2: Define a Source Set in Gradle for Our New Folder Structure
Simply rearranging files in our project isn't enough for Gradle to work with them. We need to define a source set -- which gives a name to a set of files in a particular location that match a pattern.

You can add the following snippet to `build.gradle` to define the appropriate source set:
```Groovy
sourceSets {
    aspects {
        java {
            srcDir  'src/aspects/aspectj'
            include '**/*.aj'

            // This ensures that all of the classes available to the main application are accessible
            // to aspects as well.
            compileClasspath = sourceSets.main.compileClasspath
            runtimeClasspath = sourceSets.main.runtimeClasspath
        }
    }
}
```

The first two lines tell Gradle about our new folder structure, while the `include` line gaurantees that Gradle looks for more than just files ending in `.java`. If you're wondering why this is all under a block called `java`, it's because source sets are a concept defined by the Java plug-in for Gradle, and it only supports two groups of files: Java files, which get compiled; and Resource files, which get copied verbatim to the output. We want our AspectJ files compiled, not copied.

The last two lines workaround a limitation in this approach. As you may know, source sets have a corresponding compile-time and run-time classpath. Even though we are moving our aspects to a new spot in our project, we need to make sure that they can access all of the same classes we had access to when they lived in their previous location. These two lines do just that.

_There may be a better way to accomplish all of this than my approach here. Again, pull requests and feedback are both welcome. :)_

## Step #5.3: Port Annotation-based Aspects to Native AspectJ Syntax
If we want to switch to the native syntax, we might as well have some files written in that syntax that we can test out. Although the syntax between POJOs with @AspectJ annotations and native AspectJ is different, most of the code is portable between the two.

For example, here is what the profiling aspect from step #2 looks like in native syntax:
```AspectJ
package com.rosieapp.aop.aspect;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.StopWatch;

public aspect ProfilingAspect {
  private static final Logger LOGGER = LoggerFactory.getLogger(ProfilingAspect.class);

  pointcut methodsToBeProfiled():
    execution(* com.rosieapp.aop.service.SampleService.createSample(java.lang.String));

  Object around() : methodsToBeProfiled() {
    StopWatch sw = new StopWatch(getClass().getSimpleName());
    try {
        sw.start(thisJoinPoint.getSignature().getName());
        return proceed();
    } finally {
        sw.stop();
        LOGGER.info(sw.prettyPrint());
    }
  }
}
```

We only need to port one of the aspects in the project to test native support.

## Step #5.4: Configure Gradle to Execute the AspectJ Compiler
We're finally here -- the final configuration step. Unfortunately, it's also one of the hardest steps because Gradle lacks a native plug-in for AspectJ, and the plug-ins out there don't work well in a workflow where we have separate source sets for aspects.

Since we need to invoke the AspectJ tools from Gradle, we need to make sure they are pulled-in and referenceable at a specific point in the build process. This is similar to what we were trying to accomplish back in step 3.3, where we needed to reference a dependency at a very specific point before run-time. Unsurprisingly, that means we're going to use a similar tool &ndash; a configuration.

Add a new `ajc` configuration to your `build.gradle` file:
```Groovy
configurations {
    // Configuration that tracks libraries needed to run the AspectJ compiler
    ajc
    
    // ... your other configurations here...
}
```

This will let us associate the JARs containing the AspectJ tools and their dependencies with the new `ajc` configuration.

Now, let's pull in the dependencies we need (some sections have been omitted to draw attention to changes):
```Groovy
buildscript {
    ext {
        // ... your other constants here ...
        
        aspectjVersion = '1.8.10'
    }
    
    // ... the rest of your buildscript section here ...
}

dependencies {
    // ... your other dependencies here ...
    
    compile "org.aspectj:aspectjrt:$aspectjVersion"
    compile "org.aspectj:aspectjweaver:$aspectjVersion"
    ajc "org.aspectj:aspectjtools:$aspectjVersion"
}
```

This allows us to control what version of AspectJ we're pulling in, just by changing the `aspectJVersion` constant. It also makes the AspectJ runtime and weaver available when we're compiling our project, and stashes away references to the JARs containing the AspectJ tools in our `ajc` configuration, which we declared earlier.

Now, we need to actually teach Gradle how to invoke the AspectJ compiler on the files from our `aspects` source set. AspectJ exposes most of its power in the form of tools for Ant &ndash; another build process tool &ndash; and luckily, Gradle knows how to communicate with ant asks. Since we want our solution to be reusable, we'll define the code that invokes the compiler in a _closure_ that we can call after we compile our Java code.

Here's what the closure looks like in `build.gradle` (put it at the top level of the file &ndash; so it's outside all code blocks):
```Groovy
def aspectj = { sourceFileSet, destDir ->
    ant.taskdef(
        resource:  "org/aspectj/tools/ant/taskdefs/aspectjTaskdefs.properties",
        classpath: configurations.ajc.asPath
    )

    ant.iajc(maxmem:    "1024m",
             fork:      "true",
             Xlint:     "warning",
             classpath: sourceFileSet.runtimeClasspath.asPath,
             destDir:   destDir,
             source:    project.sourceCompatibility,
             target:    project.targetCompatibility) {
        sourceroots {
            sourceFileSet.java.srcDirs.each { dir ->
                pathelement(path: dir)
            }
        }
    }
}
```
This gives us a closure called `aspectj` that we can pass a source set and a destination directory, which will do all the heavy lifting!

Now, all that's left is to invoke the closure whenever we compile Java code. Here's what to add to `build.gradle` (this has to go further down in the file after the closure we defined earlier):

```Groovy
compileJava {
    doLast {
        aspectj(project.sourceSets.aspects,
                project.sourceSets.main.output.classesDir.absolutePath)
    }
}
```

This will automatically invoke the AspectJ compiler on all of the folders we defined in our `aspects` source set, and generate classes in the same output location as our main Java code. This ensures that all of our classes are discoverable and ready to go for LTW at run-time, just like back when we were using the @AspectJ syntax with POJOs in the same source set as the main code.

## Step 6: Final Test
We're here &ndash; the moment of truth. Time to run our application and see everything work.

You can run commands the same way you ran them in step #4. Most of the troubleshooting guide still applies.

When in doubt, try cleaning your project first before it runs, as follows:
```
gradle clean buildRun
```

If all goes well, the log you see from Spring should be identical to the log you saw in step #4. The main difference is now your aspects are written with the native syntax!

## Next Steps
Feel free to write more aspects and try out the effect they have on your sample project. Just remember to update `aop.xml`.

I hope this post has been helpful. Please feel free to leave a comment.

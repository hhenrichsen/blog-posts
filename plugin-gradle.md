---
title: "On Minecraft Plugins: Gradle and Reproducible Builds"
date: 2021-09-24T12:16:41-06:00
draft: false
tags:
 - Gradle
 - Minecraft
 - Java
---
## Introduction
When I was first starting out writing plugins, I would see people talking about
setups with Gradle and wonder what all the hype was about. That lasted until I
decided to try my hand at updating someone else's plugin. They had a couple 
dependencies in their code, which forced me to figure out where they all lived,
then integrate that into my IDE's environment, then build a jar properly based
on how they had organized their project.

I never wanted to do that again.

These are a few of the many things that Gradle can do for us: managing 
dependencies, building jars, and setting up projects in a way that follows a 
similar pattern for each. Additionally, it lets you hook into cool things like
running tests automatically, doing repetitive tasks, and a host of other cool
things that are difficult to do in an IDE alone, and will transfer to a new 
editor if you do decide to switch.

## What is Gradle?
So, then, what is Gradle? Gradle describes itself as a build automation tool,
which I think is accurate. It helps you compile, package, test, and build your
code in a way that can be easily repeated, even on a different machine.

This is where the term "Reproducable Builds" comes in--the builds can be
run over again, and they'll produce the same result every time. Even if they're
on another computer.

## Why use Gradle?

Aside from the benefits above, if you ever were to no longer be able to support
a project, or if you wanted the help of another developer, whether that's making
a new feature, or helping to solve a problem that you're having, it helps 
significantly when their environment is exactly the same. 

Since Gradle is also a frequently used tool in other types of development,
there's a lot of resources for it online, too. It's well-supported in all sorts
of ecosystems, like 
[GitHub Actions]({{< ref "posts/plugin-github-actions.md" >}}). That means that
same ease of creating jars and running tasks can be done for you automatically
so that you never have to go through the process of manually verifiying things
again, or releasing development builds every time you make a change.

Finally, using Gradle means that your build environment is stored in code. 
Adding a library is the same as adding a new line in any other code file, which
means it's a much more natural environment for programming than a GUI. It
also has support in every IDE, so it doesn't matter if you're using IntelliJ,
Eclipse, NetBeans, Vim, or Notepad. It'll work anywhere you have gradle 
installed, and only needs a text editor and a terminal to build your plugin.

## How do I use Gradle?

[Paper](https://github.com/PaperMC/Paper#how-to-plugin-developers) and
[Spigot](https://www.spigotmc.org/wiki/spigot-gradle/) both have good setups for
building plugins with Gradle. I use a slightly more advanced setup for most of
my projects because they depend on more libraries and have tests that they can
run before they build a Jar. I have another post that I'm working on that can 
help you set up your code to work more easily with tests, but building 
infrastructure before you need it is a good idea, so that there's less 
friction when you want to add tests.

There are a couple big ideas that we need to cover before we write any of the
files that will help us build our project.

### A little bit of history

There are three major build tools in the Java ecosystem. In order of release,
they are Ant (2000), Maven (2004), and Gradle (2007). Ant and Maven both use 
XML-based configurations that set up steps and execute them. Pretty simple. 
Maven introduced the idea of external repositories and dependencies, so you 
didn't need to go and get jar  files that you needed to build projects yourself.
This idea was carried into Gradle, and because Maven was widely adapted and 
supported, Gradle borrowed many ideas from Maven and supports using Maven
repositories and dependencies.

I've mentioned Repositories and Dependencies a little bit in here already, so
let's talk about those two next.

### Repositories

Repositories are a pretty simple concept--they're places out on the internet
that store code for use in other places. It's not that much of a different 
concept from a Git repository. You can upload and download libraries from them
easily, provided you have some pieces of identifying information about them.

### Dependencies

Dependencies are the code that can be downloaded from repositories. For Maven
and Gradle, these have three (or more) pieces of information:
1. Their Group ID, which follows the same format as a package name, for example:
`com.google.code`.
2. Their Artifact Id, which normally is the same as the project name, for 
example: `gson`.
3. Their Version, which is completely up to the package's maintainer/author to
decide on. Most people use [Semantic Versioning (SemVer)](https://semver.org/),
for example: `1.0.0`.

### Build Scripts

Build scripts are collections of the pieces below that outline sets of steps 
needed to accomplish different tasks needed to build a project. For Java, this
could be things like compiling the JavaDoc into HTML, building a jar, or 
publishing that jar to a maven repo so that other projects can depend on the
code in this project.

There are two domain-specific languages (DSLs) for Gradle build scripts; the
Groovy DSL (`build.gradle`), and the Kotlin DSL (`build.gradle.kts`). I prefer 
the Kotlin DSL because it much more closely resembles what I'm used to in 
writing Java, and because it's a little more explicit and typesafe about doing 
certain things, where in the Groovy DSL things can be super implicit and 
nonspecific about where they're coming from.

### Tasks

Tasks are the basic unit of a Gradle file. They represent a step in the process
of building your project into something that you can run. These can be as simple
as "replace this text in a file with something else", or as complex as "take 
this set of libraries, change their base packages to these base packages, and 
include them in my jar".

### Plugins

The last important thing to know about before we get into the code are Plugins.
Plugins give us a way to package tasks up together in a way that can be 
distributed to other developers really easily. Some commonly used plugins are
the `java` plugin which gives us steps to produce jars and compile Java code,
and the `jacoco` (**Ja**va **Co**de **Co**verage) plugin which gives us tasks
that tell us how much of our code is tested by the automated tests we have.

## `build.gradle.kts`
So, then. Let's use this knowledge to build a `build.gradle.kts` file. If you
don't have Gradle installed on your computer, now would be a good time to get
that set up. [Here's a guide on that](https://gradle.org/install/).

The easiest way to get a super-simple `build.gradle.kts` file is to run
`gradle init` in a project folder, ideally before you've written anything in 
the project. This way, it sets up the project structure for you. This should
still work in existing projects, but you might have to move things around if
they're not set up in the right places.

### Using `gradle init`

First I'm going to make a folder for my project.

{{< terminal >}}
mkdir SampleProject
cd SampleProject
{{< /terminal >}}


Next, I'll run `gradle init` and see this prompt:

{{< terminal >}}
gradle init
{{< /terminal >}}

{{< output >}}
Select type of project to generate:
  1: basic
  2: application
  3: library
  4: Gradle plugin
Enter selection (default: basic) [1..4]
{{< /output >}}

For this case, I'm going to choose `1` since we'll be writing our own 
configurations for our plugin. Next we choose our DSL:

{{< output >}}
Select build script DSL:
  1: Groovy
  2: Kotlin
Enter selection (default: Groovy) [1..2]
{{< /output >}}

I prefer the Kotlin DSL, so I'm going to choose `2`. This will create a bunch of
files for us:

```
SampleProject/
├── build.gradle.kts
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── settings.gradle.kts
```

This includes our `build.gradle.kts` file, a `settings.gradle.kts` file that
sets the project name, and the gradle wrapper which helps us to keep a
consistent gradle version for this project regardless of which machine we run it
on.

{{< filelabel settings.gradle.kts >}}
```kotlin {linenos=table}
rootProject.name = "SampleProject"
```
{{< /filelabel >}}

### Doing It Yourself

Doing it yourself just means making a couple of files and running a slightly
different gradle command. First, make a `settings.gradle.kts` file:

{{< filelabel settings.gradle.kts >}}
```kotlin {linenos=table}
rootProject.name = "SampleProject"
```
{{< /filelabel >}}

Then make an empty `build.gradle.kts` file.

Then run the command to generate the Gradle wrapper:

{{< terminal >}}
gradle wrapper
{{< /terminal >}}

And you're all set!

### Building the `build.gradle`

Now let's start working on writing our file using some of the concepts we talked
about above. First, let's load a plugin that makes it a lot easier to deal with
Java, and set up some fundamental information:

{{< filelabel build.gradle.kts >}}
```kotlin {linenos=table}
plugins {
  java
}

// The main package name
group = "me.hhenrichsen"
// Using SemVer instead of SNAPSHOT versioning
version = "1.0.0"
```
{{< /filelabel >}}

Next, let's get some repositories in to let us get Spigot:

{{< filelabel build.gradle.kts >}}
```kotlin {linenos=table}
plugins {
  java
}
 
// The main package name
group = "me.hhenrichsen"
// Using SemVer instead of SNAPSHOT versioning
version = "1.0.0"

repositories {
  // Used for anything we've build with BuildTools
  mavenLocal()

  // Used for Spigot-API and non-NMS Spigot
  maven(url = "https://hub.spigotmc.org/nexus/content/repositories/snapshots/") 
}

dependencies {
  // Request the Spigot API.
  implementation("org.spigotmc:spigot-api:1.17-R0.1-SNAPSHOT")
}
```
{{< /filelabel >}}

What we're saying here is look for things locally first, in our `mavenLocal`
repository. If we can't find it there, go look in another maven repository--
the one located at the URL we give it.

Then we tell Gradle what we're looking for--in this case, Spigot. And that's all
we need to be able to compile and package a plugin. Much less typing than
Maven's XML, and pretty straightforward as far as things go. The one thing that
might be unusual here is the `implementation` function call. That's saying that
we require this dependency both when we compile this project, and that it's
going to be given to us when we run this project. There are a couple other of 
these functions called *configurations* that do a variety of things, but the
main three that I use are `implementation`, `testImplementation`, and
`testRuntimeOnly`.

And interestingly enough, that's all we need to turn this project into a jar.

{{< terminal >}}
gradle jar
{{< /terminal >}}

If all you have is what we've set up above, this will go through, but give you
a very empty jar. If we wanted to add code to our project, this is the
structure it would take:

```
SampleProject/
├── build.gradle.kts
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle.kts
└── src
    └── main
        ├── java
        │   └── me
        │       └── hhenrichsen
        │           └── SamplePlugin.java
        └── resources
            └── plugin.yml
```

And here are my super-simple files:

{{< filelabel "src/main/java/me/hhenrichsen/SamplePlugin.java" >}}
```java {linenos=table}
package me.hhenrichsen;

import org.bukkit.plugin.java.JavaPlugin;

public class SamplePlugin extends JavaPlugin { }
```
{{< /filelabel >}}

{{< filelabel "src/main/resources/plugin.yml" >}}
```yaml {linenos=table}
name: SamplePlugin
main: me.hhenrichsen.SamplePlugin
version: 1.0.0
```
{{< /filelabel >}}

With the Java plugin, gradle will automatically compile anything in the
src/main/java folder into your jar, and copy anything in the resources folder
into your jar. `me.hhenrichsen` in this case is my base package. Notice how it
matches my `group` in the gradle file up above. Your base package and group
should match as well. In my own projects I like to introduce another package
between my base group and the rest of the project, like
`me.hhenrichsen.sampleplugin`.

## Leveling Up Beyond the Basics

Now that we've got plugins, repositories, and dependencies covered, let's talk
about tasks. I said earlier that tasks let us do repetitive things, and one 
example of that is updating strings like the version string. I want that to be
in one place, and for Gradle to copy that everywhere else that it's needed.

To do that, we need to look in a task called `processResources` that 
unsurprisingly lets us configure the steps of how resources are processed.

First, I'm going to add a new file called `gradle.properties`. This is a
great place to store settings that are prone to change, like plugin versions,
base spigot versions, and the versions of libraries.

{{< filelabel "gradle.properties" >}}
```text {linenos=table}
version=1.0.0
```
{{< /filelabel >}}

Next, I'm going to modify my `build.gradle.kts` to pull from those properties:

{{< filelabel build.gradle.kts >}}
```kotlin {linenos=table,hl_lines=["7-8", "27-30"]}
plugins {
  java
}
 
// The main package name
group = "me.hhenrichsen"
// Pull version from gradle.properties
val version: String by project

repositories {
  // Used for anything we've build with BuildTools
  mavenLocal()

  // Used for Spigot-API and non-NMS Spigot
  maven(url = "https://hub.spigotmc.org/nexus/content/repositories/snapshots/") 
}

dependencies {
  // Request the Spigot API.
  implementation("org.spigotmc:spigot-api:1.17-R0.1-SNAPSHOT")
}

// If you wanted to expand author to "UberPilot", you'd write
// expand("version" to version, "author" to "UberPilot")
// in the task below.

// Replace ${version} in resources with the version
tasks.processResources {
  expand("version" to version)
}
```
{{< /filelabel >}}

I'll also update my very basic `plugin.yml` at this point:

{{< filelabel "src/main/resources/plugin.yml" >}}
```yaml {linenos=table,hl_lines=[3]}
name: SamplePlugin
main: me.hhenrichsen.SamplePlugin
version: ${version}
```
{{< /filelabel >}}

If I update the version to `1.0.5` in `gradle.properties`, and run these
commands:

{{< terminal >}}
gradle clean
gradle jar
{{< /terminal >}}

We get a new jar produced in `build/libs/SamplePlugin-1.0.5.jar`. If I were to
drop it into a server, I'd get this message on startup:

{{< output >}}
[SamplePlugin] Loading SamplePlugin v1.0.5
{{< /output >}}

So not only did it update the project's version, but it also moved that version
into the `plugin.yml` which can then be used from within the plugin!

You can also try moving the Spigot API version into that properties file. That
one is relatively simple to do, especially if you use [String 
Interpolation](https://kotlinlang.org/docs/idioms.html#string-interpolation).

## Takeaways

This is just one of the cool automated things you can do with Gradle, but seeing
that I could do things like this was one of the reasons that I swapped over to
Gradle from Maven. Another cool thing you can do with Gradle is include
libraries in your jar, or run tests and check coverage. Each of these take a 
little more setup, so I'll write some more posts on them later.
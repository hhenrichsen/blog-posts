---
title: "On Minecraft Plugins: Gradle and Reproducible Builds"
date: 2021-09-23T12:16:41-06:00
draft: true
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

## Some Best Practices
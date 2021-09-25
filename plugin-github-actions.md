---
title: "On Minecraft Plugins: Github Actions"
date: 2021-09-22T18:35:23-06:00
draft: false
tags:
 - Gradle
 - Java
 - Minecraft
 - GitHub
---

## Introduction
If you're hosting your code on GitHub and not using GitHub Actions, you're
missing out. (Aside: You should be hosting your code somewhere, since there are
a lot of benefits from being backed up off of your computer, especially if you
use an open source license that allows other people to fork or propose changes
to your code.) GitHub Actions will do all sorts of cool things for you as soon
as you push your code, including running tests, building development jars,
checking test coverage, and even sending Discord notifications. That's also one
way you can get those fancy "Build: Passing" badges on your plugin and GitHub
pages.

In this post I'll take you through the basics of getting GitHub actions running
tests and building Jars. If you need a good gradle setup and don't have existing
code, I have one [here](https://github.com/ShatteredSoftware/KotlinPlugin) (plus
that one will let you use Kotlin, and already has the GitHub actions installed
on it). Otherwise, you might take a look at my other post 
[here](/posts/plugin-gradle) for more info on building your
own.

## The Basic Gradle Setup

*If you already have a working Gradle setup, feel free to skip to 
[here]({{< relref "#a-simple-action" >}}).*

{{< filelabel "build.gradle.kts" >}}
```kotlin 
// We're developing for Java
plugins {
    java
}

// The main package name
group = "shattered.software"
// Using semver instead of SNAPSHOT versioning
version = "1.0.0"

// Places we can get libraries
repositories {
    // Use local versions first
    mavenLocal()
    // Look in Spigot's repos next
    maven(url = "https://hub.spigotmc.org/nexus/content/repositories/snapshots/")
    // Look in Maven Central last
    mavenCentral()
}

// Libraries we need
dependencies {
    // We want Spigot
    implementation("org.spigotmc:spigot-api:1.17-R0.1-SNAPSHOT")
}
```
{{< /filelabel >}}

I commented most of that file, but the idea is that we say "these are the 
places we can get the things we need", then say "these are the things
we need", and use some processes specific to Java. Ideally we also have
a gradle wrapper in the folder. If you have gradle installed on your 
computer you can run `gradle wrapper` and that'll make it easier to run
and build the project in environments where the gradle version isn't the
same.

**Pro Tip:** Make sure that your gradle wrappers are marked as executable to git
by running:

{{< terminal >}}
git update-index --chmod=+x ./gradlew
git update-index --chmod=+x ./gradlew.bat
{{< /terminal >}}

Source files (Java files) are stored in the `src/main/java` directory. 
Resource files (like `plugin.yml`) are stored in the `src/main/resources`
directory. 

For example, my main plugin file would live at 
`src/main/java/software/shattered/TestPlugin.java` and my `plugin.yml` would
live at `src/main/resources/plugin.yml`.

What does this let us do? In the terminal, we can run this command:

{{< terminal >}}
gradle jar
{{< /terminal >}}

And it'll spit out a jar file in the `build/libs` folder. There are a bunch
of improvements we can make to this process, but I'll leave that to another
post.

## A Simple Action

So let's get to the point, then. We're here to do some of the cool things you
can do with GitHub actions. First, let's start off by making an individual
action, or workflow, that'll build a jar for us every time we push code to
GitHub.

To start, we need to make the folder where these files will live, 
`.github/workflows`. 

{{< terminal >}}
mkdir .github/workflows
{{< /terminal >}}

In that folder, let's make a file named `prerelease.yml`. 

{{< filelabel "prerelease.yml" >}}
```yaml {linenos=table,linenostart=1}
---
name: "Prerelease"
on: 
  push:
    # Run on the main branch
    branches: 
      - "main"
    # Ignore documentation
    paths-ignore:
      - "README.md"
      - "docs/**"

jobs:
  pre-release:
    name: "Pre-release"
    runs-on: ubuntu-latest
```
{{< /filelabel >}}

This sets up some basic settings before we get to the meat of the action; what
it runs on, when it runs, what files it should ignore, etc. Now we should add
some build steps.

{{< filelabel "prerelease.yml" >}}
```yaml {linenos=table}
name: "Prerelease"
on: 
  push:
    # Run on the main branch
    branches: 
      - "main"
    # Ignore documentation
    paths-ignore:
      - "README.md"
      - "docs/**"

jobs:
  pre-release:
    name: "Pre-release"
    runs-on: ubuntu-latest
    steps:
      # clone our repository into the runner
      - uses: actions/checkout@v2
      # install java on the runner
      - name: Setup JDK 16
        uses: actions/setup-java@v1
        with:
          java-version: 16
      # check that gradle is set up with a wrapper correctly
      - name: Validate Gradle Wrapper
        uses: gradle/wrapper-validation-action@v1 
      # make sure that our plugin builds
      - name: Build Project
        run: ./gradlew build
      # build the jars
      - name: Build Jars
        run: ./gradlew jar
      # create a development build
      - name: Release
        uses: "marvinpinto/action-automatic-releases@latest"
        with:
          repo_token: "${{ secrets.GITHUB_TOKEN }}"
          automatic_release_tag: "latest"
          prerelease: true
          title: "Development Build"
          files: |
            build/libs/*.jar
```
{{< /filelabel >}}

This is a lot to parse through, so let's start at the beginning. We're adding
a section that's a list of steps. Each step does something important to the
build process. In order, they:

1. Clone our project to the machine that the steps are running on.
2. Set up Java on that machine.
3. Checks that the gradle wrapper is set up properly.
4. Uses the gradle wrapper to `./gradlew build` the project, compiling it.
5. Uses the gradle wrapper to `./gradlew jar` the project, packaging it into a
jar.  
6. Uploads the jar to a new release on GitHub.

While this sort of build will work for most projects, I like to break mine up
and have a couple of other steps for testing my code. I also like to set it up
so that normal commits go to a Pre-release, and then when I tag a certain
commit with a version, it goes to a full release.

## Full Release

So now my goal is to do a full release when I run `git tag 1.0.0` or any other
combination of numbers. My first step is to edit the Prerelease workflow so 
that it doesn't run when I'm doing a full release. I need to do some more 
complex operations with Gradle to accomplish some of these things, so here's
a new `build.gradle.kts`.

### The New `build.gradle.kts`

{{< filelabel "build.gradle.kts" >}}
```kotlin {linenos=table}
// We're developing for Java
plugins {
    java
    jacoco // Java Code Coverage
}

// The main package name
group = "shattered.software"
// Using semver instead of SNAPSHOT versioning
version = "1.0.0"

// Places we can get libraries
repositories {
    // Use local versions first
    mavenLocal()
    // Look in Spigot's repos next
    maven(url = "https://hub.spigotmc.org/nexus/content/repositories/snapshots/")
    // Look in Maven Central last
    mavenCentral()
}

// Libraries we need
dependencies {
    // We want Spigot
    implementation("org.spigotmc:spigot-api:1.17-R0.1-SNAPSHOT")

    testImplementation("org.junit.jupiter:junit-jupiter-api:5.8.0")
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine:5.8.0")
}

// Get tests to work with JUnit 5.
tasks.withType<Test> {
    useJUnitPlatform()
}

// Produce code coverage reports.
tasks.withType<JacocoReport> {
    reports {
        xml.required.set(true)
        html.required.set(false)
    }
}

```
{{< /filelabel >}}

### The Workflow Files
Now onto the workflow files. For `prerelease.yml`, I only want to run the build
when it's not a full release, so I should tell it to ignore tags that match 
those that will be run when I run `release.yml`.

{{< filelabel "prerelease.yml" >}}
```yaml {linenos=table}
---
name: "Prerelease"
on: 
  push:
    # Run on the main branch
    branches: 
      - "main"
    # Ignore full releases
    tags-ignore:
      - "*.*.*"
    # Ignore documentation
    paths-ignore:
      - "README.md"
      - "docs/**"

jobs:
```
{{< /filelabel >}}

Next, I'm going to set up a new workflow for my full releases:

{{< filelabel "release.yml" >}}
```yaml {linenos=table}
---
name: "Full Release"

on:
  push:
    # Only run when the pushed commit is tagged with #.#.#.
    tags:
      - "*.*.*"

jobs:
  tagged-release:
    name: "Full Release"
    runs-on: "ubuntu-latest"
    steps:
      # clone our repository into the runner
      - uses: actions/checkout@v2
      # install java on the runner
      - name: Setup JDK 16
        uses: actions/setup-java@v1
        with:
          java-version: 16
      # check that gradle is set up with a wrapper correctly
      - name: Validate Gradle Wrapper
        uses: gradle/wrapper-validation-action@v1 
      # make sure that our plugin builds
      - name: Build Project
        run: ./gradlew build
      # run any tests
      - name: Test Project
        run: ./gradlew test
      # see how much of the code is tested
      - name: Code Coverage
        uses: codecov/codecov-action@v2
      # build the jars
      - name: Build Jars
        run: ./gradlew jar
      # create a full release
      - name: Release
        uses: "marvinpinto/action-automatic-releases@latest"
        with:
          repo_token: "${{ secrets.GITHUB_TOKEN }}"
          prerelease: false
          title: "Development Build"
          files: |
            build/libs/*.jar
```
{{< /filelabel >}}

Now I can run `git tag 1.0.0` and then `git push --tags` and I'll get a nice
fancy release set up on my GitHub page, and a coverage report generated for
my tests that lets me know what I'm missing. The release will stick around
forever as well, so I have an archive of every version that I've built so far.

## Takeaways

Not only does the gradle setup help you to let anyone else build your plugin
without too many problems, but it lets you hook into these cool GitHub actions
that can help you to do many of the tedious parts of the development process.

You can do a lot of really cool things with GitHub actions. I primarily use
them for the above, to build releases for me and to tell me where I'm missing
test coverage so that I can have a plugin that's less buggy.
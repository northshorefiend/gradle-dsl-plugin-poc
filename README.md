# A gradle plugin written in gradle DSL rather than Java or Kotlin

## TL;DR
1. Build the plugin with the commands in [plugin/README.md](plugin/README.md)
2. Build the example project with the commands in [example/README.md](example/README.md)

## Why a Plugin in Gradle DSL
I've been holding out switching to Gradle for far too long. I know Maven can be annoying, but for most simple projects
it is fine and concise. OK, so XML has fallen out of favour now, but it works.

I recently took over a project that used Gradle to integrate with [SonarQube](https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/scanners/sonarscanner-for-gradle/),
[NexusIQ](https://blog.sonatype.com/new-sonatype-scan-gradle-plugin) and a couple of other things. Gradle
was also used to version the artefacts using the [nebula plugins](https://nebula-plugins.github.io/).

This setup was used in a couple of projects by different teams in the organisation by just copying all the Gradle code,
then making it work with the new project. Being new to Gradle, I wanted to start my journey by doing things properly,
so on reading the [Gradle docs](https://docs.gradle.org/current/userguide/custom_plugins.html) I wanted to create
a custom plugin.

## Gradle Plugin
Now, we are all busy people, and I didn't want to have to rewrite the working Gradle DSL in Kotlin, Java or Groovy. I
saw in the docs you could write plugins in Gradle Groovy DSL. Though from the docs, it looks like you can only do this
using the buildSrc method. This is where you put a directory called `buildSrc`
in the root of your Gradle project and write your plugin there. But that doesn't solve my problem; I want a standalone
plugin that I can build separately and use in multiple projects. So, I had the question, can you make a standalone
Gradle DSL plugin? Well, with a bit of trial and error I worked it out.

## Gradle DSL standalone plugin
This all works by convention. Firstly, let's start with the `build.gradle` for the plugin:
```
plugins {
    id 'groovy-gradle-plugin'
    id 'maven-publish'
}

group 'com.example.java-conventions'
version '0.1'
```

Here, you need the `groovy-gradle-plugin` that compiles and packages to code as a Gradle plugin. Then, we need to talk about
names: If you want your plugin to have the id `com.example.java-conventions` then that needs to be set as the group
in your `build.gradle`. You will also need to set a `version`. I have included the `maven-publish` plugin so that we can
use the local `~/.m2` repository to test our new plugin locally.
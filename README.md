# A Gradle plugin written in Gradle DSL rather than Java or Kotlin

## TL;DR
1. Get the code from [my github]()
2. Build the plugin with the commands in [plugin/README.md](./plugin/README.md)
3. Build the example project with the commands in [example/README.md](./example/README.md)

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

Next we need to write the actual DSL of the plugin. This, again, follows a convention. If you want the id of your plugin
to be `com.example.java-conventions` then you need to create a file in:

```src/main/groovy/com.example.java-conventions.gradle```

In this we put our plugin code.

### Writing your plugin
First, let's put in the basics. We're also going to configure the checkstyle plugin
as an example of what you can include.

```
plugins {
    id 'java-library'
    id 'checkstyle'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
```

Next we need to configure the checkstyle plugin. This plugin needs a config file. I show here a way you can do this.

```
checkstyle {
    configFile file("${rootDir}/build/tmp/google_checks.xml")
    toolVersion '10.12.6'
}

// these two stanzas copy the config file from the plugin resources to the project build dir ready for the above
// checkstyle config

tasks.register('copyCheckStyleConfig') {
    outputs.file('build/myConfig/google_checks.xml')
    doLast {
        copy {
            from resources.text.fromArchiveEntry(buildscript.configurations.classpath, "google_checks.xml").asFile()
            into 'build/myConfig'
        }
    }
}

tasks.named('checkstyleMain') {
    dependsOn('copyCheckStyleConfig')
}
```

I am configuring checkstyle to use a config file that is in the plugin resources `src/main/resources/google_checks.xml`.
It is copied by the copyCheckStyleConfig task I have created. 
It's a good example of how you can configure something once in the plugin and roll it out to all your projects just by
bumping the version of the plugin they are pulling in.

To test our plugin locally, we will publish it to the local Maven repo (~/.m2):

```./gradlew build publishToMavenLocal```

## Using the plugin
You use the plugin like any other. In your project's `settings.gradle` you need to configure where you are getting your 
plugins from:
```
pluginManagement {
    repositories {
        mavenLocal()
        gradlePluginPortal()
    }
}
```

Here, I have configured the local Maven repository, and also the central Gradle plugin repository for the checkstyle
plugin we are using. Next, in `build.gradle` you need to include the plugin:

```
plugins {
    id 'com.example.java-conventions' version '0.1'
}

repositories {
    mavenCentral()
}
```

We also need to specify somewhere to get the dependencies of the checkstyle plugin from, here mavenCentral().

And that is it, you have created a plugin using the Gradle groovy dsl! In our case, running a build will
invoke the checkstyle plugin with our custom config:

```
$ ./gradlew clean build

> Task :checkstyleMain
[ant:checkstyle] [WARN] /home/northshorefiend/git/gradle-dsl-plugin-poc/example/src/main/java/com/example/hello/Main.java:3:1: Missing a Javadoc comment. [MissingJavadocType]
[ant:checkstyle] [WARN] /home/northshorefiend/git/gradle-dsl-plugin-poc/example/src/main/java/com/example/hello/Main.java:4:5: 'method def modifier' has incorrect indentation level 4, expected level should be 2. [Indentation]
[ant:checkstyle] [WARN] /home/northshorefiend/git/gradle-dsl-plugin-poc/example/src/main/java/com/example/hello/Main.java:5:13: 'method def' child has incorrect indentation level 12, expected level should be 4. [Indentation]
[ant:checkstyle] [WARN] /home/northshorefiend/git/gradle-dsl-plugin-poc/example/src/main/java/com/example/hello/Main.java:6:5: 'method def rcurly' has incorrect indentation level 4, expected level should be 2. [Indentation]
Checkstyle rule violations were found. See the report at: file:///home/northshorefiend/git/gradle-dsl-plugin-poc/example/build/reports/checkstyle/main.html
Checkstyle files with violations: 1
Checkstyle violations by severity: [warning:4]


BUILD SUCCESSFUL in 2s
5 actionable tasks: 5 executed
```

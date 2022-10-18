---
title:  "Including the Git Commit Hash in a Runnable Jar"
date:   2021-10-22
tags: software-engineering maven java
---

Because implementing your system and running experiments with it can happen concurrently,
it is important to keep track of which version of your code you used for which experiment.
With a fast development cycle, that can get tricky.

Fortunately, this problem is easy to solve when working with Maven. Using the `buildnumber-maven-plugin`,
you can automatically add the current git commit hash to your JAR file.
If you keep your JAR around, you can always tell from which commit it was built.

First declare the plugin in your `pom.xml`:

```xml
<!-- Version control information used by the buildnumber-maven-plugin -->
<scm>
    <developerConnection>scm:git:git@github.com:username/repo.git</developerConnection>
    <url>https://github.com/username/repo</url>
</scm>

<build>
    <plugins>
        <!-- Plugin that lets us package the git commit hash in the executable JAR -->
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>buildnumber-maven-plugin</artifactId>
            <version>1.4</version>
            <executions>
                <execution>
                    <phase>validate</phase>
                    <goals>
                        <goal>create</goal>
                    </goals>
                    <configuration>
                        <doCheck>true</doCheck>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

The plugin requires the `scm` tag, which tells it which source control management (SCM) tool you are using and where to find the code repository.
The execution configures the plugin to add the commit hash to the JAR during Maven's validate phase.

The `doCheck` parameter checks that there are no uncommitted changes in the repository. This is important, because you identify the JAR based on the commit hash.
If you can build the code with uncommitted changes, these builds are no longer unique.
If you do need to build your project without committing, you can use the flag `-Dmaven.buildNumber.doCheck=false`.

Now, use the following code to store the commit hash in your JAR file.
The `buildnumber` plugin makes the commit hash available as `${buildNumber}` in your POM file,
which you can pass to the `maven-jar-plugin` to add the value to your manifest file.

```xml
<!-- A plugin that places the buildNumber (i.e., git hash) in the generated JAR archive -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.2.0</version>
    <configuration>
        <archive>
            <manifest>
                <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
            </manifest>
            <manifestEntries>
                <Implementation-Build>${buildNumber}</Implementation-Build>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

The result:

```bash
$ unzip -p target/build-dev.jar META-INF/MANIFEST.MF
Manifest-Version: 1.0
Created-By: Maven Jar Plugin 3.2.0
Build-Jdk-Spec: 11
Implementation-Title: dist
Implementation-Version: dev
Implementation-Build: 09fcf03c5037fa4ee7699a112f77f70f00d435da
```

This JAR file was built from commit `09fcf03c5037fa4ee7699a112f77f70f00d435da`!

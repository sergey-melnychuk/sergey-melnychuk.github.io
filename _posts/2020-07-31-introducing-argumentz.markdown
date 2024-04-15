---
layout: post
title:  "Argumentz: small & simple command-line arguments parser in Java"
date:   2020-07-31 09:35:42 +0200
categories: java command-line arguments parser
---
Every now and then there is a need to pass arguments to the command-line application. While there are a lot of libraries/frameworks to do that in Java, I still had this feeling "why such an easy thing must be so cumbersome and require so much boilerplate?".

TL/DR: [code][argumentz-github].

So I decided: the world needs one more command-line arguments parser for Java. Why? Well, maybe just because I can! First thing to decide on a new project is a name: 'argumentz' was available (according to maven central and one popular web search engine). 'Z' in 'Argumentz' stands for the last letter in the alphabet: I wanted to make the library as simple as possible (but not simpler) and keep user in the total control, so there cannot be anything simpler beyond it, like there are no letters after Z.

### Example application

The example application takes following command-line arguments: host (string), port (integer), timeout in seconds (integer), user name (string) and a flag to enable verbose logging. So the execution will look like this:

```bash
java -cp . App -u admin -h localhost -p 8080 -s 30 -v
```

Or with full names of flags:

```bash
java -cp . App --user admin --host localhost --port 8080 --seconds 30 --verbose
```

The default value for `user` is "guest", for `port` 8080, so if one of those arguments is missing, application will use default values. Each flag has a default value as false, meaning it is disabled until explicitly enabled, pretty standard behavior for a flag.

### Setup

To start, checking out prepared empty project from [empty-project][empty-project-github] and slightly cleaning it up:

```bash
$ git clone https://github.com/sergey-melnychuk/empty-project.git \
    -b maven-java8-junit5 \
    args-parser
$ cd args-parser
$ rm -rf .git   # remove any association with 'empty-project' repository
```

Adding dependency to `pom.xml`:

```xml
<dependency>
  <groupId>io.github.sergey-melnychuk</groupId>
  <artifactId>argumentz</artifactId>
  <version>0.3.9</version>
</dependency>
```

### Describe command-line arguments

To describe desired setup in Arguments, `Arguments.builder()` is used:

```java
Argumentz arguments = Argumentz.builder()
        .withParam('u', "user", "username to connect to the server", () -> "guest")
        .withParam('p', "port", "port for server to listen", Integer::parseInt, () -> 8080)
        .withParam('s', "seconds", "timeout in seconds", Integer::parseInt)
        .withParam('h', "host", "host for client to connect to")
        .withFlag('v', "verbose", "enable extra logging")
        .withErrorHandler((RuntimeException e, Argumentz a) -> {
            System.err.println(e.getMessage() + "\n");
            a.printUsage(System.out);
            System.exit(1);
        })
        .build();
```

So each argument is either a parameter (`String` or actually any type that can be parsed from it) or a flag (just a parameter of type `boolean`). Adding parameter with short form `-p` and long one `--param` can be done in one of the following ways:

- Required, of type `String`: `.withParam('p', "param", "description goes here")`
- Required, produced from `String` (in this case `int`): `.withParam('p', "param", "...", Integer::parseInt)`
  - Any type `T` is supported, as long as instance of `Function<String, T>` is provided.
- Not required `String` (thus having default value): `.withParam('p', "param", "...", () -> "default")`
- Not required of any type (e.g. `int`): `.withParam('p', "param", "...", Integer::parseInt, () -> 42)`

Flag is defined with just `.withFlag('f', "flag", "...")`. Distinguish flags between mandatory or optional does not make much sense to me, as well as providing a default value (it is just `false`). So flag, is just, well, a flag - either it is there or not.

Error handler provided with `.withErrorHandler` is called when the problem arises (missing required argument or parsing arguments value from `String` failed). In this example, the error message is printed to `stderr`, usage instructions are printed to `stdout` and application terminates. One required thing for such error handler to do is to break the execution flow of an application: either throw a runtime exceptions or call `System.exit`. Reasons for this requirement are simple: fail fast and let user keep the control.

### Match against provided arguments

After there is a definition of `Argumentz`, it is possible to `match` arguments from `String args[]`:

```java
Argumentz.Match match = arguments.match(args);
```

After that, all matched values are available from the `match` instance. No binding to class properties via annotattions, just plain and simple `.get{,Int,Flag}(String name)`, inspired by [Lightbend Config][lightbend-config]. Getting all the actual values is pretty straightforward:

```java
String user = match.get("user");
int port = match.getInt("port");
String host = match.get("host");
int seconds = match.getInt("seconds");
boolean verbose = match.getFlag("verbose");
```

For generic value of type `T`, use `match.getAs(Class<T> clazz, String name)`. For example if `T` is `Param`, the call is `match.getAs(Param.class, "param")`. The casting from `Object` to `T` is done by `Class<T>::cast`, so if anything goes wrong, `ClassCastException` will land into error handler.

### Error handling

If an exception is re-thrown from error handler, it will be thrown from `arguments.match`. As promised, the user keeps total control under the situation - how to handle the exception is totally up to the user. For example, with `try-catch` around `arguments.match` it is easy to extract into pure function all code related to the command-line arguments:

```java
static Optional<Params> fetchParams(String args[]) {
    Argumentz arguments = Argumentz.builder()
        // ...
        .withErrorHandler((RuntimeException e, Argumentz a) -> { throw e; })
        .build();

    try {
        Argumentz.Match match = arguments.match(args);
    
        String user = match.get("user");
        int port = match.getInt("port");
        String host = match.get("host");
        int seconds = match.getInt("seconds");
        boolean verbose = match.getFlag("verbose");
    
        return Optional.of(new Params(host, port, user, seconds, verbose));

    } catch (IllegalArgumentException e) {
        log.error("Failed to fetch Params.", e);
        return Optional.empty();
    }
}
```

### Summary

I was presenting [Argumentz][argumentz-github], very small, simple and flexible library for parsing command line arguments in Java. It is pure Java, has zero dependencies and has <300 LoC. In the example app, code that handles command-line arguments is straightforward, easy and leaves user in charge.

If you found a bug or a missing feature, please do let me know in comments at the bottom of the page.

### P.S. Publishing to Maven Central

The process is pretty much described [here][deploy-guide].

I was trying to follow it, but still there were some issues along the way. Below is the summary of what I did to achieve successful release and promotion to Maven Central.

#### Sonatype JIRA

1. Create an account
1. Create a [ticket][argumentz-sonatype-jira]

#### GPG

Described [here][sonatype-gpg].

```bash
$ gpg2 --version
...
$ gpg2 --gen-key
...
$ gpg2 --list-keys
...
pub   rsa2048 2020-07-29 [SC] [expires: 2022-07-29]
      <key-id-goes-here>
uid           [ultimate] Sergey Melnychuk <sergey-melnychuk@users.noreply.github.com>
sub   rsa2048 2020-07-29 [E] [expires: 2022-07-29]
...

$ gpg2 --keyserver hkp://pool.sks-keyservers.net --send-keys <key-id>
gpg: sending key <key-id> to hkp://pool.sks-keyservers.net
```

#### pom.xml

```xml
...
  <groupId>io.github.sergey-melnychuk</groupId>
  <artifactId>argumentz</artifactId>
  <version>0.3.9-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>${project.groupId}:${project.artifactId}</name>
  <description>Command-line arguments parser in Java. Small, simple and flexible.</description>
  <url>https://github.com/sergey-melnychuk/argumentz</url>

  <licenses>
    <license>
      <name>Apache License, Version 2.0</name>
      <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
      <distribution>repo</distribution>
    </license>
  </licenses>

  <developers>
    <developer>
      <name>Sergey Melnychuk</name>
      <url>https://github.com/sergey-melnychuk/</url>
    </developer>
  </developers>

  <scm>
    <connection>scm:git:git://github.com/sergey-melnychuk/argumentz.git</connection>
    <developerConnection>scm:git:git@github.com:sergey-melnychuk/argumentz.git</developerConnection>
    <url>http://github.com/sergey-melnychuk/argumentz/tree/master</url>
  </scm>
...
```

```xml
...
  <distributionManagement>
    <snapshotRepository>
      <id>ossrh</id>
      <url>https://oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    <repository>
      <id>ossrh</id>
      <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
  </distributionManagement>
...
```

```xml
...
  <profiles>
    <profile>
      <id>release</id>
      <build>
        <plugins>
          <plugin>
            <groupId>org.sonatype.plugins</groupId>
            <artifactId>nexus-staging-maven-plugin</artifactId>
            <version>1.6.7</version>
            <extensions>true</extensions>
            <configuration>
              <serverId>ossrh</serverId>
              <nexusUrl>https://oss.sonatype.org/</nexusUrl>
              <autoReleaseAfterClose>true</autoReleaseAfterClose>
            </configuration>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>2.2.1</version>
            <executions>
              <execution>
                <id>attach-sources</id>
                <goals>
                  <goal>jar-no-fork</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>2.9.1</version>
            <executions>
              <execution>
                <id>attach-javadocs</id>
                <goals>
                  <goal>jar</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <version>1.5</version>
            <executions>
              <execution>
                <id>sign-artifacts</id>
                <phase>verify</phase>
                <goals>
                  <goal>sign</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-release-plugin</artifactId>
            <version>2.5.3</version>
            <configuration>
              <autoVersionSubmodules>true</autoVersionSubmodules>
              <useReleaseProfile>false</useReleaseProfile>
              <releaseProfiles>release</releaseProfiles>
              <goals>deploy</goals>
            </configuration>
          </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>
...
```

#### ~/.m2/settings.xml

```xml
<settings>
  <servers>
    <server>
      <id>ossrh</id>
      <username>jira-username</username>
      <password>jira-password</password>
    </server>
  </servers>
  <profiles>
    <profile>
      <id>ossrh</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <gpg.executable>gpg2</gpg.executable>
        <gpg.passphrase>***</gpg.passphrase>
      </properties>
    </profile>
  </profiles>
</settings>
```

#### Release

```bash
$ mvn release:clean release:prepare -P release
$ mvn release:perform -P release
```

#### Artifacts

- [Sonatype][artifact-sonatype]
- [Central][artifact-central]

[empty-project-github]: https://github.com/sergey-melnychuk/empty-project
[lightbend-config]: https://github.com/lightbend/config
[argumentz-github]: https://github.com/sergey-melnychuk/argumentz
[deploy-guide]: https://central.sonatype.org/pages/ossrh-guide.html
[argumentz-sonatype-jira]: https://issues.sonatype.org/browse/OSSRH-59588
[sonatype-gpg]: https://central.sonatype.org/pages/working-with-pgp-signatures.html
[artifact-sonatype]: https://oss.sonatype.org/service/local/repositories/releases/content/io/github/sergey-melnychuk/argumentz/0.3.9/
[artifact-central]: https://search.maven.org/artifact/io.github.sergey-melnychuk/argumentz/0.3.9/jar

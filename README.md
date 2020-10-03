# Dependency Management at Scale - JConf Mexico 2020

This repository contains demos for "Dependency Management at Scale" talk @ JConf Mexico 2020

Slides can be found in [https://www.slideshare.net/rperezalcolea/dependency-management-at-scale](https://www.slideshare.net/rperezalcolea/dependency-management-at-scale)

## Structure

This project has been structured via git tags:

* `init`
* `resolution-rules-usage`
* `dependency-locks`
* `lint-unused-dependencies`
* `lint-undeclared-dependencies`

### init

This tag contains the project skeleton. Also, it contains a set of modules for the examples. These modules can be found in the `repo` folder which follows Maven repo structure.

To check this project execute the following:

```
git checkout init
./gradlew dependencies
```

You should see the following dependency tree:

```
+--- netflix:foo-client:latest.release -> 1.0.1
|    \--- com.amazonaws:aws-java-sdk-dynamodb:1.11.870
|         +--- com.amazonaws:aws-java-sdk-s3:1.11.870
|         |    +--- com.amazonaws:aws-java-sdk-kms:1.11.870
|         |    |    +--- com.amazonaws:aws-java-sdk-core:1.11.870
|         |    |    |    +--- commons-logging:commons-logging:1.1.3 -> 1.2
|         |    |    |    +--- org.apache.httpcomponents:httpclient:4.5.9
|         |    |    |    |    +--- org.apache.httpcomponents:httpcore:4.4.11
|         |    |    |    |    +--- commons-logging:commons-logging:1.2
|         |    |    |    |    \--- commons-codec:commons-codec:1.11
|         |    |    |    +--- software.amazon.ion:ion-java:1.0.2
|         |    |    |    +--- com.fasterxml.jackson.core:jackson-databind:2.6.7.3
|         |    |    |    |    +--- com.fasterxml.jackson.core:jackson-annotations:2.6.0
|         |    |    |    |    \--- com.fasterxml.jackson.core:jackson-core:2.6.7
|         |    |    |    +--- com.fasterxml.jackson.dataformat:jackson-dataformat-cbor:2.6.7
|         |    |    |    |    \--- com.fasterxml.jackson.core:jackson-core:2.6.7
|         |    |    |    \--- joda-time:joda-time:2.8.1
|         |    |    \--- com.amazonaws:jmespath-java:1.11.870
|         |    |         \--- com.fasterxml.jackson.core:jackson-databind:2.6.7.3 (*)
|         |    +--- com.amazonaws:aws-java-sdk-core:1.11.870 (*)
|         |    \--- com.amazonaws:jmespath-java:1.11.870 (*)
|         +--- com.amazonaws:aws-java-sdk-core:1.11.870 (*)
|         \--- com.amazonaws:jmespath-java:1.11.870 (*)
\--- netflix:bar-client:latest.release -> 1.0.0
     \--- com.amazonaws:aws-java-sdk-sqs:1.11.847
          +--- com.amazonaws:aws-java-sdk-core:1.11.847 -> 1.11.870 (*)
          \--- com.amazonaws:jmespath-java:1.11.847 -> 1.11.870 (*)
```

Notice the changes in `com.amazonaws:aws-java-sdk-core:1.11.847 -> 1.11.870` because of Gradle's nature

### resolution-rules-usage

This part of the demo contains examples on how we can affect dependency resolution in our project.

Important changes to our `build.gradle`:

Introduction of resolution-rules plugin:

```
    id "nebula.resolution-rules" version "7.8.0"
```

Apply resolution-rules:

```
dependencies {
    resolutionRules 'com.netflix.nebula:gradle-resolution-rules:latest.release'
    resolutionRules files('local-rules.json')
    ...
}
```

If you run `./gradlew dependencyInsight --dependency jackson-core`, you should see the release candidate being substituted:

```
Selection reasons:
      - Selected by rule : substituted com.fasterxml.jackson.core:jackson-core:2.11.0.rc+ with com.fasterxml.jackson.core:jackson-core:2.11.0 because 'pre-release version should not be chosen by default' by rule local-rules
com.fasterxml.jackson.core:jackson-core:2.11.0.rc1 -> 2.11.0
```

If you run `./gradlew dependencyInsight --dependency guava`, you should see that gradle can't resolve this dependency since we are rejecting it:

```
 Task :dependencyInsight
Resolution rules could not resolve all dependencies to align configuration ':compileClasspath':
 - com.google.guava:guava:19.0 -> com.google.guava:guava:19.0 - Could not find com.google.guava:guava:19.0.
Searched in the following locations:
  - file:/Users/rperezalcolea/Projects/github/rpalcolea/dependency-management-at-scale-jconf-mexico/repo/com/google/guava/guava/19.0/guava-19.0.pom
If the artifact you are trying to retrieve can be found in the repository but without metadata in 'Maven POM' format, you need to adjust the 'metadataSources { ... }' of the repository declaration.
com.google.guava:guava:19.0 FAILED
   Failures:
      - Could not find com.google.guava:guava:19.0.
```

If you run `./gradlew dependencyInsight --dependency aws-java-sdk-sqs`, you should see it is aligned now:

```
  Selection reasons:
      - Was requested : didn't match versions 1.11.875, 1.11.874, 1.11.873, 1.11.872, 1.11.871
      - Selected by rule : aligned to 1.11.870 by rule align-aws-java-sdk aligning group 'com.amazonaws'

com.amazonaws:aws-java-sdk-sqs:1.11.847 -> 1.11.870
\--- netflix:bar-client:1.0.0
     \--- compileClasspath (requested netflix:bar-client:latest.release)
```

### dependency-locks

Important changes to our `build.gradle`:

Introduction of gradle dependency locking plugin:

```
dependencyLocking {
    lockAllConfigurations()
}
```

If you run `./gradlew dependencies --writelocks` you should see files being generated under `gradle/dependency-locks.

These `.lockfile` files contain the version that was used for the build and will continue using them by enforcing.

Let's take a look by doing a dependency insight `./gradlew dependencyInsight --dependency aws-java-sdk-sqs`

```
   Selection reasons:
      - By constraint : dependency was locked to version '1.11.870'
      - By ancestor

com.amazonaws:aws-java-sdk-sqs:{strictly 1.11.870} -> 1.11.870
\--- compileClasspath
```

### undeclared-dependencies

Important changes to our `build.gradle`:

Introduction of gradle lint plugin:

```
    id "nebula.lint" version "16.9.1"
```

Also, we add rules dependency to our dependencies:

```
dependencies {
    //Rules can be found in https://github.com/nebula-plugins/gradle-lint-plugin/tree/master/src/main/resources/META-INF/lint-rules
    gradleLint.rules = ['undeclared-dependency']
}
```

if you run `./gradlew lintGradle` you should see the following warning

```
> Task :lintGradle FAILED

This project contains lint violations. A complete listing of the violations follows.
Because none were serious, the build's overall status was unaffected.

warning   undeclared-dependency              one or more classes in com.fasterxml.jackson.core:jackson-databind:2.6.7.3 are required by your code directly

✖ 1 problem (0 errors, 1 warning)

To apply fixes automatically, run fixGradleLint, review, and commit the changes.
```

by running `./gradlew fixGradleLint`, this should be fixed. After this, you should see the jackson dependency in your `build.gradle`:

```
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.6.7.3'
```

### unused-dependencies

Important changes to our `build.gradle`:

Introduction of gradle lint plugin:

```
    id "nebula.lint" version "16.9.1"
```

Also, we add rules dependency to our dependencies:

```
dependencies {
    //Rules can be found in https://github.com/nebula-plugins/gradle-lint-plugin/tree/master/src/main/resources/META-INF/lint-rules
    gradleLint.rules = ['unused-dependency']
}
```

if you run `./gradlew lintGradle` you should see the following warning

```
> Task :lintGradle FAILED

This project contains lint violations. A complete listing of the violations follows.
Because none were serious, the build's overall status was unaffected.

warning   unused-dependency                  this dependency should be removed since its artifact is empty (no auto-fix available)
build.gradle:26
implementation 'netflix:foo-client:latest.release'

warning   unused-dependency                  this dependency should be removed since its artifact is empty (no auto-fix available)
build.gradle:27
implementation 'netflix:bar-client:latest.release'

✖ 2 problems (0 errors, 2 warnings)
```

This particular rule does not have an auto fix, so we expect users to react based on the project context
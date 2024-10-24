# Deploying a Native Spring Boot 3 REST API Application to AWS Lambda

AWS Lambda is a possible solution to host a Spring Boot application.
If you take it further and complile the application ahead of time into a native binary with [GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/), it can dramatically reduce the cold start times of the application, which is crucial for serverless deployments that may scale up and down frequently.

This guide is an effort to extend the exisiting [Quick Start Spring Boot 3](https://github.com/aws/serverless-java-container/wiki/Quick-start---Spring-Boot3) documentation with steps necessary to compile a Spring Boot 3 Serverless application ahead of time with GraalVM Native Image and package it for deployments to AWS Lambda.
AWS Lambda offers a [custom runtime option](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-provided.html) which is exactly what is needed to deploy a native executable file that does not require a JRE to run.

### Prerequisites

Install a GraalVM JDK. 
The easiest way to get started is with [SDKMAN!](https://sdkman.io/jdks/#graal):
```bash
sdk install java 23.0.1-graal
```
For other download options, see [GraalVM Downloads](https://www.graalvm.org/downloads/).

Follow the steps below to create an application from scratch.
However, you can clone this repository containing a completed example and jump to the step 5.3:
```bash
git clone https://github.com/olyagpl/spring-boot3-aws-lambda-function.git
cd spring-boot3-aws-lambda-function
```

## Step 1: Create a Spring Boot 3 Serverless Application

The guide starts with creating a Spring Boot 3 serverless application from the Maven archetype `aws-serverless-springboot3-archetype`.
It simplifies the project setup:
```bash
mvn archetype:generate -DgroupId=spring.lambda -DartifactId=spring-lambda -Dversion=1.0-SNAPSHOT \
  -DarchetypeGroupId=com.amazonaws.serverless.archetypes \
  -DarchetypeArtifactId=aws-serverless-springboot3-archetype \
  -DarchetypeVersion=2.0.1
```

This archetype provides a project skeleton with both configuration files for Gradle (_build.gradle_) and Maven (_pom.xml_).
This guide will focus on Maven.

The key class generated is `StreamLambdaHandler`, which will handle Lambda invocations.

## Step 2: Compile and Package the Application

To ensure your setup is correct, compile and package your application on the Java VM:
```bash
mvn clean package
```

The Maven Assembly Plugin, `maven-assembly-plugin`, packages the project into a ZIP file with all runtime dependencies, necessary for deployment to AWS Lambda.
The _spring-function-1.0-SNAPSHOT-lambda-package.zip_ file is created in the _target/_ directory.

## Step 3: Prepare for a Native Image Build

To compile your Spring Boot application into a native image, you need to update _pom.xml_ by adding the [Native Build Tools](https://graalvm.github.io/native-build-tools/latest/index.html) and Spring Boot Maven plugins.
In the <profiles> section of _pom.xml_, define a new `native` profile that will handle the ahead-of-time compilation of your application:
```xml
<profile>
    <id>native</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <jvmArguments>-agentlib:native-image-agent=config-merge-dir=src/main/resources/META-INF/native-image/</jvmArguments>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <configuration>
                    <imageName>native-function</imageName>
                    <buildArgs>
                        <buildArg>--enable-url-protocols=http</buildArg>
                    </buildArgs>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                        </goals>
                        <phase>package</phase>
                    </execution>
                    <execution>
                        <id>test</id>
                        <goals>
                            <goal>test</goal>
                        </goals>
                        <phase>test</phase>
                    </execution>
                </executions>
            </plugin>
        <plugins>
    <build>
</profile>
```
This profile speficies how the native image should be built and what the output file name should be.
If you would like to pass additional options to `native-image`, add those in the plugin configuration:
```xml
<configuration>
	<buildArgs>
    		<buildArg>--option</buildArg>
	</buildArgs>
</configuration>
```
Learn more in the [plugin documentation](https://graalvm.github.io/native-build-tools/latest/maven-plugin.html).

## Step 4: Compile the Application with Native Image (Optional)

To compile your Spring Boot application into a native image, use the following command:
```bash
mvn -Pnative native:compile
```
This process should take less approximately 2 minutes.
Once complete, run the generated native executable to ensure it works:
```bash
./target/native-function
```

You should see a Spring Boot startup banner, verifying that your native image was successfully built.

> If your application behaves differently than on a Java VM, refer to the [Troubleshoot Native Image Run-Time Errors](https://www.graalvm.org/latest/reference-manual/native-image/guides/troubleshoot-run-time-errors/) guide.

## Step 5: Package the Native Image for AWS Lambda Deployment

For deployment to AWS Lambda, you need to package the native image together with a bootstrap file into an archive.

### 5.1. Add a Package Stage to the Native Profile 

Add the Maven Assembly plugin to the `native` profile in _pom.xml_ that will package the native image into a ZIP file format required for deployment to AWS Lambda:
```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <executions>
        <execution>
            <id>native-zip</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <inherited>false</inherited>
        </execution>
    </executions>
    <configuration>
        <descriptors>
            <descriptor>src/assembly/native.xml</descriptor>
        </descriptors>
    </configuration>
</plugin>
```

### 5.2. Create a Descriptor File for the Maven Assembly Plugin

Create a descriptor file named _native.xml_ in _src/assembly_ for the Maven Assembly plugin, specifying how the artifacts should be bundled.
It defines the packaging format and the entry point for an AWS Lambda invocation.
```xml
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2 https://maven.apache.org/xsd/assembly-1.1.2.xsd">
    <id>native</id>
    <formats>
        <format>zip</format>
    </formats>
    <baseDirectory>false</baseDirectory>
    <fileSets>
        <fileSet>
            <directory>src/shell/native</directory>
            <outputDirectory>/</outputDirectory>
            <useDefaultExcludes>true</useDefaultExcludes>
            <fileMode>0775</fileMode>
            <includes>
                <include>bootstrap</include>
            </includes>
        </fileSet>
        <fileSet>
            <directory>target</directory>
            <outputDirectory>/</outputDirectory>
            <useDefaultExcludes>true</useDefaultExcludes>
            <fileMode>0775</fileMode>
            <includes>
                <include>native-function</include>
            </includes>
        </fileSet>
    </fileSets>
</assembly>
```

### 5.3. Create a Bootstrap File

1. Under _src/_, create a new directory _shell/native_. Then create a new file in it named _bootstrap_.

2. Copy the following contents to the file:
    ```shell
    #!/bin/sh
    cd ${LAMBDA_TASK_ROOT:-.}
    ./target/native-function -Dlogging.level.org.springframework=DEBUG -Dlogging.level.com.amazonaws.serverless.proxy.spring=DEBUG
    ```

3. Update file permissions to make it executable:
    ```bash
    chmod +x src/shell/native/bootstrap 
    ```

### 5.4. Create a ZIP File

Now re-compile your Spring Boot application by packaging the native image into a ZIP file:
```bash
mvn -Pnative native:compile
```
Once packaged, the archive _spring-function-1.0-SNAPSHOT-native.zip_ can be uploaded to an AWS Lambda custom runtime for execution.

### Summary

By following these steps, you have successfully created a Spring Boot 3 serverless application and complied it into a native image, ready to be deployed to a [custom runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-provided.html) in AWS Lambda.
With the benefits of GraalVM Native Image, your application can consume fewer resources, making it ideal for cloud-native serverless workloads.
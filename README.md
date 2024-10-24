# Compiling a Spring Boot 3 REST API Application with Native Image and Deploying It to AWS Lambda

This guide is an effort to extend the exisiting [Quick Start Spring Boot 3](https://github.com/aws/serverless-java-container/wiki/Quick-start---Spring-Boot3) documentation with steps necessary to compile a Spring Boot 3 Serverless application ahead of time with GraalVM Native Image and package it for deployments to AWS Lambda.

AWS Lambda is a possible solution to host your Spring Boot project.
If you take is further and complile your project ahead of time into a native binary with [GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/), it can dramatically reduce the cold start times of the application, which is crucial for serverless deployments that may scale up and down frequently.

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

The distinguishing project dependencies based on the archetype are:
```xml
<dependency>
    <groupId>com.amazonaws.serverless</groupId>
    <artifactId>aws-serverless-java-container-springboot3</artifactId>
    <version>2.0.1</version>
</dependency>
<dependency>
    <groupId>com.amazonaws.serverless</groupId>
    <artifactId>aws-serverless-java-container-core</artifactId>
    <version>2.0.1</version>
    <classifier>tests</classifier>
    <type>test-jar</type>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.2.1</version>
    <scope>test</scope>
</dependency>
```

## Step 2: Compile and Package the Application

To ensure your setup is correct, compile and package your application on the Java VM:
```bash
mvn clean package
```

The Maven Shade plugin, `maven-shade-plugin`, creates an uber JAR including all necessary dependencies inside the JAR.
This is useful for creating standalone applications.
It also excludes `org.apache.tomcat.embed`, because it is not needed in a serverless environment (AWS Lambda uses a different mechanism to run web applications).

The Maven Assembly Plugin, `maven-assembly-plugin`, packages the project into a ZIP file with all runtime dependencies, necessary for deployment to AWS Lambda.
The _spring-lambda-1.0-SNAPSHOT-lambda-package.zip_ file is created in the _target/_ directory.

## Step 3: Prepare for a Native Image Build

### 3.1. Add Native Build Tools and Spring Boot Maven Plugins

To compile your Spring Boot application into a native image, you will need to update your _pom.xml_ by adding the Native Build Tools and Spring Boot Maven plugins. 
Insert the following `<build>` section at the end of the file before the closing `</project>` tag:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
            <version>0.10.3</version>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```
These plugins will handle the ahead-of-time compilation with GraalVM Native Image.

### 3.2. Configure the Native Image Build Profile

In the <profiles> section of your _pom.xml_, define a new profile that will be used to build a native image of your application.
Add the following profile:
```xml
<profile>
    <id>native</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <configuration>
                    <mainClass>spring.lambda.Application</mainClass>
                    <imageName>native-function-2</imageName>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                        </goals>
                        <phase>package</phase>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</profile>
```
This profile speficies how the native image is built and what the output file should be named.
If you would like to pass additional options to `native-image`, add those in the plugin configuration within the profile:
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

You should see the familiar Spring Boot startup banner, verifying that your native image was successfully built.

## Step 5: Package the Application for AWS Lambda

For deployment to AWS Lambda, you need to package the native image and its dependencies into a ZIP file.

### 5.1. Add a Package Stage to the Native Profile 

Add the Maven Assembly plugin to the `native` profile in _pom.xml_ that will package the native image into a ZIP file format required for deployment to AWS Lambda:
```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <executions>
        <execution>
            <id>native-archive</id>
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

Create a descriptor file named _native.xml_ in _src/assembly_ for the Maven Assembly plugin, specifying how the artifacts should be bundled. It defines the packaging format and the entry point for an AWS Lambda invocation.
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

### 5.3. Create a Native Image Archive

Now re-compile your Spring Boot application by packaging the native image into a ZIP file:
```bash
mvn -Pnative native:compile
```
Once packaged, the archive _spring-lambda-1.0-SNAPSHOT-native.zip_ can be uploaded to AWS Lambda for execution.

## Step 6: Deploy to AWS Lambda from the AWS Management Console

You can now proceed to deploy it to AWS Lambda. 

1. Navigate to the [AWS Management Console](https://aws.amazon.com/console/), select **AWS Lambda**, and create a new Lambda function.
In the function creation wizard, make sure to select **Author from scratch** and **Java 21** as the runtime. Click create.

2. Upload the ZIP file containing your application. Click **Upload from** and select the ZIP file from your local machine. 

3. Edit the **Runtime settings** and set the handler for the Lambda function, defining the entry point for the application and how requests should be handled. In this case, enter `spring.lambda.StreamLambdaHandler` for the handler class, and `handleRequest` for the method name. Click **Save** to apply the configuration.

4. Test your Lambda function from the console. Click **Test**, and create a new test event using the **API Gateway AWS Proxy** template. Enter `/ping` for the path to the resource you need to test and click **Test**. You should see a successful response from your Lambda function.

Now your Lambda function is deployed and running.
Lastly, you can create an API Gateway to forward incoming requests to the Lambda function.

## Step 7: Create an API Gateway

The API Gateway acts as a proxy, forwarding incoming requests to the Lambda function.

1. In the AWS Management Console, select **API Gateway**. Click **Create API** and choose **REST API**.

2. Enter a name, and click **Create API**. This will create a new REST API in API Gateway.

3. Configure the integration between the API Gateway and your Lambda function. Select a resource and method, and click **Integration Request**. Choose the Lambda function as the integration type and select the necessary Lambda function from the list. **Save** to apply the changes.

4. Deploy the API by creating a new stage. This will provide you with a unique **Invoke URL**. You can now access your REST API using the provided Invoke URL.

### Summary

By following these steps, you have successfully created a Spring Boot 3 serverless application, complied it into a native image, and deployed to AWS Lambda. With the benefits of GraalVM Native Image, your application can consume fewer resources, making it ideal for cloud-native serverless workloads.
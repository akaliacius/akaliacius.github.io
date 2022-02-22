---
layout: post
title:  "Deploy like a rockstar with AWS Cloud Development Kit"
description: Find out what are similarities and differences between these concepts
date:   2022-02-22 10:14:28 +0000
categories: jekyll update
permalink: /:year/:title.jGeek/
---
[click here](##something)

[Construct Hub](https://constructs.dev/)

[CDK Patterns](https://cdkpatterns.com/)

[API Reference for each language](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-construct-library.html)

## Introduction
Thanks to cloud technologies it becomes much easier for software developers to deploy their applications
to the cloud. However, I personally have never found it very attractive, writing those long JSON or YAML files
with definitions of resources. 

If you one of such people like me, who likes your programming language and write the code. What if I say now you can do 
that even defining cloud infrastructure. Welcome to AWS Cloud Development Kit (CDK)

It lets you use your programming language to build reliable, scalable and cost-effective applications in the cloud.
* More infrastructure with less code
* Application code, configuration and infrastructure all in one place
* Use the best software practices such as code reviews and unit testing for your infrastructure
* Share infrastructure design patterns within organisation or even with the public

AWS CDK supports
* TypeScript
* JavaScript
* Python
* Java
* C#
* .Net
* Go (in developer preview)

```java
public class MyEcsConstructStack extends Stack {

    public MyEcsConstructStack(final Construct scope, final String id) {
        this(scope, id, null);
    }

    public MyEcsConstructStack(final Construct scope, final String id,
            StackProps props) {
        super(scope, id, props);

        Vpc vpc = Vpc.Builder.create(this, "MyVpc").maxAzs(3).build();

        Cluster cluster = Cluster.Builder.create(this, "MyCluster")
                .vpc(vpc).build();

        ApplicationLoadBalancedFargateService.Builder.create(this, "MyFargateService")
                .cluster(cluster)
                .cpu(512)
                .desiredCount(6)
                .taskImageOptions(
                       ApplicationLoadBalancedTaskImageOptions.builder()
                               .image(ContainerImage
                                       .fromRegistry("amazon/amazon-ecs-sample"))
                               .build()).memoryLimitMiB(2048)
                .publicLoadBalancer(true).build();
    }
}
```

This little block of java code will generate AWS CloudFormation template of more than 500 lines. It's up to you, of course, 
whether  you prefer writing 500 lines of YAML just to create Amazon ECS service with AWS Fargate launch type or 
use programming language that you use on your day-to-day work? And what about all benefits that your IDE provides, code
completion etc.? Do you still want YAML?
* Java Development Kit (JDK) 8 or later
Import
#### Maven
```xml
<dependency>
    <groupId>software.amazon.awscdk</groupId>
    <artifactId>core</artifactId>
    <version>1.145.0</version>
</dependency>

```
#### Gradle kts
```java
implementation("software.amazon.awscdk:core:1.145.0")
```




## AWS CDK tools
Command line interface to interact with your AWS CDK app.
`brew install aws-cdk`




you might want to 
know more about AWS CDK. It might change your life for good. Now

app
stack
construct
nestedstack to overcome CloudFormation 500-resource limit

By default it uses AWS CloudFormation
there is cdktf for terraform
and cdk8s for kuberneties


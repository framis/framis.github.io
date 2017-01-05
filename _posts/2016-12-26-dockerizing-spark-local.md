---
layout: post
title:  "Getting started with Spark and Docker"
date:   2016-12-26 08:00:00 -0700
categories: spark
---

Spark is a very nice framework processing a large amount of data. In this article, we will see the caveats to have spark running locally with Docker.

<!--more-->

The Dockerfile
--------------

Unfortunatley for us, Spark's most popular Spark docker image [sequenceiq/spark](https://hub.docker.com/r/sequenceiq/spark) is outdated and uses Spark 1.6 only. Current version of Spark while writting this article is 2.0.2.

A deeper search in dockerhub does not give more relevant results from popular images. Hopefully, Spark is not too hard to install, here are the few dependencies if you want to use spark-scala:
- Java
- Scala & sbt
- Spark & Hadoop

This Docker image has all we need: https://github.com/P7h/docker-spark

Personnally, I do not like depending on someone else's Dockerfile, especially when it is unofficial and so simple. Below, the adapted version I use. 

```bash
# Dockerfile
FROM openjdk:8

# Adapted from https://github.com/P7h/docker-spark
MAINTAINER Francois Misslin <francois.misslin@gmail.com>

ARG SCALA_VERSION=2.11.8
ARG SCALA_BINARY_ARCHIVE_NAME=scala-${SCALA_VERSION}
ARG SCALA_BINARY_DOWNLOAD_URL=http://downloads.lightbend.com/scala/${SCALA_VERSION}/${SCALA_BINARY_ARCHIVE_NAME}.tgz

ARG SBT_VERSION=0.13.13
ARG SBT_BINARY_ARCHIVE_NAME=sbt-$SBT_VERSION
ARG SBT_BINARY_DOWNLOAD_URL=https://dl.bintray.com/sbt/native-packages/sbt/${SBT_VERSION}/${SBT_BINARY_ARCHIVE_NAME}.tgz

ARG SPARK_VERSION=2.0.2
ARG HADOOP_VERSION=2.7
ARG SPARK_BINARY_ARCHIVE_NAME=spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}
ARG SPARK_BINARY_DOWNLOAD_URL=http://d3kbcqa49mib13.cloudfront.net/${SPARK_BINARY_ARCHIVE_NAME}.tgz

# Configure env variables for Scala, SBT and Spark and include bin folders to PATH
ENV SCALA_HOME  /usr/local/scala
ENV SBT_HOME    /usr/local/sbt-launcher-packaging-${SBT_VERSION}
ENV SPARK_HOME  /usr/local/spark
ENV PATH        $JAVA_HOME/bin:$SCALA_HOME/bin:$SBT_HOME/bin:$SPARK_HOME/bin:$SPARK_HOME/sbin:$PATH

RUN apt-get -yqq update && \
    apt-get install -yqq vim screen tmux && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /tmp/* && \
    wget -qO - ${SCALA_BINARY_DOWNLOAD_URL} | tar -xz -C /usr/local/ && \
    wget -qO - ${SBT_BINARY_DOWNLOAD_URL} | tar -xz -C /usr/local/  && \
    wget -qO - ${SPARK_BINARY_DOWNLOAD_URL} | tar -xz -C /usr/local/ && \
    cd /usr/local/ && \
    ln -s ${SCALA_BINARY_ARCHIVE_NAME} scala && \
    ln -s ${SPARK_BINARY_ARCHIVE_NAME} spark

# To install sbt
RUN sbt about

USER root
EXPOSE 4040 8080 8081
WORKDIR /app

CMD ["/bin/bash"]
```

A few remarks:
- openjdk:8 is Docker's official's Java 8 dockerfile. It is based on Debian Jessie. There is a version for Alpine as well but nothing for Ubuntu or other distributions.
- `sbt about` is not mandatory but it installs sbt which takes time and caches the results, making it much faster on subsequent runs. It might be useful if you intend to run sbt commands from within the container.

In order to run the container, you can either run docker command directly or use docker-compose. Below, an example of docker-compose:

```yml
# docker-compose.yml
version: '2'
services:
  spark:
    build: .
    ports:
      - "8081:8081"
      - "8080:8080"
      - "4040:4040"
    volumes:
      - .:/app
```

Using the current directory as the volume will make it easy rebuild the project with sbt or run a spark job from within the  container. Therefore, anyone, even without a local Java, Scala, Sbt and spark distribution can do some modifications and get started.

ANd to get started, run:

```bash
 docker-compose run spark
```

From within the container, you can now run a sample Spark Job:

```bash
spark-submit --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples*.jar 100
```

or use the shell:

```bash
spark-shell
```

or run and expose a master or slave spark server to respectively port 8080 or port 8081

```bash
start-master.sh
start-slave.sh
```

## Real project

This is pretty cool. However, if you are building a real project, and if like me you are pretty new to Scala and Spark, you might encounter a few more problems.

### Sbt

You can now add a custom project as mentionned in the [Spark Getting Started](http://spark.apache.org/docs/latest/quick-start.html#self-contained-applications)

It worked pretty smoothly for me by running sbt package from with the container. However, when using sbt from my computer, I ran into some 'unresolved dependencies' exceptions. Basically there was some dependencies problems with the ivy cache. I managed to resolve the problem by following those instructions from the [sbt dependency management flow tutorial](http://www.scala-sbt.org/release/docs/Dependency-Management-Flow.html). Because `sbt update` and `sbt clean` did not give any result, I ended up nuking the ivy cache : `rm -r ~/.ivy2/cache`

### Fat JARS

For more complex projects you might need more dependencies than just the standard library. Spark will not include transitive dependencies. Because `sbt package` will not include those dependencies into you JAR, you want to use a plugin such as [sbt-assembly](https://github.com/sbt/sbt-assembly). This [stackoverflow answer](http://stackoverflow.com/questions/28459333/how-to-build-an-uber-jar-fat-jar-using-sbt-within-intellij-idea) has been pretty helpful as well.

Basically, you need to add to project/assembly.sbt the following line

```bash
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.3")
```

And use a build.sbt like the folling:

```scala
// build.sbt
val sparkVersion = "2.0.2"

lazy val root = (project in file(".")).
  settings(
    name := "geoname",
    version := "1.0",
    scalaVersion := "2.11.8"
  )

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core" % sparkVersion % "provided",
  "org.apache.spark" %% "spark-streaming" % sparkVersion % "provided",
  "org.apache.spark" %% "spark-streaming-twitter" % "2.0.1"
)

// META-INF discarding
assemblyMergeStrategy in assembly := {
  case PathList("META-INF", xs @ _*) => MergeStrategy.discard
  case x => MergeStrategy.first
}
```

Note the `provided` that will not re-add spark to you jar.
Also note the difference between %% (dependency built with sbt) and %.

The assembly task might be pretty slow. Therefore, for development, the spark-shell might be very helpful. When using the shell, do not forget to point to dependencies in the --package option.

I tried to use a library [algolia-search-client](https://github.com/algolia/algoliasearch-client-scala/) on Spark which explicitely depends on json4s version 3.4.0. Unfortunaltely, Spark#2.0.2 depends on json4s version 3.2.11 and those binaries are incompatible as stated [here](https://github.com/json4s/json4s/issues/316). An alternative solution would have been to rebuild spark with json4s version 3.4.0 but it does not seem like a viable solution, especially for production.

You can find a full example in the following repo: [framis/geonames-spark](https://github.com/framis/geonames-spark)
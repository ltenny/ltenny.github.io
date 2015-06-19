---
layout: post
title: Building a simple producer and consumer using Scala and Akka
---

One of the great things about software engineering is the unbounded innovation, sometimes out of necessity and sometimes out of curiosity, of the open source community.  Two of the many very cool innovations in the last few years are Scala and Akka.  Scala is a nearly-functional programming language that runs on the Java platform. Akka is a rapidly maturing message-passing framework. From these two little gems, together with a whole bunch of other innovative ideas, we get Apache Spark.  Spark will be the topic of a future post.  For now, let's look at a simple example of Akka message passing using Scala.

With the velocity of (often incompatible) changes in emerging open source tools, it's often hard to find even simple examples that work with the currently available versions of the tools.  With that in mind I thought it might be helpful to put together perhaps the simplest example of message passing using Scala (version 2.11.6) and Akka (version 2.3.11).

Our goal is to create two programs using Scala and Akka: a producer of the message "Hello World" and a consumer that prints the message on the screen.  Here we'll use remoting so the programs can run on different machines.  All this means we have to get a number of things right....sometimes a difficult feat, but that's the purpose of the post.  

We need:

 - A producer program and a consumer program
 - A couple of Akka application.conf files that configure remoting
 - A maven pom.xml that brings together all of the dependencies and automates the build
 
## The Environment ##
I'm on a Mac using IntelliJ IDEA community edition 14.1.3 but just about any editor will work because we're going to depend on maven for the build process.

## A Little Theory ##
Akka is a message passing framework. In place of the traditional synchronous 'call a method and wait to get a value back' approach we're all familiar with, Akka sees the world as a bunch of actors on the stage. The actors send messages back and forth without waiting for responses. It's kind of like a room full of people carrying on a bunch of conversations all at the same time.  The beauty of this kind of program organization has been known for more than 30 years.  Message passing solves many of the vexing problems of asynchronous and distributing computing. It's also a lot of fun.

## The Producer ##
The first of the two programs we need to write is the producer.  Here's the code:

{% highlight scala %}
package com.ctd.bigdata

import akka.actor._
import akka.actor.{ActorSystem,Props}
import akka.remote.RemoteScope

object SimpleProducer {

  // args: remoteHost remoteSystem remoteActorName
  // we assume port 11729
  def main(args: Array[String]): Unit = {
    val system = ActorSystem("SimpleProducer")
    val worker = system.actorOf(Props(new DoWorkActor(args(0),args(1),args(2))))
    for(i <- 0 until 5) {
      worker ! SendHelloWorld
      Thread.sleep(150)
    }
    worker ! AllDone
  }

  sealed trait Message
  case object SendHelloWorld extends Message
  case object AllDone extends Message

  class DoWorkActor(host: String, remoteSystem: String, actorName: String) extends Actor {
    val uri =  s"akka.tcp://$remoteSystem@$host:11729/user/$actorName"
    val remoteActor = context.actorFor(uri)

    def receive = {
      case SendHelloWorld => {
        remoteActor ! "Hello world!"
      }
      case AllDone => {
        println("*" * 50)
        println("All messages sent...")
        println("Shutting down in 5 seconds...")
        Thread.sleep(5 * 1000)
        context.system.shutdown()
        println("*" * 50)
      }
    }
  }
}
{% endhighlight %}

As with Java, we have the obligatory imports to bring in stuff we're referencing. In this case, things from the Akka framework. And, just like with Java, we need an entry point for our program. In our case, main with the familiar array of string arguments from the command line.

The first thing we do is create the actor system.  This is the heart of the Akka presence in an application.  We just give it a name: "SimpleProducer" and let all of the surrounding configuration come in from other sources. We'l have a bit more on that later.

Once we have the actor system, we create the one local actor we'll need.  This local actor does all the work in the producer. We create a new instance of the DoWorkActor class grabbing three parameters from the command line: the remote host name, the remote system name, and the remote actor's name.  We didn't bother giving this local actor a name.

Next, we simply iterate from 0 to 5 sending the SendHelloWorld message to our actor.  We pause very briefly with the Thread.sleep() method just to spread out the messages a little. When we're all done sending these five messages we follow up with an AllDone message to our local actor.

The messages we're sending are simple Scala objects. Nothing fancy.

The actor does all of the heavy lifting.

{% highlight scala %}
class DoWorkActor(host: String, remoteSystem: String, actorName: String) extends Actor {
    val uri =  s"akka.tcp://$remoteSystem@$host:11729/user/$actorName"
    val remoteActor = context.actorFor(uri)

    def receive = {
      case SendHelloWorld => {
        remoteActor ! "Hello world!"
      }
      case AllDone => {
        println("*" * 50)
        println("All messages sent...")
        println("Shutting down in 5 seconds...")
        Thread.sleep(5 * 1000)
        context.system.shutdown()
        println("*" * 50)
      }
    }
  }
{% endhighlight %}

Here we format the URI for the remote endpoint.  The syntax is simple.  It starts with the protocol. We're using akka.tcp  This is followed by the full path to the remote actor.  This path includes the remote system's name, the remote host name and port, and the path to the actor.  In our case, our remote actor is at the top of the /user level.

{% highlight scala %}
akka.tcp://SimpleConsumer@127.0.0.1:11729/user/consumer
{% endhighlight %}

With the URI for our remote actor, we use the context.actorFor() method to create a reference to the remote actor we'll be sending messages to.

We handle just two messages in this worker actor.  Scala match syntax here makes selecting messages very clean.  We simply have a case statement for each message.  For the SendHelloWorld message, we send the string "Hello world!" to our remote actor.  For the AllDone message, we wait around for a little bit just make sure all the messages are delivered and then we shutdown the system....we're all done.

## The Consumer ##
The code for the consumer is simple.  We just need to create the actor system and the consumer actor.  We handle just one message which is a String.

{% highlight scala %}
package com.ctd.bigdata

import akka.actor._
import akka.actor.ActorSystem
import akka.actor.Props


object SimpleConsumer {
  def main(args: Array[String]): Unit = {
    val system = ActorSystem("SimpleConsumer")
    val consumer = system.actorOf(Props[ConsumerActor], name ="consumer")
  }

  class ConsumerActor extends Actor {
    def receive = {
      case m: String => {
        println(s"Message is: $m")
      }
    }
  }
}
{% endhighlight %}

The two things that are a little different here is that we create the actor with just a reference to the type: ConsumerActor.  In the producer, we passed in a few parameters to the constructor.  We also provided a name for our actor. This is needed by the producer to fully qualify the path to our consumer actor.

We respond to the String message by printing it out to the console.

## The Hard Part ##
The code is simple and fun. Configuring the build is really not.  Here the subtle version differences can get maddening. Fortunately, once you get these figured out, they don't change until the next version.  As you are developing any complex system, keeping up with the changes of the components you are using is just part of the process.

Here's the pom.xml for producer.  The consumer is almost identical.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.ctd.bigdata</groupId>
    <artifactId>SimpleProducer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>"SimpleProducer"</name>

    <repositories>
        <repository>
            <id>scala-tools.org</id>
            <name>Scala-tools Maven2 Repository</name>
            <url>http://scala-tools.org/repo-releases</url>
        </repository>
    </repositories>

    <pluginRepositories>
        <pluginRepository>
            <id>scala-tools.org</id>
            <name>Scala-tools Maven2 Repository</name>
            <url>http://scala-tools.org/repo-releases</url>
        </pluginRepository>
    </pluginRepositories>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    </properties>

    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <includes>
                    <include>*.conf</include>
                </includes>
                <targetPath>${project.build.directory}/classes</targetPath>
            </resource>
        </resources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>reference.conf</resource>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.scala-tools</groupId>
                <artifactId>maven-scala-plugin</artifactId>
                <version>2.15.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.3.1</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.ctd.bigdata.SimpleProducer</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-actor_2.11</artifactId>
            <version>2.3.11</version>
        </dependency>
        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-remote_2.11</artifactId>
            <version>2.3.11</version>
        </dependency>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>2.11.6</version>
            <scope>runtime</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
</project>
{% endhighlight %}

Aside from the various version numbers for the components (which is really critical for everyone to play nice) there is one important and subtle part.

We need to include an application.conf file in the classpath. This file configures just about everything in the actor system.  Actually, it together with various reference.conf files literally define everything as default values are a bit frowned upon by Akka. 

The application.conf file needs to go into the src/main/resources folder. From here it will get copied to the classpath.

Here's what our application.conf file looks like:

{% highlight scala %}
akka {
  actor {
    provider = "akka.remote.RemoteActorRefProvider"
  }
  remote {
    enabled-transports = ["akka.remote.netty.tcp"]
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
    }
  }
}
{% endhighlight %}

It's simple. We just add in the configuration we need to get remoting to work.  Unfortunately, if we put this out on our classpath, the actor system would be furiously upset about not having all of the key values it needs.  It turns out that what we really need to do is concatenate this with all of the reference.conf files associated with all the Akka libraries we're using.  This is where the maven-shade-plugin comes in handy.

{% highlight xml %}
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-shade-plugin</artifactId>
   <version>2.4</version>
   <executions>
       <execution>
           <phase>package</phase>
           <goals>
               <goal>shade</goal>
           </goals>
           <configuration>
               <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                         <resource>reference.conf</resource>
                    </transformer>
               </transformers>
           </configuration>
       </execution>
   </executions>
</plugin>
{% endhighlight %}

This plugin appends all of the reference.conf files together and everyone plays nicely with the application.conf file.

## Running The Code ##
We've created two programs: SimpleProducer which generates a few String messages and sends them to a remote actor in the second program: SimpleConsumer.

{% highlight tcsh %}
ff62ps1:SimpleConsumer ltenny$ java -jar target/Simple*.jar
[DEBUG] [06/16/2015 20:02:33.259] [main] [EventStream(akka://SimpleConsumer)] logger log1-Logging$DefaultLogger started
[DEBUG] [06/16/2015 20:02:33.260] [main] [EventStream(akka://SimpleConsumer)] Default Loggers started
[INFO] [06/16/2015 20:02:33.277] [main] [Remoting] Starting remoting
[INFO] [06/16/2015 20:02:33.389] [main] [Remoting] Remoting started; listening on addresses :[akka.tcp://SimpleConsumer@127.0.0.1:11729]
[INFO] [06/16/2015 20:02:33.390] [main] [Remoting] Remoting now listens on addresses: [akka.tcp://SimpleConsumer@127.0.0.1:11729]

{% endhighlight %}

The SimpleConsumer is patiently listening on port 11729 on the local machine 127.0.0.1  Curiously, localhost does not seem to work for a host name.  Other DNS resolving names work just fine. Simply update the application.conf file with the appropriate host name and port.

Running the SimpleProducer generates a little more output

{% highlight tcsh %}
scala> ff62ps1:SimpleProducer ltenny$ java -jar target/Simple*.jar 127.0.0.1 SimpleConsumer consumer
[INFO] [06/16/2015 20:06:58.126] [main] [Remoting] Starting remoting
[INFO] [06/16/2015 20:06:58.294] [main] [Remoting] Remoting started; listening on addresses :[akka.tcp://SimpleProducer@127.0.0.1:51236]
[INFO] [06/16/2015 20:06:58.295] [main] [Remoting] Remoting now listens on addresses: [akka.tcp://SimpleProducer@127.0.0.1:51236]
**************************************************
All messages sent...
Shutting down in 5 seconds...
**************************************************
[INFO] [06/16/2015 20:07:04.084] [SimpleProducer-akka.remote.default-remote-dispatcher-6] [akka.tcp://SimpleProducer@127.0.0.1:51236/system/remoting-terminator] Shutting down remote daemon.
[INFO] [06/16/2015 20:07:04.086] [SimpleProducer-akka.remote.default-remote-dispatcher-6] [akka.tcp://SimpleProducer@127.0.0.1:51236/system/remoting-terminator] Remote daemon shut down; proceeding with flushing remote transports.
[INFO] [06/16/2015 20:07:04.106] [ForkJoinPool-3-worker-15] [Remoting] Remoting shut down
[INFO] [06/16/2015 20:07:04.107] [SimpleProducer-akka.remote.default-remote-dispatcher-5] [akka.tcp://SimpleProducer@127.0.0.1:51236/system/remoting-terminator] Remoting shut down.
ff62ps1:SimpleProducer ltenny$ 

{% endhighlight %}

Which produces the expected results from the SimpleConsumer

{% highlight tcsh %}
ff62ps1:SimpleConsumer ltenny$ java -jar target/Simple*.jar
[DEBUG] [06/16/2015 20:02:33.259] [main] [EventStream(akka://SimpleConsumer)] logger log1-Logging$DefaultLogger started
[DEBUG] [06/16/2015 20:02:33.260] [main] [EventStream(akka://SimpleConsumer)] Default Loggers started
[INFO] [06/16/2015 20:02:33.277] [main] [Remoting] Starting remoting
[INFO] [06/16/2015 20:02:33.389] [main] [Remoting] Remoting started; listening on addresses :[akka.tcp://SimpleConsumer@127.0.0.1:11729]
[INFO] [06/16/2015 20:02:33.390] [main] [Remoting] Remoting now listens on addresses: [akka.tcp://SimpleConsumer@127.0.0.1:11729]
[DEBUG] [06/16/2015 20:06:58.469] [SimpleConsumer-akka.remote.default-remote-dispatcher-6] [Remoting] Associated [akka.tcp://SimpleConsumer@127.0.0.1:11729] <- [akka.tcp://SimpleProducer@127.0.0.1:51236]
Message is: Hello world!
Message is: Hello world!
Message is: Hello world!
Message is: Hello world!
Message is: Hello world!
[ERROR] [06/16/2015 20:07:04.103] [SimpleConsumer-akka.remote.default-remote-dispatcher-5] [akka.tcp://SimpleConsumer@127.0.0.1:11729/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2FSimpleProducer%40127.0.0.1%3A51236-0/endpointWriter] AssociationError [akka.tcp://SimpleConsumer@127.0.0.1:11729] <- [akka.tcp://SimpleProducer@127.0.0.1:51236]: Error [Shut down address: akka.tcp://SimpleProducer@127.0.0.1:51236] [
akka.remote.ShutDownAssociation: Shut down address: akka.tcp://SimpleProducer@127.0.0.1:51236
Caused by: akka.remote.transport.Transport$InvalidAssociationException: The remote system terminated the association because it is shutting down.
]
[DEBUG] [06/16/2015 20:07:04.106] [SimpleConsumer-akka.remote.default-remote-dispatcher-5] [Remoting] Remote system with address [akka.tcp://SimpleProducer@127.0.0.1:51236] has shut down. Address is now gated for 5000 ms, all messages to this address will be delivered to dead letters.
[DEBUG] [06/16/2015 20:07:04.110] [SimpleConsumer-akka.remote.default-remote-dispatcher-5] [akka.tcp://SimpleConsumer@127.0.0.1:11729/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2FSimpleProducer%40127.0.0.1%3A51236-0/endpointWriter] Disassociated [akka.tcp://SimpleConsumer@127.0.0.1:11729] <- [akka.tcp://SimpleProducer@127.0.0.1:51236]

{% endhighlight %}

Of course, all of this is not the cleanest example possible.  To keep things simple, I left out a lot of what a real program should contain.  We should send a message from the producer informing the consumer that we're all done.  Here we simply shutdown the actor system on the producer. This causes the consumer to complain with a few choice error messages.

##### The code for this post can be found <a href="https://github.com/ltenny/SimpleProducerConsumer">here</a>.#####

#####Update: I've also included the sbt files if you would rather use sbt than maven.#####





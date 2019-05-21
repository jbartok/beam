---
layout: section
title: "Hazelcast Jet Runner"
section_menu: section-menu/runners.html
permalink: /documentation/runners/jet/
redirect_from: /learn/runners/jet/
---
<!--
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Overview

The Hazelcast Jet Runner can be used to execute Beam pipelines using [Hazelcat
Jet](https://jet.hazelcast.org/). 

The Jet Runner and Jet are suitable for large scale, continuous jobs, and provide:
* Support for both batch and data stream processing
* A runtime that supports very high throughput and low event latency at the same time
* Natural back-pressure in streaming programs

It's important to note that the Jet Runner is currently in an *EXPERIMENTAL* state and can not make use of many of
the capabilities present in Jet:
* while Jet has full Fault Tolerance support, the Jet Runner does not; if a job fails it must be restarted
* while Jet's internal performance is extremely high (see [benchmarks](https://jet.hazelcast.org/performance/)), 
the Runner can't match it as of now, because Beam pipeline optimization/surgery has not been fully implemented

The [Beam Capability Matrix]({{ site.baseurl }}/documentation/runners/capability-matrix/) documents the
supported capabilities of the Nemo Runner.

# Using the Hazelcast Jet Runner

Since the Jet Runner is a newcomer in the Beam Runner family it didn't yet made it
into a released version of Beam. Because of that using it is a bit cumbersome. 
Cumbersome but possible and once a release of it will be made it will be just as
convenient to use as any other Runner.

## Running WordCount from sources ##

Execution can happen in two major ways: either on a locally started Jet cluster (with configurable 
number of members), or on a completely independen one. We'll describe how to set up both.

### Common steps ### 

Get the [Beam module](https://github.com/apache/beam.git): 
```
    $ git fetch https://github.com/apache/beam.git
```

Make sure it builds:
```
    $ cd beam
    $ gradle build -x test
```
    
Generate a JAR containing the Runner and publish it into the local Maven repository:
```
    $ cd runners/jet-experimental/
    $ gradle publishToMavenLocal
```
    
Generate a JAR containing the Examples Maven Archetype and publish it into the local Maven repository:
```
    $ cd ../../sdks/java/maven-archetypes/examples
    $ gradle publishToMavenLocal
```

Generate the Examples Maven Project:
```
    $ cd ../../../../..
    $ mvn archetype:generate \
        -DarchetypeGroupId=org.apache.beam \
        -DarchetypeArtifactId=beam-sdks-java-maven-archetypes-examples \
        -DarchetypeVersion=2.14.0-SNAPSHOT \
        -DgroupId=org.example \
        -DartifactId=word-count-beam \
        -Dversion="0.1" \
        -Dpackage=org.apache.beam.examples \
        -DinteractiveMode=false
```

### Starting on a Local Cluster ###
Issue following command and the Runner will automatically start the needed cluster.
```
    $ cd word-count-beam
    $ mvn package exec:java \
        -DskipTests \
        -Dexec.mainClass=org.apache.beam.examples.WordCount \
        -Dexec.args="\
            --runner=JetRunner \
            --jetGroupName=jet \
            --jetStartOwnCluster=true
            --jetClusterMemberCount=3
            --inputFile=pom.xml \
            --output=counts" \
        -Pjet-runner
```

### Starting on a Separate Cluster ###
Download latest stable Hazelcast Jet code from [Hazelcast Website](https://jet.hazelcast.org/download/) and 
install/unarchive it. Let's say it ends up in a folder called "hazelcast-jet-3.0". Go there and start a new 
cluster with two members (for more sophisticated cluster setups consult the 
[Hazelcast Reference Manual](https://docs.hazelcast.org/docs/jet/3.0/manual/):
```
    $ cd hazelcast-jet-3.0
    $ ./jet-start.sh &
    $ ./jet-start.sh &
```

Check the cluster is up and running:
```
    $ ./jet.sh cluster
```

You should see something like:
```
State: ACTIVE
Version: 3.0
Size: 2

ADDRESS                  UUID               
[192.168.0.117]:5701     76bea7ba-f032-4c25-ad04-bdef6782f481
[192.168.0.117]:5702     03ecfaa2-be16-41b6-b5cf-eea584d7fb86
```

Download [Jet Management Center](https://docs.hazelcast.org/docs/jet-management-center/3.0/manual/)
from the same location and use it to monitor your cluster and later executions.

Go to the Beam Examples project and issue following command to execute your Pipeline on this cluster:
```
    $ cd word-count-beam
    $ mvn package exec:java \
        -DskipTests \
        -Dexec.mainClass=org.apache.beam.examples.WordCount \
        -Dexec.args="\
            --runner=JetRunner \
            --jetGroupName=jet \
            --jetStartOwnCluster=falste
            --jetClusterAddresses=192.168.0.117:5701,192.168.0.117:5702
            --codeJarLocation=target/word-count-beam-bundled-0.1.jar
            --inputFile=LICENSE.txt \
            --output=counts" \
        -Pjet-runner
```

# OmniSci + Kafka Demo 
This demo will use Kafka to send some data to OmniSci. All running in containers with Kubernetes on Red Hat's OpenShift!

## What is Omnisci?
OmniSci provides open-source software built for hyper-speed visual analytics at scale. OmniSci enables querying and visual exploration of billions of records in milliseconds, by harnessing the parallel processing power of modern GPUs.

OmniSci Core is an open-source SQL engine that optimizes query performance on the GPU. OmniSci Core keeps data close to compute, and it compiles queries to create one custom function. These and other GPU-specific innovations make queries hundreds to thousands of times faster than on legacy CPU databases. OmniSci Core also supports a High Availability configuration, providing guaranteed reliability for enterprise customers.

OmniSci Immerse is the visualization system integrated natively with OmniSci Core, also designed for the GPU. Immerse can display billions of datapoints with the same hyper-speed, allowing data scientists and analysts to interact with their data and find answers as quickly as they can think. OmniSci Immerse was designed for server-side rendering, and OmniSci architects created a unique pipeline that combines the CUDA API, the Vega visualization grammar, and the OpenGL rendering API to deliver PNGs to the user’s browser--without moving data. The OmniSci rendering engine also includes a ‘Density Gradient’ feature producing clear highlights of concentration within data visualizations.

## What is Kafka?
Kafka aims to provide a unified, high-throughput, low-latency platform for handling real-time data feeds. Kafka can connect to external systems (for data import/export) via Kafka Connect and provides Kafka Streams, a Java stream processing library.

## What is OpenShift?
OpenShift is a leading hybrid cloud, enterprise Kubernetes application platform, trusted by 1000+ organizations. Find out more here:
[https://www.openshift.com/learn/what-is-openshift](1)

We're using OpenShift as a platform to run the UI, the database, and all the components of the messaging in containers - and to keep everything alive, map persistent storage, and load balance traffic.

### I assume you have OpenShift
If you don't, you can setup a cluster on-prem or in the cloud by [following instructions here](2). Or if you have a beefy machine, you can try using a single-node installation with [CodeReady Containers](3).

## How to run this demo (the easy way)
*COMING SOON - FOR NOW, GO THE LONG WAY*

Clone this repo with: `git clone https://github.com/dudash/openshift-omnisci-kafka.git`

Run the setup script: `./openshift-omnisci-kafka/easy_setup.sh`

## How to run this demo (the long way)

### Install Kafka using Red Hat's Strimzi
[Strimzi](4) uses k8s operators to make installing a Kafka cluster on OpenShift super easy. I'm going to explain the CLI steps, but feel free to just use the web console if that's your preference. You have to be a cluster-admin to do these steps.

* `oc new-project kafka`
* `curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.13.0/strimzi-cluster-operator-0.13.0.yaml | sed 's/namespace: .*/namespace: kafka/' | oc apply -f - -n kafka`
* `oc apply -f https://raw.githubusercontent.com/dudash/openshift-omnisci-kafka/master/omnisci-kafka-persistent.yaml -n kafka`

### Install OmniSci from their dockerhub images
Omnisci keeps the EE version of their product [on dockerhub here](5). You need to [request a trial key](7) to use it. Alternatively, you can try the [community version](6) - but I haven't so YMMV.

Create a project to isolate this:
* `oc new-project omnisci`

Load image from dockerhub (omnisci/omnisci-ee-cpu):
* `oc new-app omnisci/omnisci-ee-cpu --name omnisci`

Add storage for omnisci to use:
* `oc set volume dc/omnisci --add --name=mystor --mount-path=/omnisci-storage`

Expose a route to this service so we can get them via a web browser.
* `oc expose svc/omnisci`

### Run the Kafka Connect connector for OmniSci
Kafka Connect is a tool for streaming data between Apache Kafka and external systems. It provides a framework for moving large amounts of data into and out of your Kafka cluster while maintaining scalability and reliability. Kafka Connect is typically used to integrate Kafka with external databases and storage and messaging systems.

First, we install a custom Kafka Connect service using Strimzi:
* `oc project kafka`
* `oc apply -f https://raw.githubusercontent.com/dudash/openshift-omnisci-kafka/master/omnisci-kafka-connect.yaml`
* `CONNECT_URL = $(oc get route omnisci-kafka-connect -n kafka --template={{.spec.host}})`

Next, install the OmniSci Sink connector using the Strimzi instructions (this part is a little tricky):
* https://strimzi.io/docs/latest/#using-kafka-connect-with-plug-ins-str

([Here is a tutorial](8) on how to do this with Debezium)
(Alternatively you might be able to use [confluent instructions](9), but I haven't tried)

Finally, configure and create an instance of the OmniSci Sink connector by:
* `curl https://raw.githubusercontent.com/dudash/openshift-omnisci-kafka/master/README.md -o omnisci-sink-connector.json`
* `curl -X POST -d @omnisci-sink-connector.json http://XXXXX:8083/connectors -H "Content-Type: application/json"`

### Login to the OmniSci UI
Open the route you exposed in the prior steps in a web browser. Let's make sure we can login. (You'll need your license key now if you are using Omnisci EE). If you forgot where it is, check the webconsole for exposed routes or run `oc get routes -n omnisci`.

* Username: `admin`
* Password: `HyperInteractive`
* Database: omnisci

And paste in your license key.

Let's create a dashboard...TODO

### Let's send some test data with a tool called kafkacat
We are going to dump some Kafka messages on to the bus for OmniSci to visualize:
* `kafkacat xxx`

Now look at the OmniSci web page and... TODO

[1]: https://www.openshift.com/learn/what-is-openshift
[2]: https://try.openshift.com/
[3]: https://code-ready.github.io/crc/
[4]: https://strimzi.io/
[5]: https://hub.docker.com/r/omnisci/omnisci-ee-cpu
[6]: https://hub.docker.com/r/omnisci/core-os-cpu
[7]: https://www.omnisci.com/platform/downloads/enterprise
[8]: https://medium.com/@sincysebastian/setup-kafka-with-debezium-using-strimzi-in-kubernetes-efd494642585
[9]: https://docs.confluent.io/current/connect/kafka-connect-omnisci/index.html#

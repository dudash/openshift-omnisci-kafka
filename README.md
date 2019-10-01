[![OpenShift Version][openshift41-logo]][openshift41-url]

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
* `oc expose svc/demo-kc-connect-api --name=connect-rest-service`
* `CONNECT_URL = $(oc get route connect-rest-service -n kafka --template={{.spec.host}})`

*Note that you can further customize the Kafka Connect image to add more connectors. Use the [Strimzi instructions here](10) (doing this is a little tricky, [but this is a good tutorial](8) on how to do it). Alternatively, you might be able to use [Confluent instructions](9), but I haven't tried*

Finally, configure and create an instance of the OmniSci Sink connector by:
* `curl https://raw.githubusercontent.com/dudash/openshift-omnisci-kafka/master/omnisci-sink-connector.json -o omnisci-sink-connector.json`
* customize the `omnisci-sink-connector.json` if needed
* `curl -X POST -d @omnisci-sink-connector.json $CONNECT_URL/connectors -H "Content-Type: application/json"`

### Login to the OmniSci UI
Open the route you exposed in the prior steps in a web browser. Let's make sure we can login. (You'll need your license key now if you are using Omnisci EE). If you forgot where it is, check the webconsole for exposed routes or run `oc get routes -n omnisci`.

* Username: `admin`
* Password: `HyperInteractive`
* Database: omnisci

And paste in your license key.

### Let's send some test data with a tool called kafkacat
We are going to dump some Kafka `ships_ais` topics on to the Kafka bus for OmniSci:
* `kafkacat -b localhost:9092 -t ships_ais -P -l ais-topics-85kdump.raw`

Now look at the OmniSci web page and navigate to the Data Manager tab. You should see data coming for the `ships_ais` topic we've been dumping in to Kakfa.

### Let's visualize some data

Let's create a dashboard. First, we need to download the dashboard file by:
* `curl https://raw.githubusercontent.com/dudash/openshift-omnisci-kafka/master/ais-dashboard.json -o ais-dashboard.json`

On the Dashboards tab, click the Import button (top right - looks like a down arrow). And select the `ais-dashboard.json` file. We should now be able to see our AIS transmissions on a nice visual dashboard.

---

[1]: https://www.openshift.com/learn/what-is-openshift
[2]: https://try.openshift.com/
[3]: https://code-ready.github.io/crc/
[4]: https://strimzi.io/
[5]: https://hub.docker.com/r/omnisci/omnisci-ee-cpu
[6]: https://hub.docker.com/r/omnisci/core-os-cpu
[7]: https://www.omnisci.com/platform/downloads/enterprise
[8]: https://medium.com/@sincysebastian/setup-kafka-with-debezium-using-strimzi-in-kubernetes-efd494642585
[9]: https://docs.confluent.io/current/connect/kafka-connect-omnisci/index.html#
[10]: https://strimzi.io/docs/latest/#using-kafka-connect-with-plug-ins-str

---

[openshift311-url]: https://docs.openshift.com/container-platform/3.11/welcome/index.html
[openshift41-url]: https://docs.openshift.com/container-platform/4.1/welcome/index.html

[openshift311-logo]: https://img.shields.io/badge/openshift-3.11-820000.svg?labelColor=grey&logo=data:image/png;base64,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

[openshift41-logo]: https://img.shields.io/badge/openshift-4.1-820000.svg?labelColor=grey&logo=data:image/png;base64,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
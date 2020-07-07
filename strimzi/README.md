# Binding Event-driven Applications to an In-cluster Operator Managed Kafka Cluster

## Status

This sample is currently **EXPERIMENTAL**, which means that not all pieces are available in the community today.


## Introduction

This scenario illustrates binding two event-driven applications to an in-cluster Operator Managed Kafka cluster using the Service Binding Specification. 

## Actions to Perform by Users in 2 Roles

In this example there are 2 roles:

* Cluster Admin - Installs the operators to the cluster
* Application Developer - Creates a Kafka cluster, creates required Kafka resources, imports applications and creates a request to bind the applications to the Kafka cluster and the Kafka resources.

### Cluster Admin

The cluster admin needs to install 3 operators into the cluster:

* Service Binding Operator
* Strimzi Operator
* Runtime Component Operator

#### Install the Service Binding Operator

Navigate to the **Operator** → **OperatorHub** in the OpenShift console. From the `Developer Tools` category, select the `Service Binding Operator` operator and install a `alpha` version. This would install the `ServiceBindingRequest` *custom resource definition* (CRD) in the cluster.

#### Install the Strimzi Operator

Create the following `OperatorSource`:

```console
cat <<EOS |kubectl apply -f -
---
apiVersion: operators.coreos.com/v1
kind: OperatorSource
metadata:
  name: strimzi-operators
  namespace: openshift-marketplace
spec:
  type: appregistry
  endpoint: https://quay.io/cnr
  registryNamespace: navidsh
  displayName: "Bindable Strimzi Operators"
EOS
```

Once the `OperatorSource` is created, the "bindable" Strimzi Operator will be available to install from OperatorHub catalog in OpenShift console.

Then, navigate to the **Operator** → **OperatorHub** in the OpenShift console. Select **Strimzi** operator labelled as "custom" (not "Community") and install on the cluster.

#### Install the Runtime Component Operator

Navigate to the **Operator** → **OperatorHub** in the OpenShift console. Select the `Runtime Component Operator` operator and install a `beta` version. This would install the `RuntimeComponent` *custom resource definition* (CRD) in the cluster.

### Application Developer

#### Create a namespace called `service-binding-demo`

The application and the service instance needs a namespace to live in so let's create one for them:

```console
cat <<EOS |kubectl apply -f -
---
kind: Namespace
apiVersion: v1
metadata:
  name: service-binding-demo
EOS
```

#### Create a Kafka cluster

Now we use the Strimzi Operator that the cluster admin has installed. To create a Kafka cluster instance, just create a [`Kafka` custom resource](./01-kafka.yaml) in the `service-binding-demo` namespace called `my-cluster`:

```console
oc apply -f 01-kafka.yaml
```

It takes usually a few seconds to spin-up a new Kafka cluster. You can check the status in the custom resources:

```console
oc get kafka my-cluster -o yaml
```

This creates a Kafka cluster with a `tls` listener with mutual TLS authentication.

#### Create a Kafka Topic

To create a Kafka Topic, apply the [`KafkaTopic` custom resource](./02-kafka-topic.yaml) in the `service-binding-demo` namespace:

```console
oc apply -f 02-kafka-topic.yaml
```

Check the created custom resource:

```console
oc get kt my-topic
```

#### Create Kafka Users

To create Kafka Users, one user for producer application and one user for consumer application, apply the [`KafkaUser` custom resources](./03-kafka-user.yaml) in the `service-binding-demo` namespace:

```console
oc apply -f 03-kafka-user.yaml
```

Check the created custom resource:

```console
oc get kafkausers
```

#### Create a Service Binding

In order to bind the application to the Kafka resources, create [`ServiceBinding` custom resources](./04-service-binding.yaml) in the `service-binding-demo` namespace:

```console
oc apply -f 04-service-binding.yaml
```

#### Deploy the Producer and the Consumer Applications

To deploy the producer and the consumer applications, create [`RuntimeComponent` resources](./05-runtime-component.yaml) in the `service-binding-demo` namespace:

```console
oc apply -f 05-runtime-component.yaml
```

The Runtime Component Operator will create `Deployment` resources and defines binding information as environment variables for the application containers with values from the binding secret created by the Service Binding Operator.

#### View Messages

To see messages sent by the producer application, run:

```console
oc logs -n service-binding-demo -f $(oc get pods -l app=kafka-producer -o name)
```

You can press <kbd>Ctrl</kbd>+<kbd>C</kbd> to stop viewing the log messages.

Similarly, use the following command to see messages received by the consumer application:

```console
oc logs -n service-binding-demo -f $(oc get pods -l app=kafka-consumer -o name)
```


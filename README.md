# Fortanix / Kroxylicious Demo

The Record Encryption filter provides an Encryption-at-Rest solution for Apache Kafka(tm) which is transparent to both clients and brokers.

The filter is responsible for encrypting messages that are sent by producing applications so the Kafka Broker never sees the plain text content of the messages. The filter also decrypts messages before they are returned to consuming applications.

In this directory, you'll find examples that help you deploy Kroxylicious with the Record Encryption filter to your OpenShift Cluster so that you may try out the feature together
with your own application.

The Record Encryption filter requires a Key Management System (KMS). In this demo, we choose to use the Fortanix DSM(tm) integration.

Follow the [KMS preparation instructions](./PREPARE_KMS.md) then proceed to deploy one of the examples.  Each example configures ingress to the proxy to suit a
different use-case.

* [Cluster IP](./cluster-ip) (for on-cluster).
* [External Load Balancer](./load-balancer) (for off-cluster using a Load Balancer Service).
* [OpenShift Route](./openshift-route) (for off-cluster using OpenShift Routes).


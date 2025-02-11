# Kroxylicious Record Encryption, exposed using Cluster-IP

In this example, the proxy is exposed using a Cluster-IP.  This is suitable if the applications are
deployed on-cluster.

# Prerequisites

* [KMS is prepared](../PREPARE_KMS.md).
* Administrative access to an OpenShift Cluster with Streams for Apache Kafka Operator  installed through Operator Hub.

Tools:

* OpenShift CLI (`oc`)
* Fortanix DSM CLI.

# Deploying the Example


1. Deploy the Example:
   ```sh
   oc apply -k cluster-ip
   ```

# Try out the example

1. Create a key for topic `trades` using the instructions applicable to your KMS provider:

   ```sh
   KEY_NAME="KEK_trades"
   GROUP_ID=$(sdkms-cli  list-groups | grep topic-keks | awk '{print $1}')
   sdkms-cli create-key --obj-type AES --key-size 256 --group-id ${GROUP_ID} --name ${KEY_NAME} --key-ops ENCRYPT,DECRYPT,APPMANAGEABLE
   ```

2. Create a topic `trades` on the cluster, via the proxy:
   ```sh
   oc run -n kafka-proxy -qi create-topic --image=registry.redhat.io/amq-streams/kafka-38-rhel9:2.8.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server proxy-service:9092 --create --topic trades
   ```
3. Produce some messages to the topic:
   ```sh
   echo 'IBM:100\nAPPLE:99' | oc run -n kafka-proxy -qi proxy-producer --image=registry.redhat.io/amq-streams/kafka-38-rhel9:2.8.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server proxy-service:9092 --topic trades
   ```
4. Consume messages *direct* from the Kafka Cluster, showing that they are encrypted:
   ```sh
    oc run -n kafka cluster-consumer -qi --image=registry.redhat.io/amq-streams/kafka-38-rhel9:2.8.0 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic trades --from-beginning --timeout-ms 10000
   ```
5. Consume messages from the *proxy* showing they get decrypted automatically:
   ```sh
    oc run -n kafka-proxy proxy-consumer -qi --image=registry.redhat.io/amq-streams/kafka-38-rhel9:2.8.0 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server proxy-service:9092 --topic trades --from-beginning --timeout-ms 10000
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster like this:

```sh
oc delete -k cluster-ip
```

To remove the KMS configuration, see [the KMS cleanup instructions](../PREPARE_KMS.md#cleaning-up).


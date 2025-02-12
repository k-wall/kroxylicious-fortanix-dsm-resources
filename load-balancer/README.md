# Kroxylicious Record Encryption, exposed using External Load Balancer

In this example, the proxy is exposed off-cluster using a External Load Balancer.  This is suitable if the applications are
running off-cluster.

# Prerequisites

* [KMS is prepared](../PREPARE_KMS.md).
* Administrative access to an OpenShift Cluster with Streams for Apache Kafka Operator installed through Operator Hub.

Tools:

* OpenShift CLI (`oc`)
* Apache Kafka CLI tools (`kafka-topics.sh`, `kafka-console-producer.sh`, and `kafka-console-consumer.sh`) found in the `bin` directory of the Streams for Apache Kafka on RHEL distribution.
* Fortanix DSM CLI.
* GNU `sed` (if you are on a Mac and have gnu-sed installed from Brew, use `gsed` rather than `sed`).

# Deploying the Example

1. Deploy the Example:
   ```sh
   oc apply -k load-balancer
   ```
2. Get the external address of the proxy service:
   ```sh
   LOAD_BALANCER_ADDRESS=$(oc get service -n kafka-proxy proxy-service --template='{{(index .status.loadBalancer.ingress 0).hostname}}')
   ```
3. Now update the `brokerAddressPattern:` to match the `LOAD_BALANCER_ADDRESS`:
   ```sh
     sed -i  "s|\(brokerAddressPattern:\).*$|\1 ${LOAD_BALANCER_ADDRESS}|" load-balancer/proxy/proxy-config.yaml
   ```
4. Reapply and restart:
   ```sh
      oc apply -k load-balancer && oc delete pod -n kafka-proxy --all
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
   kafka-topics.sh --bootstrap-server ${LOAD_BALANCER_ADDRESS}:9092 --create -topic trades
   ```
   Note it may take a minute or two for `${LOAD_BALANCER_ADDRESS}` to resolve in your environment and for the load balancer to begin routing
   network traffic.
3. Produce some messages to the topic:
   ```sh
   echo 'IBM:100\nAPPLE:99' | kafka-console-producer.sh --bootstrap-server ${LOAD_BALANCER_ADDRESS}:9092 --topic trades
   ```
4. Consume messages direct from the Kafka Cluster, showing that they are encrypted:
   ```sh
    oc run -n kafka cluster-consumer -qi --image=registry.redhat.io/amq-streams/kafka-38-rhel9:2.8.0 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic trades --from-beginning --timeout-ms 10000
   ```
5. Consume messages from the proxy showing they are decrypted:
   ```sh
    kafka-console-consumer.sh --bootstrap-server ${LOAD_BALANCER_ADDRESS}:9092 --topic trades --from-beginning --timeout-ms 10000
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster like this:

```sh
oc delete -k load-balancer
```

To remove the KMS configuration, see [the KMS cleanup instructions](../PREPARE_KMS.md#cleaning-up).


# Kroxylicious Record Encryption, exposed using External Load Balancer

In this example, the proxy is exposed off-cluster using OpenShift Routes.  The proxy uses a self-signed cert.  You
can adapt the cert-manager configuration to generate a certificate signed by a CA if you wish. Do this by adjusting the
certmgr CRs within `openshift-route/proxy/`.

This is suitable if the applications are running off-cluster. 

# Prerequisites

* [KMS is prepared](../PREPARE_KMS.md).
* Administrative access to an OpenShift Cluster with Streams for Apache Kafka Operator and Cert-Manager (Red Hat or Community) installed through Operator Hub.

Tools:

* OpenShift CLI (`oc`)
* Apache Kafka CLI tools (`kafka-topics.sh`, `kafka-console-producer.sh`, and `kafka-console-consumer.sh`) found in the `bin` directory of the Streams for Apache Kafka on RHEL distribution.
* Fortanix DSM CLI.
* GNU `sed`
* `jq`

# Deploying the Example

1. Get the domain of the default OpenShift Ingress Controller assigned to your cluster.
   ```sh
   DOMAIN=$(oc get ingresscontrollers.operator.openshift.io -n openshift-ingress-operator default  --template='{{.spec.domain}}')
   BOOTSTRAP=proxy-bootstrap-kafka-proxy.${DOMAIN}:443
   ```
2. Now update the config to use the domain
   ```sh
     sed -i  "s|apps.[a-z0-9.]*.openshiftapps.com|${DOMAIN}|g" openshift-route/proxy/proxy-config.yaml openshift-route/proxy/server-certificate.yaml
   ```
3. Deploy the Example:
   ```sh
   oc apply -k openshift-route
   ```
5. Extract the client trust
   ```sh
   oc get secret -n kafka-proxy kroxy-server-key-material -o json | jq -r ".data.\"tls.crt\" | @base64d" > client.pem
   ```

# Try out the example:

1. Create a key for topic `trades` using the instructions applicable to your KMS provider:
   
   ```sh
   KEY_NAME="KEK_trades"
   GROUP_ID=$(sdkms-cli  list-groups | grep topic-keks | awk '{print $1}')
   sdkms-cli create-key --obj-type AES --key-size 256 --group-id ${GROUP_ID} --name ${KEY_NAME} --key-ops ENCRYPT,DECRYPT,APPMANAGEABLE
   ```

2. Create a topic `trades` on the cluster, via the proxy:
   ```sh
   kafka-topics.sh --bootstrap-server ${BOOTSTRAP} --create -topic trades  --command-config openshift-route/client.properties
   ```
3. Produce some messages to the topic:
   ```sh
   echo 'IBM:100\nAPPLE:99' | kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP} --topic trades --producer.config openshift-route/client.properties
   ```
4. Consume messages direct from the Kafka Cluster, showing that they are encrypted:
   ```sh
    oc run -n kafka cluster-consumer -qi --image=registry.redhat.io/amq-streams/kafka-38-rhel9:2.8.0 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic trades --from-beginning --timeout-ms 10000
   ```
5. Consume messages from the proxy showing they are decrypted:
   ```sh
    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP} --topic trades --from-beginning --timeout-ms 10000 --consumer.config openshift-route/client.properties
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster like this:

```sh
oc delete -k openshift-route
```

To remove the KMS configuration, see [the KMS cleanup instructions](../PREPARE_KMS.md#cleaning-up).


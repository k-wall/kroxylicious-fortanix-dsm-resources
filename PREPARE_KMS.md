# Preparing a Key Management Service (KMS)

In order to use the Record Encryption Filter, you must provide a [KMS solution](./README.md).   Although Record Encryption supports several KMSs, this
example assumes Fortanix DSM.  You may use either SaaS or an on-premise instance.

### Prerequisites

You must have:

* Fortanix DSM account with permissions to create apps, groups and keys
* Know the Fortanix DSM endpoint e.g. `https://api.uk.smartkey.io`


And the following tools
* Python 3 (for the Fortanix CLI)
* OpenShift Client
* GNU sed (if you are on a Mac and have gnu-sed installed from Brew, use `gsed` rather than `sed`).

### Installing the Fortanix DSM CLI

If you haven't got the Fortanix CLI installed on your system, install it locally.  You can use a Python Virtual Env if you wish.

```
python3 -m venv ./venv
. ./venv/bin/activate
pip3 install sdkms-cli
```

### Login to Fortanix DSM

Log into Fortanix DSM:

```
export FORTANIX_API_ENDPOINT=https://api.uk.smartkey.io   # Use your Fortanix DSM endpoint.
sdkms-cli user-login --username xxxx@yyyy.zzz
```

### Create a Fortanix Group for the Topic Keys

```
sdkms-cli create-group --name topic-keks
```

Keep a note of the group id, you'll need that later.

### Create a Fortanix App for use by Record Encryption and retrieve the API key

```
sdkms-cli create-app --name kroxylicious --default-group topic-keks --groups topic-keks
sdkms-cli get-app-api-key --name kroxylicious > fortanix-dsm.apikey
```

1. Create a secret containing the Fortanix Api Key
   ```sh
   oc create secret generic proxy-encryption-kms-secret -n kafka-proxy --from-file=fortanix-dsm-apikey.txt=fortanix-dsm.apikey --dry-run=client -o yaml > base/proxy/proxy-encryption-kms-secret.yaml
   ```

2. Update the proxy config to refer to your Fortanix DSM instance:
   ```sh
      sed -i "s|\(endpointUrl:\).*$|\1 ${FORTANIX_API_ENDPOINT}|" */proxy/proxy-config.yaml
   ```  

You are now ready to continue to one of the demos:

* [Cluster IP](./cluster-ip)
* [External Load Balancer](./load-balancer)
* [OpenShift Route](./openshift-route)

# Cleaning up

Once you've finished experimenting, you can use these commands to clean up Fortanix DSM returning it to the initial state.

Delete the key.

```sh
sdkms-cli list-keys
# then delete the keys by its kid, e.g.:
sdkms-cli delete-key --kid 0e83e230-964c-4358-921a-5fa8d4b6ef88
```

Delete the group.

```sh
sdkms-cli delete-group --name topic-keks
```

Delete the app.
```sh
sdkms-cli delete-app  --name kroxylicious
```

Finally, delete the .apikey files:
```sh
rm *.apikey
```


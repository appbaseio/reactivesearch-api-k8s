# K8S setup for ARC + ElasticSearch

[Arc](https://arc-site.netlify.com) acts as an API gateway for ElasticSearch and augments the search experience by offering:

- Out-of-the-box search and click analytics,
- Fine-grained security controls but without any restrictions (Apache 2.0 licensed),
- A superior development experience: Import data via GUI, build and test search relevancy visually with no code, set query rules and advanced query suggestions.

This example demonstrates how you can deploy ElasticSearch kubernetes operator, i.e. [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html) with [Arc](https://arc-site.netlify.com/) on any kubernetes cluster.

## Quick Start

> Note: Steps described here assumes a kubernetes installation on the system. This will allow you to execute `kubectl` commands.

- **Step 1 -** Create kubernetes cluster with your favourite provider. Here are some of the examples

  - [Google Cloud](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)
  - [AWS](https://aws.amazon.com/kubernetes/)
  - [Azure](https://azure.microsoft.com/en-in/services/kubernetes-service/)
  - [Digital Ocean](https://www.digitalocean.com/products/kubernetes/)

- **Step 2 -** Connect to cluster using CLI, so that you can run `kubectl` commands

  **Example:**

  ```bash
  gcloud container clusters get-credentials cluster-name --zone us-east1-b --project test
  ```

- **Step 3 -** Get custom resources for ECK k8s operator

  ```bash
  kubectl apply -f https://download.elastic.co/downloads/eck/1.0.0-beta1/all-in-one.yaml
  ```

- **Step 4 -** Deploy ElasticSearch

  > **Note:** ElasticSearch configuration file below has 1 node cluster with 5gb storage, 4gb memory and 2 core CPU limit. You can clone and update configuration in [eck_config](https://github.com/appbaseio/arc-k8s/blob/master/eck_config.yaml) as per the requirement. For more configuration options you can check [Elastic Cloud on Kubernetes docs](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/appbaseio/arc-k8s/master/eck_config.yaml
  ```

- **Step 5 -** Monitor cluster health & creation process

  ```bash
  kubectl get elasticsearch

  ```

  > **Note:** Wait until health shows up as `green`. It can take upto 3-4 min.

- **Step 6 -** Get password for ElasticSearch and update in [arc.yaml](https://github.com/appbaseio/arc-k8s/blob/master/arc.yaml)

  ```bash
  kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
  ```

  ![arc-cluster-env](https://i.imgur.com/4OXfxQC.png)

- **Step 7 -** Obtain Arc ID by following the steps mentioned [here](https://docs.appbase.io/docs/hosting/BYOC/#how-to-create-arc-instance)

- **Step 8 -** Update env variables, set ARC ID: obtained in step 7 and update desired username and password. This credentials will act as master credentials for Arc ARC.
  ![arc-env](https://i.imgur.com/cC7sxUP.png)

- **Step 9 -** Deploy arc

  ```bash

  kubectl apply -f arc.yaml
  ```

- **Step 10 -** Update tls certificate
  Nginx is used to proxy Arc, so that we can enable TLS for ARC URL. This example comes with Self Signed certificate, but you can update the values with actual certificate, by encoding the content of `.crt` and `.key` files into base64 and replacing the values in `nginx.yaml`.

  ![nginx-cert](https://i.imgur.com/6UkZwMC.png)

# K8S setup for ARC + ElasticSearch

This demonstrates how you can use ElasticSearch kubernetes operator, i.e. [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html) with [Arc](https://arc-site.netlify.com/)

> Quick start guide below enables you to quickly configure ElasticSearch and Arc on local kubernetes. For the production ready configuration of ElasticSearch please go through [ECK docs](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html) and update `eck_config.yaml` accordingly.

## Quick Start

- Get Arc ID following the steps mentioned [here](https://docs.appbase.io/docs/hosting/BYOC/#how-to-create-arc-instance)

- Start minikube with approximately 3GB of RAM (Required by ElasticSearch)

```bash
minikube start --memory 3096
```

- Get custom resources for ECK k8s operator

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/1.0.0-beta1/all-in-one.yaml
```

- Change the number of master, data nodes required by your application in `eck_config.yaml`. For more details you can check Elastic Cloud on Kubernetes [docs](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-node-configuration.html)

- Change the volume space required by your application in `eck_config.yaml` L42

- Change the memory required in `eck_config.yaml` L10

* Deploy elasticsearch cluster

```bash
kubectl apply -f eck_config.yaml
```

- Monitor cluster health & creation process

```bash
kubectl get elasticsearch

```

> Wait until health shows up as `green`. It can take upto 3-4 min.

- Get password for the deployed ES

```bash
kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

- Change password for ElasticSearch URL in `arc.yaml` L28

- Update env variables, set ARC ID: obtained in step 1

- Set username and password for providing basic auth to Arc url.

- Deploy arc

```bash

kubectl apply -f arc.yaml
```

- Nginx is used to proxy Arc, so that we can enable TLS for ARC URL. Current example comes with Self Signed certificate, but you can update the values with actual certificate, by encoding the content of `.crt` and `.key` files into base64 and replacing the values in `nginx.yaml` L7-L9.

- Create nginx ingress

```bash
kubectl apply -f nginx.yaml
```

For more information on how to configure elasticsearch + plugins, you can check Elastic Cloud on Kubernetes (ECK) [docs](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)

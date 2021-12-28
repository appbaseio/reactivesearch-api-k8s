# K8S setup for Appbase.io + ElasticSearch

[Appbase.io](https://appbase.io) acts as an API gateway for ElasticSearch and augments the search experience by offering:

- Out-of-the-box search and click analytics,
- Fine-grained security controls but without any restrictions (Apache 2.0 licensed),
- A superior development experience: Import data via GUI, build and test search relevancy visually with no code, set query rules and advanced query suggestions.

This example demonstrates how you can deploy ElasticSearch kubernetes operator, i.e. [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html) with [Appbase.io](https://appbase.io/) on any kubernetes cluster.

## Quick Start

> Note: Steps described here assumes a [kubernetes](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installation on the system. This will allow you to execute `kubectl` commands.

- **Step 1 -** Create kubernetes cluster with your favourite provider. Here are some of the well known Kubernetes Cluster providers.

  - [Google Cloud](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)
  - [AWS](https://aws.amazon.com/kubernetes/)
  - [Azure](https://azure.microsoft.com/en-in/services/kubernetes-service/)
  - [Digital Ocean](https://www.digitalocean.com/products/kubernetes/)

  _Here is an example on how you can create a cluster on Google Cloud._

  > _Note: We assume that you have `gcloud` sdk installed on your machine, if not you can install from [here](https://cloud.google.com/sdk/install). Also you have a project on [Google Cloud](https://console.cloud.google.com/) with [Kubernetes Engine](https://console.cloud.google.com/apis/library/container.googleapis.com?q=kubernetes%20engine&_ga=2.12989943.-1532196267.1571335381) enabled_

  _1. Login and Initialized Google Cloud_

  ```bash
  gcloud init
  ```

  _2. Create Cluster_

  ```bash

  gcloud container clusters create YOUR_CLUSTER_NAME --num-nodes 1 --machine-type  e2-standard-2 --zone us-east1-b --node-locations us-east1-b --cluster-version latest --network default --create-subnetwork range=/19 --enable-ip-alias --no-enable-autoupgrade --project YOUR_PROJECT
  ```

  _For more configuration options, you can go through `glcoud container cluster create` command [docs](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create). Also in the above command, you can choose a different zone, machine size, number of nodes, etc based on your requirements._

  _Once the cluster is created you will get cluster IP + status in output._

  ![](https://i.imgur.com/yq9gS3Y.png)

  > _Note: For GKE clusters you will need to have cluster admin role added. Following is the command: `kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin`_

- **Step 2 -** Get custom resources for ECK k8s operator

  ```bash
  kubectl apply -f https://download.elastic.co/downloads/eck/1.1.2/all-in-one.yaml
  ```

  ![](https://i.imgur.com/yUX4pYx.png)

- **Step 3 -** Deploy ElasticSearch

  Here is a basic Kubernetes configuration which will allow you to quickly deploy a single node ElasticSearch cluster, with 4gb RAM, 2 core CPU, and 5GB storage.

  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: elasticsearch.k8s.elastic.co/v1
  kind: Elasticsearch
  metadata:
      name: elasticsearch
  spec:
    version: 7.9.0
    http:
      tls:
        selfSignedCertificate:
          disabled: true
    nodeSets:
    - name: default
      count: 1
      config:
        node.master: true
        node.data: true
        node.ingest: true
        node.store.allow_mmap: false
        network.publish_host: localhost
      volumeClaimTemplates:
      - metadata:
          name: elasticsearch-data
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
          storageClassName: standard
      podTemplate:
        spec:
          containers:
          - name: elasticsearch
            env:
            - name: ES_JAVA_OPTS
              value: -Xms2g -Xmx2g
            resources:
              requests:
                memory: 2Gi
                cpu: 0.5
              limits:
                memory: 2.5Gi
                cpu: 2
  EOF
  ```

  > Note: For more configuration options you can check [Elastic Cloud on Kubernetes docs](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)

- **Step 4 -** Monitor cluster health & creation process

  ```bash
  kubectl get elasticsearch

  ```

  ![](https://i.imgur.com/ChTgd2v.png)

  > **Note:** Wait until health shows up as `green`. It can take upto 3-4 min.

- **Step 5 -** Create Appbase.io instance. Appbase.io instance will enable you to access Appbase.io Dashboard. While following instance creation process, you will get an `APPBASE_ID` , which will help you successfully deploy appbase.io on the cluster.
  Follow the steps listed below to successfully create an appbase.io instance.

  - Go to [Appbase.io Dashboard](https://dash.appbase.io/install)

  ![](https://i.imgur.com/qVSHx0F.png)

  - Enter your email address

  - You will receive an OTP on an entered email address. Enter OTP to verify the email address.

  - You will receive an email with APPBASE_ID which can be used with Appbase.io configuration.


- **Step 6 -** Deploy Appbase.io

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/appbaseio/arc-k8s/master/appbase.yaml
  ```

  ![](https://i.imgur.com/3K5HsWj.png)

- **Step 7 -** Get Environment Variables. By now all the resources are deployed but Appbase.io service will need the following environment variables to start successfully.

  - `APPBASE_ID`
  - `ES_CLUSTER_URL`
  - `USERNAME`
  - `PASSWORD`

  We have already obtained APPBASE_ID , in step 5. Now we need ElasticSearch cluster URL, i.e. `ES_CLUSTER_URL`. Since we are using Kubernetes orchestration, we can connect to ElasticSearch using its service name and port number. But [ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html) comes with [xpack](http://elastic.co/guide/en/elasticsearch/reference/current/security-settings.html) security which has basic auth enabled. So first let's get the user name password to successfully connect with ElasticSearch. Username here defaults to elastic , so we only need a way to get the password, which is stored in the secrets of Kubernetes.

  Here is the command to get the Password for ElasticSearch

  ```bash
  kubectl get secret elasticsearch-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
  ```

  ![](https://i.imgur.com/YL2NGHT.png)

  Once we have password our `ES_CLUSTER_URL` looks like

  ```bash
  http://elastic:jwg7k9fc9l89trb7l98jmhf2@elasticsearch-es-http:9200/
  ```

  Also, default `USERNAME` and `PASSWORD` are set to `admin:admin` you can change them before deploying for production as these credentials will act as master credentials to access Appbase Dashboard.

- **Step 8 -** Configure fluent bit. Update the `ELASTICSEARCH_PASSWORD` password obtained in step 7 and execute the following command

  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: fluent-bit
    labels:
      k8s-app: fluent-bit-logging
      version: v1
      kubernetes.io/cluster-service: "true"
  spec:
    selector:
      matchLabels:
        k8s-app: fluent-bit-logging
    template:
      metadata:
        labels:
          k8s-app: fluent-bit-logging
          version: v1
          kubernetes.io/cluster-service: "true"
        annotations:
          prometheus.io/scrape: "true"
          prometheus.io/port: "2020"
          prometheus.io/path: /api/v1/metrics/prometheus
      spec:
        containers:
          - name: fluent-bit
            image: fluent/fluent-bit:1.3.11
            imagePullPolicy: Always
            ports:
              - containerPort: 2020
            env:
              - name: ELASTICSEARCH_HOST
                value: "elasticsearch-es-http"
              - name: ELASTICSEARCH_PORT
                value: "9200"
              - name: ELASTICSEARCH_USERNAME
                value: elastic
              - name: ELASTICSEARCH_PASSWORD
                value: ELASTICSEARCH_PASSWORD
            volumeMounts:
              - name: varlog
                mountPath: /var/log
                subPath: es.json
              - name: fluent-bit-config
                mountPath: /fluent-bit/etc/
        terminationGracePeriodSeconds: 10
        volumes:
          - name: varlog
            hostPath:
              path: /var/log
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers
          - name: fluent-bit-config
            configMap:
              name: fluent-bit-config
        serviceAccountName: fluent-bit
        tolerations:
          - key: node-role.kubernetes.io/master
            operator: Exists
            effect: NoSchedule
          - operator: "Exists"
            effect: "NoExecute"
          - operator: "Exists"
            effect: "NoSchedule"
  EOF
  ```

- **Step 9 -** Configure Environment Variables and redeploy appbase. You can update values gathered in the above step in the following command and execute it

  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      deployment.kubernetes.io/revision: "1"
    generation: 1
    name: appbase
  spec:
    selector:
      matchLabels:
        app: appbase
    strategy:
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: appbase
      spec:
        containers:
          - env:
              - name: USERNAME
                value: admin
              - name: PASSWORD
                value: admin
              - name: APPBASE_ID
                value: YOUR_APPBASE_ID
              - name: ES_CLUSTER_URL
                value: "http://elastic:PASSWORD@elasticsearch-es-http:9200/"
              - name: LOG_FILE_PATH
                value: "/var/log/es.json"
            image: appbaseio/reactivesearch-api:7.54.0
            imagePullPolicy: IfNotPresent
            name: appbase
            ports:
              - containerPort: 8000
                name: http
                protocol: TCP
            volumeMounts:
              - name: varlog
                mountPath: /var/log
                subPath: es.json
        volumes:
          - name: varlog
            hostPath:
              path: /var/log
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers
    replicas: 1
  EOF
  ```

  Now, we have successfully configured appbase. Let's test it. Following command will help you get the Load Balancer IP address using which our service can be accessed.

  ```bash
  kubectl get services ingress-nginx -n ingress-nginx
  ```

  ![](https://i.imgur.com/sgI9FZk.png)

  Now that we have external IP, let us execute the following command with correct IP which will help us get the cluster health and see if it is running correctly.

  ```bash
  curl http://YOUR_EXTERNAL_IP/_cluster/health\?pretty -u admin:admin
  ```

  ![](https://i.imgur.com/SltUEFy.png)

  As you can see our ElasticSearch cluster is running with a healthy state, i.e. green , means we have successfully deployed ElasticSearch + Appbase.io services 🎉.

- **Step 10 -** Configure TLS certificate + custom domain. We highly recommend using https enabled hosts so that you can seamlessly use your cluster with Appbase.io dashboard. For this, we have configured the Nginx server with reverse proxy to Appbase.io deployment.

  You can replace domain name in the following command and execute it to make sure you can access services using your domain name instead of IP address

  ```bash
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
      kubernetes.io/tls-acme: "false"
      nginx.ingress.kubernetes.io/configuration-snippet: |
        add_header 'access-control-expose-headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,X-Search-Id,X-Search-Click,X-Search-ClickPosition,X-Search-Suggestions-Click,X-Search-Suggestions-ClickPosition,X-Search-Conversion,X-Search-Query,X-Search-Filters,X-Request-Tag,X-Query-Tag,X-Search-State,X-Search-CustomEvent,X-User-Id,X-Search-Client,X-Enable-Telemetry,X-Timestamp' always;
      nginx.ingress.kubernetes.io/cors-allow-credentials: "false"
      nginx.ingress.kubernetes.io/cors-allow-headers: DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,X-Search-Id,X-Search-Click,X-Search-ClickPosition,X-Search-Suggestions-Click,X-Search-Suggestions-ClickPosition,X-Search-Conversion,X-Search-Query,X-Search-Filters,X-Request-Tag,X-Query-Tag,X-Search-State,X-Search-CustomEvent,X-User-Id,X-Search-Client,X-Enable-Telemetry,X-Timestamp
      nginx.ingress.kubernetes.io/cors-allow-methods:
        GET, PUT, POST, DELETE, PATCH,
        OPTIONS, HEAD
      nginx.ingress.kubernetes.io/enable-cors: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: 100m
      nginx.ingress.kubernetes.io/rewrite-target: /
      nginx.ingress.kubernetes.io/ssl-redirect: "false"
    generation: 2
    name: appbase-ingress
    namespace: default
  spec:
    rules:
      - http:
          paths:
            - backend:
                serviceName: appbase
                servicePort: 8000
              path: /
        # host: es.mydomain.com
    tls:
      - hosts:
        - es.mydomain.com
      - secretName: ssl
  EOF
  ```

  > Note: if you do not update the host, then also you will be able to access the appbase URL using Load Balancer IP.

  You can update the [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) certificate, Key and Root CA values in the following command and execute it so that the correct certificate is attached to your domain. Make sure that the values of certificates and keys are encoded in base64.

  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Secret
  metadata:
      name: ssl
  type: Opaque
  data:
      tls.crt: BASE64 ENDCODED CERTIFICATE CONTENT
      tls.key: BASE64 ENCODED KEY CONTENT
      ca.crt: BASE64 ENCODED ROOT CERTIFICATE CONTENT
  EOF
  ```

  🚀 Hurray! our deployment is complete.

- **Step 11 -** Access Appbase.io Dashboard. Now that all our configurations are complete, in order to access all the Appbase.io Services, let us sign in to the [appbase.io dashboard](https://dash.appbase.io/login).
  - Visit <https://dash.appbase.io>
  - Enter appbase URL, i.e. Cluster IP obtained in step 8 or domain configured on step 10.
  - Enter master credentials, i.e. username and password configured on step 7.

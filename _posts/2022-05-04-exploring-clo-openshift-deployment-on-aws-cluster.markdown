---
layout: default
title:  "Exploring CLO Openshift deployment on AWS cluster"
date:   2022-05-04 00:58:43 +0530
categories: jekyll update
---

Login to the Openshift cluster using

{% highlight shell %}
oc login -u kubeadmin -p <password> <cluster-link>
{% endhighlight %}

Cluster Logging Operator (CLO) - [github](https://github.com/openshift/cluster-logging-operator)

After performing `oc login`, navigate to the CLO repository and run `make dpeloy` to deploy CLO on the openshift cluster.

The cluster logging operator, manages the `ClusterLogging` and `ClusterLogForwarder` custom resources. We will deploy the `ClusterLogging` custom resource
with Vector as the collector. Right now support for vector in CLO is in preview stage so the annotation `logging.openshift.io/preview-vector-collector` has to be
added as well. Vector in a CLO environment gets its configuration (the vector.toml file) from `collector-config` secret. Setting the `managementState` field to `Unmanaged` under the `ClusterConfig` schema should allow us to manually edit that secret.

After the necessary changes the `hack/cr-vector.yaml` file in the CLO repository should look like this

{% highlight yaml %}
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance"
  namespace: "openshift-logging"
  annotations:
    logging.openshift.io/preview-vector-collector: enabled
spec:
  managementState: Unmanaged
  logStore:
    type: "elasticsearch"
    elasticsearch:
      nodeCount: 3
      resources:
        requests:
          limits: 2Gi
      redundancyPolicy: "ZeroRedundancy"
  visualization:
    type: "kibana"
    kibana:
      replicas: 1
  collection:
    logs:
      type: "vector"
      fluentd: {}
{% endhighlight %}

Deploy the `ClusterLogging` custom resource using the below command

{% highlight shell %}
oc apply -f hack/cr-vector.yaml
{% endhighlight %}

![Getting ClusterLogging resources in openshift-logging namespace](/assets/images/2022-05-04/1.png)

The below command gets the `collector-config` secret, base64 decrypts it and views it.

{% highlight shell %}
oc -n openshift-logging get secret collector-config -ojsonpath='{.data.vector\.toml}' | base64 –d
{% endhighlight %}

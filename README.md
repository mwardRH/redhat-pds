

# Couchbase Operator For OpenShift

The Couchbase Operator allows users to deploy Couchbase on OpenShift while significantly reducing the operational complexities typically association with managing and scaling databases.

Related resources:

- [Autonomous Operator Overview](https://docs.couchbase.com/operator/1.0/overview.html)
- [Installing on OpenShift](https://docs.couchbase.com/operator/1.0/install-openshift.html)
- TODO: insert link to RHPDS setup steps.

## Summary 

This guide is intended to be run with with Red Hat's [Partner Demo System](https://www.redhat.com/en/partners/rhpds) for demos and workshops. It provides steps and examples for deploying Couchbase clusters using the Autonomous operator.

# Setup

### Login to RH PDS Open Shift

The first step is to login to Open Shift in your local browser AND terminal.

Open https://master.couchbase.openshiftworkshop.com/login in your browser and login with **user1**'s credentials:

- username: user1
- password: openshift


Now we will login via terminal using **opentlc-mgr**'s credentials:

```
oc login https://master.couchbase.openshiftworkshop.com
```

- username: opentlc-mgr
- password: r3dh4t1!

> Note: opentlc-mgr is an admin account. Admin privileges are needed in order to install Custom Resource Definitions (CRDs).

### Create Project

Next, we need to create a project to work in. 

```
oc new-project operator-example
```

This command creates the `operator-example` project and switches to it.

### Deploy the Operator CRD

```
oc create -f crd.yaml
```

### Create Roles, Accounts & Bindings

```
oc create -f cluster-role-sa.yaml
oc create -f cluster-role-user.yaml

oc create serviceaccount couchbase-operator --namespace operator-example

oc create rolebinding couchbase-operator --clusterrole couchbase-operator --serviceaccount operator-example:couchbase-operator

oc adm policy add-scc-to-user anyuid system:serviceaccount:operator-example:couchbase-operator

oc create rolebinding couchbasecluster --clusterrole couchbasecluster --user developer --namespace operator-example

oc create clusterrolebinding couchbasecluster --clusterrole couchbasecluster --user developer

```

### Create Red Hat Registry Secrets

Before we can deploy the operator we need to specify credentials for pulling container images from Red Hat's registry and add them to the service accounts.

Replace USERNAME and PASSWORD and EMAIL below with your Red Hat account info.

```
oc create secret docker-registry rh-catalog --docker-server=registry.connect.redhat.com --docker-username=redcouch --docker-password=openshift --docker-email=redcouchredhat@gmail.com

oc secrets add serviceaccount/couchbase-operator secrets/rh-catalog --for=pull
oc secrets add serviceaccount/default secrets/rh-catalog --for=pull
```

### Deploy the Operator

```
oc create -f operator.yaml
```

After < 1 minute you should be able to see the operator running:

```
oc get pods
NAME                                  READY     STATUS    RESTARTS   AGE
couchbase-operator-5bc785c54f-kh6c2   1/1       Running   0          22s
```

**Do not proceed** to the next step until you see the `couchbase-operator` pod has status `RUNNING`.

### Deploy Couchbase Credentials Secret

The Couchbase clusters deployed in the following steps will use the credentials provided in the `cb-example-auth` secret. Deplying `secret.yaml` will create the secret.

```
oc create -f secret.yaml
```

# Cluster Recipes

### Deploy a Basic Couchbase Cluster

The first cluster that we'll deploy will be a simple, 4 node cluster, with one bucket and 2 replicas.

```
oc create -f cluster.yaml
```

You should start seeing Couchbase pods appearing immediately. It will take a couple of minutes for the cluster to be ready.

```
oc get pods
NAME                                  READY     STATUS    RESTARTS   AGE
cb-example-0000                       1/1       Running   0          3m
cb-example-0001                       1/1       Running   0          3m
cb-example-0002                       1/1       Running   0          2m
cb-example-0003                       1/1       Running   0          2m
couchbase-operator-5bc785c54f-kh6c2   1/1       Running   0          7m
```

### Expose the Couchbase UI

Next, expose the Couchbase UI so you can log into it:

```
oc expose service/cb-example-ui
```

Get the route to the Couchbase UI:

```
oc get routes
NAME            HOST/PORT                                                             PATH      SERVICES        PORT        TERMINATION   WILDCARD
cb-example-ui   cb-example-ui-operator-example.apps.couchbase.openshiftworkshop.com             cb-example-ui   couchbase                 None
``` 

Open the URL outputted by `oc get routes` in your browser and login with "Administrator/password". Navigate to "Servers" to see the server list:

![Basic Couchbase Cluster](img/cb-cluster-basic.png)

On the Pods page in OpenShift (https://master.couchbase.openshiftworkshop.com/console/project/operator-example/browse/pods):

![](img/os-cluster-basic.png)

### Failover Demo

Now that we have a cluster up and some data, we can demonstrate the operator in action. 

First, delete one of the pods:

```
oc delete pod cb-example-0003
```

By deleting the pod, we are destroying one of the Couchbase nodes. At this point the operator should take over and try to recover the cluster to our desired state.

Couchbase recognizes that a node is missing and triggers fail-over:

![](img/failover-1.png)

Couchbase recognizes the new node coming online and begins rebalancing:

![](img/failover-2.png)

The rebalance conintues until the cluster is fully healed.

![](img/failover-3.png)

### Cleanup

Delete the cluster before moving onto the next example:

```
oc delete -f cluster-basic.yaml

```


## Deploy a Cluster with Server Groups Enabled

First, we need to add labels to our OpenShift nodes. Labels are used to tell the Operator which zone a particular node belongs to. In this example, we'll declare the node1 and node2 belong to ServerGroup1 and node2 and node3 belong to ServerGroup2.

```
 oc label --overwrite nodes node1.couchbase.internal server-group.couchbase.com/zone=ServerGroup1
 oc label --overwrite nodes node2.couchbase.internal server-group.couchbase.com/zone=ServerGroup1
 oc label --overwrite nodes node3.couchbase.internal server-group.couchbase.com/zone=ServerGroup2
 oc label --overwrite nodes node4.couchbase.internal server-group.couchbase.com/zone=ServerGroup2
```

Now deploy the new cluster:

```
oc create -f cluster-server-groups.yaml
```

This deploys a 9 node cluster with ServerGroups enabled. The distribution of nodes is setup so that each zone has 2 Data nodes and 1 Query node. This allows us to keep 2 replicas of the defauly bucket in each zone.

![](img/9node-server-list.png)

# Deploy an Application


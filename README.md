# crdb-kube-workshop

Welcome to an hours workshop and demonstration on how to install and monitor CockroachDB inside Kubernetes, by the end of this tutorial you will understand how to manage and deploy CockroachDB clusters, have an understanding of it's ability to scale fast and survive anything and how to connect a demo application to the database.

## PreRequisites

A local or remote Kubernetes environment that is accessible from your machine, if you need some inspiration for local distributions, check out these: k3d, Minikube, Docker Desktop or Rancher Desktop.

* Kubectl (https://kubernetes.io/docs/tasks/tools/)
* Cockroach CLI (https://www.cockroachlabs.com/docs/v22.1/install-cockroachdb.html)
* Git (https://github.com/git-guides/install-git)


## Deploy CockroachDB into your k8s cluster

Create a variable that represents a region, as we're in a local environment we'll only use one to save on resources in this workshop.

```
export region="eu-west-2"
```

Create a namespace corresponding to our dummy region

```
kubectl create namespace $region
```

CockroachDB needs a number of certificates for node communication and client authentication, so we must create these first.

```
mkdir certs my-safe-directory
```

The first thing to be created is the CA

```
cockroach cert create-ca \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Then a client certificate is needed

```
cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

We can pass these secrets in to CockroachDB by using Kubernetes secrets, we'll deploy these in to our dummy region namespace

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--namespace $region
```

Create the node certificates for our region

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$region \
cockroachdb-public.$region.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$region" \
"*.cockroachdb.$region.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

This also needs to be deployed as a secret

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--namespace $region
```

Once this is deployed, we can remove the local copy of the certs

```
rm certs/node.crt
rm certs/node.key
```

Deploy the CRDB StatefulSet.
> There are some hard coded region names in the manifest. If you have changed the region name you will need to edit the file. You may also want to adjust the replica count and resource requests and limits depending on your computer spec.

```
kubectl apply -f manifests/cockroachdb-statefulset-secure.yaml -n $region
```

Once the pods are deployed we need to initialize the cluster. This is done by 'execing' into the container and running the `cockroach init` command.
```
kubectl exec \
--namespace $region \
-it cockroachdb-0 \
-- /cockroach/cockroach init \
--certs-dir=/cockroach/cockroach-certs
```

Check that the pod has started successfully.

```
kubectl get pods --namespace $region
```

Next, create a secure client in the region to connect to CRDB
```
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $region
```

Now exec into the pod using the CockraochDB SQL client, which will be used to connect to the database.
```
kubectl exec -it cockroachdb-client-secure -n $region -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```

Create a user and make that user an admin, we will use this to login to the Console and monitor the database.
```
CREATE USER craig WITH PASSWORD 'cockroach';
GRANT admin TO craig;
\q
```

Port-forward to the cockroachDB pod

```
kubectl port-forward cockroachdb-0 8080:8080 -n $region
```

In your browswer open the following url
```localhost:8080```

Login with the user and password created earlier
```
User: Craig
Password: cockroach
```
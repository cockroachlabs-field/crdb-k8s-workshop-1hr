# crdb-kube-workshop

Welcome to an hours workshop and demonstration on how to install and monitor CockroachDB inside Kubernetes, by the end of this tutorial you will understand how to manage and deploy CockroachDB clusters, have an understanding of it's ability to scale fast and survive anything and how to connect a demo application to the database.

## PreRequisites

A local or remote Kubernetes environment that is accessible from your machine, if you need some inspiration for local distributions, check out these: k3d, Minikube, Docker Desktop or Rancher Desktop.

* Kubectl (https://kubernetes.io/docs/tasks/tools/)
* Cockroach CLI (https://www.cockroachlabs.com/docs/v22.1/install-cockroachdb.html)
* Git (https://github.com/git-guides/install-git)
* Ked (https://k3d.io/v5.4.6/)

### Cockroach CLI Installation

```
brew install cockroachdb/tap/cockroach
```
Or if not using Brew
```
curl https://binaries.cockroachdb.com/cockroach-v22.1.10.darwin-10.9-amd64.tgz | tar -xJ && cp -i cockroach-v22.1.10.darwin-10.9-amd64/cockroach /usr/local/bin/
```

### Git Installation

```
brew install git
```
Or if not using Brew
```
Download the MacOS Git Installer: https://sourceforge.net/projects/git-osx-installer/files/git-2.23.0-intel-universal-mavericks.dmg/download?use_mirror=autoselect
```

### Kubectl Installation

```
brew install kubectl
```
Or if not using Brew (Swap to arm64 if using Silicon)
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
```
```
chmod +x ./kubectl
```
```
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
```

### k3d Installation

```
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### Create k3d Cluster (Optional if using something else)

```
k3d cluster create kube-workshop
```

```
kubectl get nodes
```


## Clone this repository

```
git clone https://github.com/cockroachlabs-field/crdb-k8s-workshop-1hr.git
```

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

NOTE: You may see that there are under replicated ranges, this is because we're currently only running a single replica, observe how this changes throughout the workshop.

## Run a workload against CockroachDB (Single Node)

In this section of the workshop we will run a sample workload against the Database, this workload specifically models a set of accounts with currency balances.

The first thing we need to do is initialise the workload using our secure client pod we deployed earlier, begin by opening a new terminal as we want to remain port-fowarded to the UI for monitoring purposes.

In your new terminal window run

```
export region="eu-west-2"
```

```
kubectl exec -it cockroachdb-client-secure -n $region  -- ./cockroach workload init bank 'postgresql://craig:cockroach@cockroachdb-public:26257'
```

Now we can begin running the workload against our cockroach deployment, the below command will continuiously simulate workload in to the database for 15 minutes.

```
kubectl exec -it cockroachdb-client-secure -n $region -- ./cockroach workload run bank --duration=15m --tolerate-errors 'postgresql://craig:cockroach@cockroachdb-public:26257'
```

Take a look in the metrics section of the UI, what do you observe here?

Once you're happy, hit ctrl-c on the window and stop the workload but leave this terminal window open, we'll re-run this once we've scaled up.

## Scaling the Database

Due to the nature of CockroachDBs architecture and the way it has been built from the ground up to run inside of Kubernetes, scaling becomes very easy. In this section we will scale the cluster to 3 nodes and observe the behaviours in the UI whilst continuing to run the workload in the previous steps.

First of all, open one more terminal window, this is so that we can keep the port-forward and running of the workload intact as we scale our cluster.

Set the region again

```
export region="eu-west-2"
```

Then, is as simple as running this command to change the amount of nodes running in our cluster.

```
kubectl scale sts cockroachdb --replicas=3 -n $region
```

Verify that there are now 3 sets of Cockroach pods running. (Note: 3 Nodes are the absolute mininum for High Availbility, we now have the ability to lose 1 node and still remain active. Scaling to 1 will break the cluster)

```
kubectl get pods --namespace $region
```

Monitor the UI, what is happening now 2 new nodes have been added to the cluster? 

Restart the workload job in your other terminal

```
kubectl exec -it cockroachdb-client-secure -n $region -- ./cockroach workload run bank --duration=15m --tolerate-errors 'postgresql://craig:cockroach@cockroachdb-public:26257'
```

Monitor the UI again, what is happening now the workload is running? What metrics do you see across the nodes?

Once you're happy, keep the workload running and move on to the next task


## Chaos Testing CockroachDB

It's important to demonstrate that your database and applications can survive different types of failure, perhaps the machine running your Kubernetes node has lost power, or an availbility zone has failed in AWS. In this simple example we will demonstrate the complete loss of a node in CockroachDB and talk about how the database continues to work during this loss.

Using the same Terminal we used to scale the database, we will start deleting a couple of nodes. (We will not be deleting the pod cockroachdb-0, this is simply because we're using that to port-forward in to the UI)

```
kubectl delete pods cockroachdb-1 -n $region
```

What do you see in the UI? Is your workload still running against the cluster? You should see Kubernetes restores this node quite quickly, you can verify the pod has come back by running

```
kubectl get pods -n $region
```

Try killing the 3rd node, the expected behaviour should be the same.

```
kubectl delete pods cockroachdb-2 -n $region
```

Now lets scale the statefulset down to just 2 replicas, this is similar to removing one node, however this time the node won't come back.

```
kubectl scale sts cockroachdb --replicas=2 -n $region
```

Observe the UI again, do you notice anything different now that the 3rd node is completely gone? Is everything still working as expected?

Finally lets scale the statefulset to 1

```
kubectl scale sts cockroachdb --replicas=1 -n $region
```

In this scenario the cluster should be in a none working state and access to the UI will be lost, this is because we have lost Quorum.

Finally, lets scale it back to 3

```
kubectl scale sts cockroachdb --replicas=3 -n $region
```

Has service now resumed? What is happening in the UI?

## Cleaning up

Now that we have demonstrated the capabilities of running CockroachDB in Kubernetes, we can clean up the resources that we have created by running the below commands.

```
kubectl delete -f manifests/cockroachdb-statefulset-secure.yaml -n $region
```
```
kubectl delete ns $region
```
```
rm -rf certs my-safe-directory
```

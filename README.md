# Patient Portal

[![main](https://github.com/ssorj/skupper-example-patient-portal/actions/workflows/main.yaml/badge.svg)](https://github.com/ssorj/skupper-example-patient-portal/actions/workflows/main.yaml)

#### A simple database-backed web application that runs in the public cloud but keeps its data in a private database

This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html

#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Configure separate console sessions](#step-1-configure-separate-console-sessions)
* [Step 2: Access your clusters](#step-2-access-your-clusters)
* [Step 3: Set up your namespaces](#step-3-set-up-your-namespaces)
* [Step 4: Install Skupper in your namespaces](#step-4-install-skupper-in-your-namespaces)
* [Step 5: Check the status of your namespaces](#step-5-check-the-status-of-your-namespaces)
* [Step 6: Link your namespaces](#step-6-link-your-namespaces)
* [Step 7: Deploy and expose the database](#step-7-deploy-and-expose-the-database)
* [Step 8: Deploy and expose the payment processor](#step-8-deploy-and-expose-the-payment-processor)
* [Step 9: Deploy and expose the frontend](#step-9-deploy-and-expose-the-frontend)
* [Step 10: Test the application](#step-10-test-the-application)
* [Cleaning up](#cleaning-up)
* [Next steps](#next-steps)

## Overview

This example is a simple database-backed web application that shows
how you can use Skupper to access a database at a remote site
without exposing it to the public internet.

It contains three services:

  * A PostgreSQL database running on a bare-metal or virtual
    machine in a private data center.

  * A payment-processing service running on Kubernetes in a private
    data center.

  * A web frontend service running on Kubernetes in the public
    cloud.  It uses the PostgreSQL database and the
    payment-processing service.

This example uses two Kubernetes namespaces, "private" and "public",
to represent the private Kubernetes cluster and the public cloud.

## Prerequisites

* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* The `skupper` command-line tool, the latest version ([installation
  guide][install-skupper])

* Access to at least one Kubernetes cluster, from any provider you
  choose

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[install-skupper]: https://skupper.io/install/index.html

## Step 1: Configure separate console sessions

Skupper is designed for use with multiple namespaces, typically on
different clusters.  The `skupper` command uses your
[kubeconfig][kubeconfig] and current context to select the namespace
where it operates.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

Your kubeconfig is stored in a file in your home directory.  The
`skupper` and `kubectl` commands use the `KUBECONFIG` environment
variable to locate it.

A single kubeconfig supports only one active context per user.
Since you will be using multiple contexts at once in this
exercise, you need to create distinct kubeconfigs.

Start a console session for each of your namespaces.  Set the
`KUBECONFIG` environment variable to a different path in each
session.

**Console for _public_:**

~~~ shell
export KUBECONFIG=~/.kube/config-public
~~~

**Console for _private_:**

~~~ shell
export KUBECONFIG=~/.kube/config-private
~~~

## Step 2: Access your clusters

The methods for accessing your clusters vary by Kubernetes provider.
Find the instructions for your chosen providers and use them to
authenticate and configure access for each console session.  See the
following links for more information:

* [Minikube](https://skupper.io/start/minikube.html)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html)
* [OpenShift](https://skupper.io/start/openshift.html)
* [More providers](https://kubernetes.io/partners/#kcsp)

## Step 3: Set up your namespaces

Use `kubectl create namespace` to create the namespaces you wish to
use (or use existing namespaces).  Use `kubectl config set-context` to
set the current namespace for each session.

**Console for _public_:**

~~~ shell
kubectl create namespace public
kubectl config set-context --current --namespace public
~~~

Sample output:

~~~ console
$ kubectl create namespace public
namespace/public created

$ kubectl config set-context --current --namespace public
Context "minikube" modified.
~~~

**Console for _private_:**

~~~ shell
kubectl create namespace private
kubectl config set-context --current --namespace private
~~~

Sample output:

~~~ console
$ kubectl create namespace private
namespace/private created

$ kubectl config set-context --current --namespace private
Context "minikube" modified.
~~~

## Step 4: Install Skupper in your namespaces

The `skupper init` command installs the Skupper router and service
controller in the current namespace.  Run the `skupper init` command
in each namespace.

**Note:** If you are using Minikube, [you need to start `minikube
tunnel`][minikube-tunnel] before you install Skupper.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

**Console for _public_:**

~~~ shell
skupper init
~~~

Sample output:

~~~ console
$ skupper init
Waiting for LoadBalancer IP or hostname...
Skupper is now installed in namespace 'public'.  Use 'skupper status' to get more information.
~~~

**Console for _private_:**

~~~ shell
skupper init
~~~

Sample output:

~~~ console
$ skupper init
Waiting for LoadBalancer IP or hostname...
Skupper is now installed in namespace 'private'.  Use 'skupper status' to get more information.
~~~

## Step 5: Check the status of your namespaces

Use `skupper status` in each console to check that Skupper is
installed.

**Console for _public_:**

~~~ shell
skupper status
~~~

**Console for _private_:**

~~~ shell
skupper status
~~~

You should see output like this for each namespace:

~~~
Skupper is enabled for namespace "<namespace>" in interior mode. It is not connected to any other sites. It has no exposed services.
The site console url is: http://<address>:8080
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

As you move through the steps below, you can use `skupper status` at
any time to check your progress.

## Step 6: Link your namespaces

Creating a link requires use of two `skupper` commands in conjunction,
`skupper token create` and `skupper link create`.

The `skupper token create` command generates a secret token that
signifies permission to create a link.  The token also carries the
link details.  Then, in a remote namespace, The `skupper link create`
command uses the token to create a link to the namespace that
generated it.

**Note:** The link token is truly a *secret*.  Anyone who has the
token can link to your namespace.  Make sure that only those you trust
have access to it.

First, use `skupper token create` in one namespace to generate the
token.  Then, use `skupper link create` in the other to create a link.

**Console for _public_:**

~~~ shell
skupper token create ~/secret.token
~~~

Sample output:

~~~ console
$ skupper token create ~/secret.token
Token written to ~/secret.token
~~~

**Console for _private_:**

~~~ shell
skupper link create ~/secret.token
~~~

Sample output:

~~~ console
$ skupper link create ~/secret.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link1)
Check the status of the link using 'skupper link status'.
~~~

If your console sessions are on different machines, you may need to
use `sftp` or a similar tool to transfer the token securely.  By
default, tokens expire after a single use or 15 minutes after
creation.

## Step 7: Deploy and expose the database

Use `docker` to run the database service on your local machine.
In the public namespace, use the `skupper gateway expose`
command to expose the database on the Skupper network.

Use `kubectl get service/database` to ensure the database
service is available.

**Console for _public_:**

~~~ shell
docker run --detach --rm -p 5432:5432 quay.io/ssorj/patient-portal-database
skupper gateway expose database localhost 5432 --type docker
kubectl get service/database
~~~

Sample output:

~~~ console
$ skupper gateway expose database localhost 5432 --type docker
2022/05/19 16:37:00 CREATE io.skupper.router.tcpConnector fancypants-jross-egress-database:5432 map[address:database:5432 host:localhost name:fancypants-jross-egress-database:5432 port:5432 siteId:0e7b70cf-1931-4c93-9614-0ecb3d0d6522]

$ kubectl get service/database
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
database   ClusterIP   10.104.77.32   <none>        5432/TCP   15s
~~~

## Step 8: Deploy and expose the payment processor

In the private namespace, use the `kubectl apply` command to
deploy the payment processor service.  Use the `skupper expose`
command to expose the service on the Skupper network.

In the public namespace, use `kubectl get service/payment-processor` to
check that the `payment-processor` service appears after a
moment.

**Console for _private_:**

~~~ shell
kubectl apply -f payment-processor/kubernetes.yaml
skupper expose deployment/payment-processor --port 8080
~~~

Sample output:

~~~ console
$ kubectl apply -f payment-processor/kubernetes.yaml
deployment.apps/payment-processor created

$ skupper expose deployment/payment-processor --port 8080
deployment payment-processor exposed as payment-processor
~~~

**Console for _public_:**

~~~ shell
kubectl get service/payment-processor
~~~

Sample output:

~~~ console
$ kubectl get service/payment-processor
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
payment-processor   ClusterIP   10.103.227.109   <none>        8080/TCP   1s
~~~

## Step 9: Deploy and expose the frontend

In the public namespace, use the `kubectl apply` command to
deploy the frontend service.  This also sets up an external load
balancer for the frontend.

**Console for _public_:**

~~~ shell
kubectl apply -f frontend/kubernetes.yaml
~~~

Sample output:

~~~ console
$ kubectl apply -f frontend/kubernetes.yaml
deployment.apps/frontend created
service/frontend created
~~~

## Step 10: Test the application

Now we're ready to try it out.  Use `kubectl get service/frontend` to
look up the external IP of the frontend service.  Then use `curl` or a
similar tool to request the `/api/health` endpoint at that address.

**Note:** The `<external-ip>` field in the following commands is a
placeholder.  The actual value is an IP address.

**Console for _public_:**

~~~ shell
kubectl get service/frontend
curl http://<external-ip>:8080/api/health
~~~

Sample output:

~~~ console
$ kubectl get service/frontend
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
frontend   LoadBalancer   10.103.232.28   <external-ip>   8080:30407/TCP   15s

$ curl http://<external-ip>:8080/api/health
OK
~~~

If everything is in order, you can now access the web interface by
navigating to `http://<external-ip>:8080/` in your browser.

## Cleaning up

To remove Skupper and the other resources from this exercise, use the
following commands.

**Console for _public_:**

~~~ shell
skupper gateway delete
skupper delete
kubectl delete service/frontend
kubectl delete deployment/frontend
~~~

**Console for _private_:**

~~~ shell
skupper delete
kubectl delete deployment/payment-processor
~~~

## Next steps

Check out the other [examples][examples] on the Skupper website.

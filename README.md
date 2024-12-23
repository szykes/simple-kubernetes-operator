# simple-kubernetes-operator (so)

![CI](https://github.com/szykes/simple-kubernetes-operator/actions/workflows/ci.yml/badge.svg) ![Docker](https://github.com/szykes/simple-kubernetes-operator/actions/workflows/docker.yml/badge.svg)

As the git project names says this is a really simple kubernetes operator implementation.

The SimpleOperator (`so`) deploys your application and makes it accessible from outside. So,:
- Creates a Deployment based on your application and number of replicas.
- Creates a Service to make Deployment available within the cluster.
- Creates an Ingress to make available your app to the users.

The following use cases are implemented (objects mean Deployments, Service, and Ingress):
- When the `so` object is created, the Operator will deploy objects based on the `so`.
- When the `so` object is changing (Host, Image, Replicas), the Operator will modify objects based on the change.
- When the `so` is triggered to be deleted and:
   - and the Operator is already running, the Operator will delete the objects, then removes the finalizer on `so`, so the deletion of `so` succeeds.
   - and the Operator starts later, the Operator will be notified about the deletion trigger event. It will delete the objects, then removes the finalizer on `so`, so the deletion of `so` succeeds.
- When someone changes the relevant part of objects, the Operator will undo thoses changes based on the `so`.

The Status of `so` wants to be transparent; therefore, the following information is available:
- Current State of Deployments, Service, and Ingress, possible values: Reconciled, Reconciling, Creating, Deleted, InternalError, etc.
- If the Current Status needs an explanation, then some details appear here to give more info what is happening.
- Available Replicas
- Last Updated.

This is never meant to be a real product.

The `config/samples/` contains an example for `simple-kubernetes-operator` with simple `NGINX` image.

> All commands must executed at level of git project root

## tl;dr

Steps to make `simpleoperator` work on a `kind` based cluster:

Create a cluster with `kind`:
```
kind create cluster --name=simple-operator --config=simple-1-control-2-workers.yaml
```

Setup `NGINX` for `kind`:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait until the `NGINX` is deployed:
```
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s
```

Setup `cert-manager`:
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```

Install the staging issuer:
```
kubectl create -f staging-issuer.yml
```

Deploy `simpleoperator`:
```
kubectl apply -f simpleoperator-0.0.1-deploy-in-cluster.yaml
```

Test with:
```
kubectl apply -f config/samples/
kubectl edit so simpleoperator-sample
kubectl delete -f config/samples/
```

## Prerequisite

Having installed `docker`, `kubectl` (v1.31.1), and `kind` (v0.25.0) on a Linux based server.

Server has CPU Intel J3455, 8 GB RAM, and having 60 GB free space for /.

Clone or download the repo.

### Setup a `kind` based cluster

Create cluster with `kind`:
```
kind create cluster --name=simple-operator --config=simple-1-control-2-workers.yaml
```

If everything goes well, the `$HOME/.kube/config` will contain the certificates, context, etc. of `simple-operator` as with name `kind-simple-operator`.

Just run to verify above statement:
```
kubectl cluster-info
```
You must see this:
```
Kubernetes control plane is running at https://127.0.0.1:36279
CoreDNS is running at https://127.0.0.1:36279/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Now we have a cluster environment.

Reference:
[phoenixNAP - Guide to Running Kubernetes with Kind](https://phoenixnap.com/kb/kubernetes-kind)

### Setup `NGINX` as Ingress in `kind`

Setup `NGINX` for `kind`:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait until the `NGINX` is deployed:
```
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s
```

Check what resources are deployed:
```
kubectl get all --namespace ingress-nginx
```

Reference:
[kind - Ingress](https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx)

### Domain name question

My advantage is I own a domain name and the network infrastructre has been already prepared to use it. However, in an ordinary home this is not available, so I would like to give you some hints what you need to do. But before doing anything you need to check wheter your router is behind [CGN](https://www.a10networks.com/glossary/what-is-carrier-grade-nat-cgn-cgnat/). Being under CGN makes harder your life to use HTTP-01 challange, either asking your ISP to give public IP address, using the VPN, or switching to DNS-01 challange can help in this case.

If the WAN IP address on your router and your IP address on [whatsmyip](https://www.whatsmyip.org) are not matching, it will mean your router is under CGN.

Hints:
* Set static IP address for the `kind` runner machine in router -> find DHCP server settings on the router and manually assgin IP to MAC address of machine.
* Open port 80 & 443 -> find Port Forwading menu and set internal and external ports to 443 and use static IP address for internal IP address.
* Sign up on [no-ip](https://www.noip.com) and create a No-IP Hostname -> After the login navigate to Dynamic DNS, No-IP Hostnames and Create Hostname. You can use whatever hostname but leave the Record Type on DNS Host (A).

Please do not forget ISP gives you dynamic IP address to your router that may change, so you need to update the IP address of you No-IP Hostname. I don't know wheter there is an automatic way.

Let's use the [FQDN](https://www.techtarget.com/whatis/definition/fully-qualified-domain-name-FQDN) e.g.: `szykes.ddns.net` in Ingress.

If you don't want to access outside the cluster, then use [nip.io](https://nip.io) just for fun.

### Setup `cert-manager` with `Let's Encrypt` for `NGINX`

Setup `cert-manager`:
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```

Check what resources are deployed:
```
kubectl get pods --namespace cert-manager
```

Install the staging issuer:
```
kubectl create -f staging-issuer.yml
```

Check the current status of certificate creation:
```
kubectl get certificate -o wide
```

I use staging issuer because I can verify TLS certifcate mechanism in this way without bothering the production side of `Let's encrypt`.

If everything goes well, you will see something like this:




<img width="553" alt="Screenshot 2023-03-17 at 19 31 36" src="https://user-images.githubusercontent.com/8822138/226025230-db70d767-340a-4268-a7f1-d72986f59cbb.png">

Reference:

[DigitalOcean - How to Set Up an Nginx Ingress with Cert-Manager on DigitalOcean Kubernetes](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes#step-4-installing-and-configuring-cert-manager)

[cert-manager - Troubleshooting Problems with ACME / Let's Encrypt Certificates](https://cert-manager.io/docs/troubleshooting/acme/)

[kubernetes - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)

## Project creation

### Custom Resource Controller

Install `go` (1.22+) & `kubebuilder` (4.3.1) at first.

Change directory to git project and execute:
```
kubebuilder init --domain szikes.io --repo github.com/szikes-adam/simple-kubernetes-operator

kubebuilder create api --group simpleoperator --version v1alpha1 --kind SimpleOperator
```
+ extend manually the api/v1alpha1/simpleoperator_types.go based on [kubebuilder - CRD validation](https://book.kubebuilder.io/reference/markers/crd-validation.html)

Reference:

[kubebuilder - Tutorial: Building CronJob](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html)

[kubebuilder - Adding a new API](https://book.kubebuilder.io/cronjob-tutorial/new-api.html)

### Operator

Reference:

[kubernetes - Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

[kubernetes - Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

[The Cluster API Book - Implementer's Guide](https://cluster-api.sigs.k8s.io/developer/providers/implementers-guide/overview.html)

[Kubernetes: What is "reconciliation"?](https://speakerdeck.com/thockin/kubernetes-what-is-reconciliation)

[Medium - 10 Things You Should Know Before Writing a Kubernetes Controller](https://medium.com/@gallettilance/10-things-you-should-know-before-writing-a-kubernetes-controller-83de8f86d659)

[kubernetes blog - Using Finalizers to Control Deletion](https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/)

[banzaicloud/k8s-objectmatcher](https://github.com/banzaicloud/k8s-objectmatcher)

And so many other pages...

### Build controller

If you made API changes then run:
```
make manifests
```

But you can skip the previous step because the following will genreate CRD and install on cluster:
```
make install
```

```
export ENABLE_WEBHOOKS=false
make run
```

Reference:
[kubebuilder - Running and deploying the controller](https://book.kubebuilder.io/cronjob-tutorial/running.html)

### Run tests

Just run:
```
make test
```

### See log of deployed controller

If the manual testing seems ok with `make run` then let's jump into the production environment. The most easiest way to do this just push the latest code to GitHub and wait for docker image.

Use GitHub's docker image to deploy:
```
make deploy IMG=ghcr.io/szykes/simple-kubernetes-operator:main
```

Let's find where the `simpleoperator` is:
```
kubectl get namespaces
```

Output:
```
NAME                                STATUS   AGE
default                             Active   40m
kube-node-lease                     Active   40m
kube-public                         Active   40m
kube-system                         Active   40m
local-path-storage                  Active   40m
simple-kubernetes-operator-system   Active   28m
```

The `simple-kubernetes-operator-system` seems promising.

What objects are there?
```
kubectl get all --namespace=simple-kubernetes-operator-system
```

Output:
```
NAME                                                                 READY   STATUS    RESTARTS   AGE
pod/simple-kubernetes-operator-controller-manager-867588699d-68rsz   2/2     Running   0          44m

NAME                                                                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/simple-kubernetes-operator-controller-manager-metrics-service   ClusterIP   10.96.97.87   <none>        8443/TCP   44m

NAME                                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/simple-kubernetes-operator-controller-manager   1/1     1            1           44m

NAME                                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/simple-kubernetes-operator-controller-manager-867588699d   1         1         1       44m
```

Finally, the log of `simpleoperator` is here:
```
kubectl logs --namespace=simple-kubernetes-operator-system pod/simple-kubernetes-operator-controller-manager-867588699d-n8j4p
```

Delete `simpleoperator`:
```
kubectl delete --namespace=simple-kubernetes-operator-system deployment.apps/simple-kubernetes-operator-controller-manager service/simple-kubernetes-operator-controller-manager-metrics-service
```

### Do a standalone config

Do a `make deploy` at first, if you have not done it. Make sure this is the wanted tag of docker image:
```
make deploy IMG=ghcr.io/szykes/simple-kubernetes-operator:0.0.2
```

Change the files according to your needs in `config/default`.

Build manually the resources:
```
bin/kustomize build config/default > simpleoperator-0.0.1-deploy-in-cluster.yaml
```

Deploy based on this, or share with anyone because this is portable:
```
kubectl apply -f simpleoperator-0.0.1-deploy-in-cluster.yaml
```

## GitHub Actions

### CI

It builds and vets using by `make`.

Triggered by pushing new commit on `main` and pull request.

File location in project:
`.github/workflows/ci.yml`

Reference:

[GitHub - Building and testing Go](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-go)

[banzaicloud/koperator - ci.yml](https://github.com/banzaicloud/koperator/blob/master/.github/workflows/ci.yml)

### Docker

It builds docker image by using `Dockerfile` at the project root.

The images are availble on `ghcr.io`.

Building and pushing docker images are triggered by pushing new commit on `main` and tag with the following version format `'*.*.*'`. For example: 2.10.5

File location in project:
`.github/workflows/docker.yml`

Reference:

[GitHub - Publishing Docker images](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)

[banzaicloud/koperator - docker.yml](https://github.com/banzaicloud/koperator/blob/master/.github/workflows/docker.yml)

### Accessing docker images

At first read & do: [Creating a personal access token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

Login with docker on the machine that needs access:
```
docker login ghcr.io
```
> It will ask for your username on GitHub and your PAT

If everything does well, you will see this:
```
WARNING! Your password will be stored unencrypted in /home/buherton/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

Verifying access by:
```
docker pull ghcr.io/szykes/simple-kubernetes-operator:0.0.2
```

You should see similar to this:
```
0.0.2: Pulling from szykes/simple-kubernetes-operator
0baecf37abee: Pull complete
bfb59b82a9b6: Pull complete
efa9d1d5d3a2: Pull complete
a62778643d56: Pull complete
7c12895b777b: Pull complete
3214acf345c0: Pull complete
5664b15f108b: Pull complete
0bab15eea81d: Pull complete
4aa0ea1413d3: Pull complete
da7816fa955e: Pull complete
9aee425378d2: Pull complete
a74c5d8d7bda: Pull complete
Digest: sha256:0cab33b161604a7554ecb717ab897042bc3ca9f49f1e665da46852f0818bef7a
Status: Downloaded newer image for ghcr.io/szykes/simple-kubernetes-operator:0.0.2
ghcr.io/szykes/simple-kubernetes-operator:0.0.2
```

Reference:
[GitHub - Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

## Further development

Not all areas of this project were deeply investigated and built.

Here is the list that I would do in a next phase:
* See `TODO`s in the code
* Have a proper versioning (rc, beta, etc.) for git project and docker image
* etc.

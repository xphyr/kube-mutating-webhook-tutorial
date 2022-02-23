# Kubernetes Mutating Webhook for Sidecar Injection

[![Go Report Card](https://goreportcard.com/badge/github.com/xphyr/kube-mutating-webhook-tutorial)](https://goreportcard.com/report/github.com/xphyr/kube-mutating-webhook-tutorial)

This tutorial shows how to build and deploy a [MutatingAdmissionWebhook](https://kubernetes.io/docs/admin/admission-controllers/#mutatingadmissionwebhook-beta-in-19) that injects a nginx sidecar container into pod prior to persistence of the object.

## Introduction

First off Credit where Credit is due. This repo is a fork of https://github.com/morvencao/kube-mutating-webhook-tutorial which has been updated to work with more modern releases of Kubernetes, as well as additional information on how to deploy this for OpenShift. Thanks to [morvencao](https://github.com/morvencao) for his original work on this. See [References](#references) for additional information and inspiration for this work.

NOTE that one of the biggest changes with this version of the Webhook Sidecar Injector is that it has been updated to work with admission v1, and dropped support for v1beta1.

## Prerequisites

- [git](https://git-scm.com/downloads)
- [go](https://golang.org/dl/) version v1.12+
- [docker](https://docs.docker.com/install/) version 17.03+
- [oc](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.11.3+
- Access to an OpenShift 4.9+ cluster with the `admissionregistration.k8s.io/v1` API enabled. Verify that by the following command:

```
oc api-versions | grep admissionregistration.k8s.io
```
The result should be:
```
admissionregistration.k8s.io/v1
```

## Building in a Container

This project is set up to allow building the application inside a container so yo do not need to have the Go toolchain installed. You can use Podman or Docker to build the container image. In this blog post we will be using Podman.

Start by checking out the code from github:

```shell
$ git clone https://github.com/xphyr/kube-mutating-webhook-tutorial.git
Cloning into 'kube-mutating-webhook-tutorial'...
remote: Enumerating objects: 250, done.
remote: Counting objects: 100% (79/79), done.
remote: Compressing objects: 100% (48/48), done.
remote: Total 250 (delta 30), reused 65 (delta 23), pack-reused 171
Receiving objects: 100% (250/250), 242.13 KiB | 1.24 MiB/s, done.
Resolving deltas: 100% (114/114), done.
$ cd kube-mutating-webhook-tutorial
```

With the code cloned locally, we can build the application using containers. We will be using a "multistage" Dockerfile that runs the build for our application in one container, and then uses the results from the first container to populate the second container. Additional information on running a multistage Dockerfile can be found here: [Build your Go image](https://docs.docker.com/language/golang/build-images/) Run the following command to build our container:

```shell
$ podman build -f Dockerfile.multistage -t mwhexample:latest .
[1/2] STEP 1/7: FROM golang:1.17-buster AS build
[1/2] STEP 2/7: WORKDIR /app
--> Using cache 400ea9b25da7730b8cc40d317b6274d898ec5e68a94f9472cdacaf9042675716
--> 400ea9b25da
[1/2] STEP 3/7: COPY go.mod ./
...
[1/2] STEP 7/7: RUN go build -o /mwh-tutorial
--> 107ce1573f1
[2/2] STEP 1/7: FROM gcr.io/distroless/base-debian10
[2/2] STEP 2/7: ENV SIDECAR_INJECTOR=/usr/local/bin/sidecar-injector   USER_UID=1001   USER_NAME=sidecar-injector
--> Using cache b520ffbe03b36eb9a338393f8d907c568e780d15217fbea40e7218260cacc28e
--> b520ffbe03b
...
[2/2] STEP 7/7: ENTRYPOINT ["/mwh-tutorial"]
[2/2] COMMIT mwhexample:latest
--> 19616f6fc07
Successfully tagged localhost/mwhexample:latest
19616f6fc079831ab7f469a4906bc74f73db8a7136ba0975f08ce5ce77111eaa
```

## Building Locally

You can also build the code locally on your machine without the use of containers.

```shell
$ go build -o build/_output/linux/bin/mwh-tutorial ./cmd/
$ podman build -f build/Dockerfile -t mwhexample:latest .
```

## Push Containerimage to Registry

Once the container is built, you will need to publish the container image to an image repository. Your steps may vary based on your particular container registry, in the example below we will push to quay.io.

```shell
$ podman tag mwhexample:latest quay.io/xphyr/mwhexample:latest
$ podman login quay.io
Authenticating with existing credentials for quay.io
Existing credentials are valid. Already logged in to quay.io
$ podman push quay.io/xphyr/mwhexample:latest
```

Our mutating webhook application is now ready for deployment.

## Deployment

Now that we have a container image to work with, we can deploy the Mutating Webhook. In an upstream Kubernetes cluster we would now need to create a x509 certificate to handle authentication within the cluster. However we are running OpenShift, and OpenShift can handle this for us using the "service.beta.openshift.io/inject-cabundle: "true"" annotation and leverage the Kubernetes APIservice.

## Deploy

Now that we have a container image to work with, we can deploy the Mutating Webhook. In an upstream Kubernetes cluster we would now need to create a x509 certificate to handle authentication within the cluster. However we are running OpenShift, and OpenShift can handle this for us using the "service.beta.openshift.io/inject-cabundle: "true"" annotation and leverage the Kubernetes APIservice.

To start, we will create a new project called "sidecar-injector" to install our mutating webhook into:

```shell
$ oc login
$ oc new-project sidecar-injector
Now using project "sidecar-injector" on server "https://api.ocp49.xphyrlab.net:6443".
$ oc create -f deploy/serviceaccount.yaml
$ oc auth reconcile -f deploy/rbac.yaml
$ oc create -f deploy/service.yaml
$ oc create -f deploy/configmap.yaml
$ oc create -f deploy/deployment.yaml
$ oc create -f deploy/apiservice.yaml
```



## Verify

1. The sidecar inject webhook should be in running state:

```
# kubectl -n sidecar-injector get pod
NAME                                                   READY   STATUS    RESTARTS   AGE
sidecar-injector-webhook-deployment-7c8bc5f4c9-28c84   1/1     Running   0          30s
# kubectl -n sidecar-injector get deploy
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
sidecar-injector-webhook-deployment   1/1     1            1           67s
```

2. Create new namespace `injection` and label it with `sidecar-injector=enabled`:

```
# kubectl create ns injection
# kubectl label namespace injection sidecar-injection=enabled
# kubectl get namespace -L sidecar-injection
NAME                 STATUS   AGE   SIDECAR-INJECTION
default              Active   26m
injection            Active   13s   enabled
kube-public          Active   26m
kube-system          Active   26m
sidecar-injector     Active   17m
```

3. Deploy an app in Kubernetes cluster, take `alpine` app as an example

```
# kubectl run alpine --image=alpine --restart=Never -n injection --overrides='{"apiVersion":"v1","metadata":{"annotations":{"sidecar-injector-webhook.morven.me/inject":"yes"}}}' --command -- sleep infinity
```

4. Verify sidecar container is injected:

```
# kubectl get pod
NAME                     READY     STATUS        RESTARTS   AGE
alpine                   2/2       Running       0          1m
# kubectl -n injection get pod alpine -o jsonpath="{.spec.containers[*].name}"
alpine sidecar-nginx
```

## Troubleshooting

Sometimes you may find that pod is injected with sidecar container as expected, check the following items:

1. The sidecar-injector webhook is in running state and no error logs.
2. The namespace in which application pod is deployed has the correct labels as configured in `mutatingwebhookconfiguration`.
3. Check the `caBundle` is patched to `mutatingwebhookconfiguration` object by checking if `caBundle` fields is empty.
4. Check if the application pod has annotation `sidecar-injector-webhook.morven.me/inject":"yes"`.


## References

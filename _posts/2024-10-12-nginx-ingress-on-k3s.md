---
title: "How to setup K3S cluster and NGINX Ingress Controller"
date: 2024-10-12 00:00:00 
categories: [k8s]
tags: [openssl, pfx, certificate, k3s, ingress, nginx, helm]
item: assets/images/post-01-01.png
---

# Introduction
When setting up a Kubernetes environment for testing, K3s is an outstanding choice due to its lightweight architecture and ease of deployment. By default, K3s includes Traefik as its ingress controller, but this may not suit all users' requirements or preferences.

In this post, I will guide you through installing a K3s cluster without Traefik and configuring the NGINX Ingress Controller instead. We will also generate a wildcard certificate using Active Directory Certificate Services (AD CS) to ensure secure communication by default in our ingress controller.

## Prerequisites
To follow this tutorial, you will need:
- One Ubuntu 22.04 server with at least 1GB of RAM pre-configured 
- Active Directory Cerficate Services (ADCS) in installed in a separated Windows box

## Step 1 - Installing K3s without Traefik
In this step, you will install the latest version of K3s on the Ubuntu machine.

Install the K3s without Traefik using the following command:

```bash
sudo curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" K3S_KUBECONFIG_MODE="644" sh -
```

This command downloads and runs a script to install K3s with specific configuration options. 

Let's break it down:
1. ```curl -sfL https://get.k3s.io```: This part downloads the K3s installation script.
    * ```-s```: Silent mode
    * ```-f```: Fail silently on server errors
    * ```-L```: Follow redirects
2. ```|```: Pipes the output of the curl commamnd to the next command.
3. ```INSTALL_K3S_EXEC="--disable traefik"```:  This sets an environment variable that will be used by the installation script. Basically it is telling K3s installer to disable Traefik.
4. ```K3S_KUBECONFIG_MODE="644"```: The kubeconfig file is owned by root, and written with a default mode of 600. Changing the mode to 644 will allow it to be read by other unprivileged users on the host.
4. ```sh -```: This runs the downloaded script using the shell.

To know more about the supported flags access https://docs.k3s.io/cli/server.

You should see an output like below, which shows the steps performed by the installation script:

![]({{ page.item | relative_url }})

Next run the following command to check the k3s status:
```bash
sudo systemctl status k3s
```
The command will show the status as ```active (running)```:
![k3s service status](assets/images/post-01-02.png)

Confirm the cluster is up and running with the command:
```bash
kubectl cluster-info
``` 
![kubectl cluster-info](assets/images/post-01-03.png)

Get Default Kubernetes Objects
```bash
kubectl get all -n kube-system
```
![kubectl all objects](assets/images/post-01-04.png)

As you might have noticed, we did not need to perform any extra steps to access our cluster, simply running ```kubectl``` with was enough to get the cluster info and its objects. 

This is because K3s comes bundled with its own version of ```kubectl```, which automatically reads the ```Kubeconfig``` from the correct path ```/etc/rancher/k3s/k3s.yaml```:

![kubectl k3s version](assets/images/post-01-05.png)

## Step 2 - Install and configure Helm 

There are two primary methods to install the NGINX Ingress Controller on a Kubernetes cluster:

1. Using Kubernetes Manifests
2. Via Helm, a package manager for Kubernetes

In this tutorial, we'll focus on the second method, using Helm to install the NGINX Ingress Controller on our K3s cluster. It's important to note that neither the NGINX Ingress Controller nor Helm come pre-installed with K3s. Both need to be manually installed and configured.

Lets first install Helm, and for that we will follow the instruction found on the official helm doc:

https://helm.sh/docs/intro/install/#from-script

Execute the following commands:
```shell
sudo curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
sudo chmod 700 get_helm.sh
sudo ./get_helm.sh
```
Installation output:
![helm installation](assets/images/post-01-06.png)

If we run now the command helm list -A we will get the error below:
![helm error](assets/images/post-01-07.png)

This is because, unlike `kubectl`, helm is not aware of the kubeconfig file stored at  ```/etc/rancher/k3s/k3s.yaml``` which is used to grant access to our k3s cluster. We need to configure Helm with the correct Kubeconfig path manually.

This can be done by mutiple ways:
1. Exporting the `KUBECONFIG` environment variable 
```shell
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm list -A
```
![helm list](assets/images/post-01-08.png)
2. Specifying the location of the kubeconfig file in the command:
```shell
unset KUBECONFIG
helm --kubeconfig /etc/rancher/k3s/k3s.yaml list -A
```
![helm list](assets/images/post-01-09.png)
3. Copying kubeconfig file to ~/.kube/config and setting the right permissions
```shell
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chown $USER:$USER ~/.kube/config
chmod 700 ~/.kube/config
helm list
```
![helm list](assets/images/post-01-10.png)

By copying the Kubeconfig file to the default ~/.kube/config location allows Helm to automatically recognize it without needing to specify the path or set the KUBECONFIG environment variable each time. This is the method I normally use.

## Step 3 - Generating Wildcard Certificate with ADCS

## Step 4 - Converting PFX to PEM with OpenSSL

## Step 5 - Creating Kubernetes TLS Secret

## Step 6 - Installing NGINX Ingress Controller with Helm

## Step 7 - Creating Deployment, Services and Ingress Objects

## Conclusion





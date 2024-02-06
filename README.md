# About the project

This repo contains instructions and shell commands for creating local minikube cluster, setting up an Istio service mesh and install python based web application with Redis cache.

## Prerequisites 

* **Docker**

Documentation -> https://docs.docker.com/engine/install/debian/

Add Docker's official GPG key:

```
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository to Apt sources:
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

Install the Docker packages.


To install the latest version, run:
`sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`


* **Minikube**

Download minikube binary:
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```
Start local minikube. For Istio components to be wokring properly we will need more resources than the default Minikube is starting with:
`minikube start --cpus 6 --memory 8192`


* **kubectl**
* 
Install binary with curl on Linux:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

* **Helm**

Install Helm:
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm
```

Add Helm repo for Istio:
```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

# Istio setup

Create namespace and install all Istio components there:

```
kubectl create ns istio-system
helm install istio-base istio/base -n istio-system --set defaultRevision=default
helm install istiod istio/istiod -n istio-system
helm install istio-ingress istio/gateway -n istio-system
```

To enable a load balancer for use by Istio, in a separate terminal run:  `minikube tunnel`


# Install python test app

Create a namespace:
`kubectl create ns test-app`

Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:
`kubectl label ns test-app istio-injection=enabled`
Install the Helm chart: 
```
helm repo add python-app-chart https://mostafaalnaggar3.github.io/Helm-Task/
helm repo update
helm install my-python-app python-app-chart/python-app --version 0.1.0
```

Install Gateway and Virtual service for Istio:
`kubectl -n istio-system apply -f istio-vs.yaml`

# Browse the two apps:

Check your Istio Ingress external IP:
`kubectl ns istio-system get svc`

Curl the istio-ingress service IP in console with the different endpoints `/python-app/` and `/redis-app/` and it should display different output based on the appliaction which is being called.
`curl http://YOUR-IP/ENDPOIN`

# Cleanup

Sometimes minikube does not clean up the tunnel network properly. To force a proper cleanup:

`minikube tunnel --cleanup`

Remove the test-app and Istio components

```
helm -n test-app delete my-python-app
kubectl ns istio-system delete -f istio-vs.yaml
helm -n istio-system delete istio-base
helm -n istio-system delete istiod
helm -n istio-system delete istio-ingress
kubectl delete ns test-app
kubectl delete ns istio-system
```

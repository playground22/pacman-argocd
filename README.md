# Pacman app deploy with Arogo CD

In this repo we will create a kubernetes cluster with minikube and deploy an app in the cluster. 
The app, is the pacman game. 
We will deploy the app using Argo CD. 

# Deploy pacman with Argo CD

First things first! To deploy the app and also Argo CD we need a cluster. For this, we will use minikube. 

1 - Create a cluster wirh minikube
$ minikube start --extra-config=kubelet.housekeeping-interval=10s -p pacman 
Note: 
"The -extra-config=kubelet.housekeeping-interval=10s switch enables showing the CPU utilization metrics in the dashboard" 
https://adamtheautomator.com/minikube-dashboard/ 
:: When the process finishs, to confirm the cluster is running, the following command can be used: 
$ kubectl get nodes 

2 - Enable metrics addon: 
$ minikube addons enable metrics-server -p pacman 
Note: the "metrics-server" will be used to allow Horizontal Pod Autoscaling (HPA)

3 - Enable ingress controller, "ingress", addon: 
$ minikube addons enable ingress -p pacman 
Note: This is the ingress controller, that we will use to consume our ingress rules.

4 - List all addons and confirm metrics-server and ingress are enable 
$ minikube addons list -p pacman 

5 - Create a namespace dedicated to Argo CD, the name for the namespace is "argcd": 
$ kubectl apply -f argocd-deploy/01-argocd-namespace.yaml 
:: Confirm the namespace is created: 
$ kubectl get namespaces 

6 - Install Argo CD: 
$ kubectl apply -f argocd-deploy/02-argocd-install.yaml -n argocd-namespace 
:: Confirm the deployment is created: 
$ kubectl get deployments -n argocd-namespace -o wide 
:: Confirm the pods from the deployment have been created: 
$ kubectl get pods -n argocd-namespace -o wide 
Note: more "kubectl get" commands can be tested, because this installation has a lot of resources. Examples: 
$ kubectl get services -n argocd-namespace -o wide 
$ kubectl get clusterrole -n argocd-namespace -o wide 
$ kubectl get clusterrolebinding -n argocd-namespace -o wide 
$ (...) 

Note: This manifest is copied from https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
So, it is the same to issue the install command as: kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd 

7 - Get the password generated to login: 
:: List all the secrets: 
$ kubectl get secrets -n argocd-namespace -o wide 
:: Get the value for the "argocd-initial-admin-secret" secrets 
$ kubectl get secret argocd-initial-admin-secret -n argocd-namespace -o yaml 
:: Since the field "password" is base64 coded, we need to decode it. To do so, use the following command: 
echo -n "PASTE-HERE-THE-PASSWORD-VALUE" | base64 -d 

8 - Acess the Argo CD UI: 
$ kubectl port-forward service/argocd-server -n argocd-namespace 8080:443 
:: Now, we can go to the browser an access "https://localhost:8080" (ignore the certificate error) and login into argo cd as "admin" and the passowrd from the step above. 

9 - Deploy the app via Argo CD! In the folder images, we can see a picture of how to fill the "New App" form. 
The following URL should also help to understand what is happening here: 
https://codefresh.io/blog/getting-started-with-gitops-and-argo-cd/ 


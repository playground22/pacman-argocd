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
:: When the process finishes, to confirm the cluster is running, the following command can be used: 
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
$ kubectl apply -f argocd-deploy/02-argocd-install.yaml -n argocd 
:: Confirm the deployment is created: 
$ kubectl get deployments -n argocd -o wide 
:: Confirm the pods from the deployment have been created: 
$ kubectl get pods -n argocd -o wide 
Note: more "kubectl get" commands can be tested, because this installation has a lot of resources. Examples: 
$ kubectl get services -n argocd -o wide 
$ kubectl get clusterrole -n argocd -o wide 
$ kubectl get clusterrolebinding -n argocd -o wide 
$ (...) 

Note: This manifest is copied from https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
So, it is the same to issue the install command as: kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd 

7 - Get the password generated to login: 
:: List all the secrets: 
$ kubectl get secrets -n argocd -o wide 
:: Get the value for the "argocd-initial-admin-secret" secrets 
$ kubectl get secret argocd-initial-admin-secret -n argocd -o yaml 
:: Since the field "password" is base64 coded, we need to decode it. To do so, use the following command: 
echo -n "PASTE-HERE-THE-PASSWORD-VALUE" | base64 -d 


8 - Acess the Argo CD UI: 
$ kubectl port-forward service/argocd-server -n argocd 8080:443 
:: Now, we can go to the browser an access "https://localhost:8080" (ignore the certificate error) and login into argo cd as "admin" and the passowrd from the step above. 

9 - Deploy the app via Argo CD! In the folder "pics", we can see several pictures, since deploying the app to configurations.
The following URL should also help to understand how to deploy an app: 
https://codefresh.io/blog/getting-started-with-gitops-and-argo-cd/ 

Nevertheless! Here's a description of what we filled in the form: 
GENERAL: 
Application name: pacman 
Project name: default 

SYNC POLICY: 
Automatic 
(selected) PRUNE RESOURCES 
(selected) SELF HEAL 
(selected) AUTO-CREATE NAMESPACE 

Notes: In this block we are saying that, the app will sync with commits to the git repository, delete "things" that are note in the repository and force the state of the app to what is defined in the repository. 
Also, the app will be in a namespace managed by Argo CD, that will be automatically created in the resources. 

SOURCE 
Repository URL: "Your repository URL" 
Revision: HEAD 
Path: Location (folder) of the yaml files with the specs of the resources used by the app 

Notes: You need to guarantee that the repository is public, or, in case of private, assign credentials for Argo CD to authenticate (Argo CD Settings > Repositories) 

DESTINATION 
Cluster URL: https://kubenetes.defautl.svc 
Namespace: pacman-argo 

Notes: When deploying internally (i.e. to the same cluster that Argo CD is running in), the https://kubernetes.default.svc hostname should be used as the application’s Kubernetes API server address. 
The namespace, is the name for the namespace that Argo CD will create automatically (as stated in the steps above). 

After this, just press the button "Create" and the app will be automatically deployed to the cluster. 

10 - After having the app running and healthy, we can export it configuration to a yaml manifest, and then, treat it as any other manifest and deploy it!
using kubectl apply -f ARGOCD-APP-FILE.yaml. 

To do so, use the command: 
kubectl get application APLICATION-NAME -o yaml -n argocd > app.yaml 
Example, according to the app we've deployed: 
$ kubectl get application pacman -o yaml -n argocd > app.yaml 
Then, just remove the unnecessary lines (take a look at the file i the repository root "argocd-exported-app.yaml"). 

To test, just delete the app in the UI (remember that all the resources will be deleted) and then: 
$ kubectl apply -f argocd-exported-app.yaml 

Wit this, the app will be deployed again, and will have the same configurations as the ones done manually! 
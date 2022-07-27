# Pacman app deploy with Arogo CD

In this repo we will create a kubernetes cluster with minikube and deploy an app in the cluster.<br />
The app, is the pacman game.<br />
We will deploy the app using Argo CD.<br />

# Deploy pacman with Argo CD

First things first! To deploy the app and also Argo CD we need a cluster. For this, we will use minikube.<br />

1 - Create a cluster with minikube<br />
$ minikube start --extra-config=kubelet.housekeeping-interval=10s -p pacman<br /> 
Note:<br />
"The -extra-config=kubelet.housekeeping-interval=10s switch enables showing the CPU utilization metrics in the dashboard"<br />
https://adamtheautomator.com/minikube-dashboard/<br />
:: When the process finishes, to confirm the cluster is running, the following command can be used:<br /> 
$ kubectl get nodes<br />

2 - Enable metrics addon:<br />
$ minikube addons enable metrics-server -p pacman<br />
Note: the "metrics-server" will be used to allow Horizontal Pod Autoscaling (HPA)<br />

3 - Enable ingress controller, "ingress", addon:<br />
$ minikube addons enable ingress -p pacman<br />
Note: This is the ingress controller, that we will use to consume our ingress rules.<br />

4 - List all addons and confirm metrics-server and ingress are enable<br />
$ minikube addons list -p pacman<br />

5 - Create a namespace dedicated to Argo CD, the name for the namespace is "argcd":<br />
$ kubectl apply -f argocd-deploy/01-argocd-namespace.yaml<br />
:: Confirm the namespace is created:<br />
$ kubectl get namespaces<br />

6 - Install Argo CD:<br />
$ kubectl apply -f argocd-deploy/02-argocd-install.yaml -n argocd<br />
:: Confirm the deployment is created:<br />
$ kubectl get deployments -n argocd -o wide<br /> 
:: Confirm the pods from the deployment have been created:<br />
$ kubectl get pods -n argocd -o wide<br />
Note: more "kubectl get" commands can be tested, because this installation has a lot of resources. Examples:<br />
$ kubectl get services -n argocd -o wide<br />
$ kubectl get clusterrole -n argocd -o wide<br />
$ kubectl get clusterrolebinding -n argocd -o wide<br />
$ (...)<br />

Note: This manifest is copied from https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml<br />
So, it is the same to issue the install command as: kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd<br />

7 - Get the password generated to login:<br />
:: List all the secrets:<br />
$ kubectl get secrets -n argocd -o wide<br />
:: Get the value for the "argocd-initial-admin-secret" secrets<br />
$ kubectl get secret argocd-initial-admin-secret -n argocd -o yaml<br />
:: Since the field "password" is base64 coded, we need to decode it. To do so, use the following command:<br />
echo -n "PASTE-HERE-THE-PASSWORD-VALUE" | base64 -d<br />


8 - Acess the Argo CD UI:<br />
$ kubectl port-forward service/argocd-server -n argocd 8080:443<br />
:: Now, we can go to the browser an access "https://localhost:8080" (ignore the certificate error) and login into argo cd as "admin" and the passowrd from the step above.<br />

9 - Deploy the app via Argo CD! In the folder "pics", we can see several pictures, since deploying the app to configurations.<br />
The following URL should also help to understand how to deploy an app:<br />
https://codefresh.io/blog/getting-started-with-gitops-and-argo-cd/<br />

Nevertheless! Here's a description of what we filled in the form:<br />
GENERAL:<br />
Application name: pacman<br />
Project name: default<br />

SYNC POLICY:<br />
Automatic<br />
(selected) PRUNE RESOURCES<br />
(selected) SELF HEAL<br />
(selected) AUTO-CREATE NAMESPACE<br />

Notes: In this block we are saying that, the app will sync with commits to the git repository, delete "things" that are note in the repository and force the state of the app to what is defined in the repository.<br />
Also, the app will be in a namespace managed by Argo CD, that will be automatically created in the resources.<br />

SOURCE<br />
Repository URL: "Your repository URL"<br />
Revision: HEAD<br />
Path: Location (folder) of the yaml files with the specs of the resources used by the app<br />

Notes: You need to guarantee that the repository is public, or, in case of private, assign credentials for Argo CD to authenticate (Argo CD Settings > Repositories)<br />

DESTINATION<br />
Cluster URL: https://kubenetes.defautl.svc<br />
Namespace: pacman-argo<br />

Notes: When deploying internally (i.e. to the same cluster that Argo CD is running in), the https://kubernetes.default.svc hostname should be used as the application’s Kubernetes API server address.<br />
The namespace, is the name for the namespace that Argo CD will create automatically (as stated in the steps above).<br />

After this, just press the button "Create" and the app will be automatically deployed to the cluster.<br />

10 - After having the app running and healthy, we can export it configuration to a yaml manifest, and then, treat it as any other manifest and deploy it!<br />
using kubectl apply -f ARGOCD-APP-FILE.yaml.<br />

To do so, use the command:<br />
kubectl get application APLICATION-NAME -o yaml -n argocd > app.yaml<br />
Example, according to the app we've deployed:<br />
$ kubectl get application pacman -o yaml -n argocd > app.yaml<br />
Then, just remove the unnecessary lines (take a look at the file i the repository root "argocd-exported-app.yaml").<br />

To test, just delete the app in the UI (remember that all the resources will be deleted) and then:<br />
$ kubectl apply -f argocd-exported-app.yaml<br />

With this, the app will be deployed again, and will have the same configurations as the ones done manually!<br />
# pacman-argocd

In this repo we will create a kubernetes cluster with minikube and deploy an app in the cluster.
The first part, "Deploy pacman app manually", show the deployment of the app amnually.
The second part, "Deploy pacman app with argocd", show how to deploy the app with arogcd.

# Deploy pacman app manually

1 - Create minikube cluster with name pacman and with flag to monitorize CPU utilization
$ minikube start --extra-config=kubelet.housekeeping-interval=10s -p pacman
Note:
"The -extra-config=kubelet.housekeeping-interval=10s switch enables showing the CPU utilization metrics in the dashboard"
https://adamtheautomator.com/minikube-dashboard/
:: When the process finishs, to confirm the cluster is running, the following command can be used:
$ kubectl get nodes

2 - Enable metrics addon:
$ minikube addons enable metrics-server -p pacman

3 - Enable ingress controller, "ingress", addon:
$ minikube addons enable ingress -p pacman

4 - List all addons and confirm metrics-server and ingress are enable
$ minikube addons list -p pacman

# Deploy pacman app

5 - Create a namespace dedicated to pacman app:
$ kubectl apply -f pacman-app-deploy/01-pacman-namespace.yaml
:: Confirm the namespace is created:
$ kubectl get namespaces


6 - Create a deployment with pacman app:
$ kubectl apply -f pacman-app-deploy/02-pacman-deployment.yaml
:: Confirm the deployment is created:
$ kubectl get deployments -n pacman-namespace -o wide
:: Confirm the pods from the deployment have been created:
$ kubectl get pods -n pacman-namespace -o wide

7 - Create a service type ClusterIP for the pods from the deployment:
$ kubectl apply -f pacman-app-deploy/03-pacman-clusterip-service.yaml
:: Confirm the service is created:
$ kubectl get services -n pacman-namespace -o wide

8 - Create an ingress (that will be consumed by the minikune ingress controller, "ingress") to the service we've just created:
$ kubectl apply -f pacman-app-deploy/04-pacman-ingress.yaml
:: Confirm the ingress is created (it takes about a minute to display the ADDRESS of the ingress):
$ kubectl get ingress -n pacman-namespace -o wide
:: Now, it should be possible to access the app in the browser. Remmeber to add to your host file the IP from the ingress along with the name "pacman.local".

9 - Create a Horizontal Pod Autoscaling (HPA) for the pacman app (it takes about a minute to have the HPA enabled):
$ kubectl apply -f pacman-app-deploy/05-horizontal-pod-autoscaler.yaml
:: Confirm the ingress is created:
$ kubectl get hpa -n pacman-namespace -o wide

10 - To test that the HPA is working, run the following command:
ab -n 1000000 -c 10 -H "Accept-Encoding: gzip, deflate" -rk http://pacman.local/
Note: This was tested in an Ubuntu. Also, this comand is from "Apache Bench". 
To install. please check:
https://diamantidis.github.io/2020/07/15/load-testing-with-apache-bench
To learn more about Apache Bench, please check:
https://httpd.apache.org/docs/2.4/programs/ab.html

:: To check the load on the hpa, open a new terminal window and type:
$ watch kubectl get hpa -n pacman-namespace -o wide
:: To check pods being created, open a new terminal window and type:
$ watch kubectl get pods -n pacman-namespace -o wide

:: When the test finishes, it could take a while (lets say, about 5 minutes) until the "extra" pods to be deleted, but, theu will be!


# Deploy pacman app with argocd
1 - Delete the previous cluster:
$ minikube delete -p pacman
:: If no other cluster is running, you can use the command:
$ minikube delete --all

2 - Create a new cluster
$ minikube start --extra-config=kubelet.housekeeping-interval=10s -p pacman

3 - Enable metrics addon:
$ minikube addons enable metrics-server -p pacman

4 - Enable ingress controller, "ingress", addon:
$ minikube addons enable ingress -p pacman

5 - List all addons and confirm metrics-server and ingress are enable
$ minikube addons list -p pacman

6 - Create a namespace dedicated to argocd:
$ kubectl apply -f argocd-deploy/01-argocd-namespace.yaml
:: Confirm the namespace is created:
$ kubectl get namespaces

7 - Install argocd:
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

Note 1: This manifest is copied from https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
So, it is the same to issue the install command as: kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd | BUT PLEASE READ NOTE 2!!!
Note 2: Since the namespace is not the "default" (argocd) as in the documentation, in the 02-argocd-install.yaml, in the "kind: ClusterRoleBinding" blocks, the namespace was updated to "namespace: argocd-namespace".
So, if you are installing argocd from the URL, change the namespace name in the "pacman-app-argocd-deploy/01-argocd-namespace.yaml" file.

8 - Get the password generated to login:
:: List all the secrets:
$ kubectl get secrets -n argocd-namespace -o wide
:: Get the value for the "argocd-initial-admin-secret" secrets
$ kubectl get secret argocd-initial-admin-secret -n argocd-namespace -o yaml
:: Since the field "password" is base64 coded, we need to decode it. To do so, use the following command:
echo -n "PASTE-HERE-THE-PASSWORD-VALUE" | base64 -d
N3M1eGFEVEtJUVdHRFItNg==
echo -n "N3M1eGFEVEtJUVdHRFItNg==" | base64 -d

7s5xaDTKIQWGDR-6

9 - Acess the argocd UI:
$ kubectl port-forward service/argocd-server -n argocd-namespace 8080:443
:: Now, we can go to the browser an access "https://localhost:8080" (ignore the certificate error) and login into argo cd as "admin" and the passowrd from the step above.

10 - Deploy the app via argocd! In the folder images, we can see a picture of how to fill the "New App" form.
The following URL should also help to understand what is happening here:
https://codefresh.io/blog/getting-started-with-gitops-and-argo-cd/


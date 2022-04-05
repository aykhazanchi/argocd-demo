# Demo Declarative Continuous Deployment on K8s using ArgoCD

## About
In imperative deployment there is often someone "pushing a button" to start a deployment. The final environment is the result of a number of steps as defined by the deployment scripts. With Declarative deployment this concept is shifted to defining the environment as a "state". As soon as this state is modified, declarative tools attempt to do deployment by "syncing" towards the new state. ArgoCD is a declarative GitOps tool that is used for continuous deployment to Kubernetes. ArgoCD considers a git repository as its source of truth and attempts to do automatic deployments on K8s whenever it observes a change in the git repo.

In this demo, I attempt to deploy a tiny nginx static application on minikube using ArgoCD. I then make a change to the docker image being used in the application and push it to the repo and demo the automatic sync capability of ArgoCD (declarative deployment). On seeing the change in the Github repo, ArgoCD should automatically update the application.

## Prerequisites
This guide assumes an existing setup of 
- Docker
- Minikube
- Familiarity with kubectl commands

For setting up K8s and understanding the basics, see [here](https://kubernetes.io/docs/tutorials/)
## Assumptions
- This guide mostly replicates the installation of ArgoCD from their docs. ArgoCD is a fast-moving project so it is always advised to refer to their official docs in case something with the installation here fails.

## Setup
- Start minikube

    ```
    minikube start
    ```

- Create ArgoCD namespace in minikube. We do this to keep our local cluster a little bit more organized.

    ```
    kubectl create namespace argocd
    ```

- Install ArgoCD

    ```
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

- Wait for the pods to come to `Running` state. You can check this by running 

    ```
    kubectl get pods -n argocd
    ```
    
- Once the pods are running, change ArgoCD service to use LoadBalancer. This makes it easier to access through our local minikube cluster. (Note: you can also port-forwarding instead)

    ```
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
    ```

- Get ArgoCD initial secret and update to use your own unique password

    ```
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    
    argocd account update-password
    ```

- In another terminal, start a minikube tunnel so you can access ArgoCD Admin Server. You will be prompted to enter your sudo password for ArgoCD to be able to access port 80 & 443. You need to keep this running in another terminal to keep the tunnel alive

    ```
    minikube tunnel
    ```

- Get the exposed URL of ArgoCD from minikube.

    ```
    minikube service —url argocd-server -n argocd
    ```

    This should give you something of the form `http://<some-ip>:<some-port>`. You can now use this in your browser to access ArgoCD using admin credentials.

### Deployment via UI (CLI steps follow below)
- Login to ArgoCD admin dashboard with the credentials you set above
- Click on New App
<img width="306" alt="Screenshot 2022-04-05 at 16 04 55" src="https://user-images.githubusercontent.com/5686467/161775956-a727b956-69fd-4456-a244-e4dd603db116.png">
- Under *General*, enter the app details as below
<img width="299" alt="Screenshot 2022-04-05 at 16 05 16" src="https://user-images.githubusercontent.com/5686467/161776112-f7f1ee22-ce4e-45e9-9b88-6ec37db4668f.png">
- Under *Source*, enter the git repo details as below
<img width="443" alt="Screenshot 2022-04-05 at 16 05 46" src="https://user-images.githubusercontent.com/5686467/161776190-223e641e-6c3e-43d2-921c-858bbf47a1a8.png">
- Under *Destination*, we deploy to the cluster url as our ArgoCD installation sits in the same cluster as our app
<img width="334" alt="Screenshot 2022-04-05 at 16 05 59" src="https://user-images.githubusercontent.com/5686467/161776233-4bfa6789-849e-498c-b6ee-b6d0e3f57c76.png">
- Click Create
- You should see your app created as shown below
- Click on the app and you should see that the current sync status is healthy (note: it may take some time to show as the pods can still be coming up in the background)
- When your app is deployed all monitors on the various pieces will turn into green hearts. Your app is up!
- Get the app url from minikube
    ```
    minikube service --url webserver -n default
    ```
- You should now see our static nginx app in the browser using the url
- You can skip now to the `Automated Sync via ArgoCD` section to proceed further

### Deployment via ArgoCD CLI
- For deploying via CLI, we need to login via ArgoCD cli. If you don't already have ArgoCD CLI installed, it is possible to install it via the brew package manager on Mac.

    ```
    brew install argocd

    # use the url you got above
    argocd login http://<some-ip>:<some-port> 
    ```

- Create the example nginx webserver we have in this repo as an ArgoCD app

    ```
    argocd app create webserver --repo https://github.com/aykhazanchi/argocd-demo.git --path . --dest-server https://kubernetes.default.svc --dest-namespace default
    ```

- Set the sync policy for our app to allow automated sync. With automatic sync, ArgoCD watches the Git repo and polls every 3 minutes to see if state has changed.

    ```
    argocd app set webserver --sync-policy automated --auto-prune --self-heal --allow-empty
    ```

- Sync (deploy) the application

    ```
    argocd app sync webserver
    ```

- Get the app url from minikube

    ```
    minikube service --url webserver -n default
    ```

- You should now see our static nginx app in the browser using the url

### Automated Sync via ArgoCD
- At this point we have our `webserver` app synced via ArgoCD. If you look at the minikube cluster you'll see that there is now a service and a deployment running for the app.

    ```
    kubectl get svc -n default
    kubectl get pods -n default
    ```

- Now, to see the automatatic sync functionality in action, we make a small change in our git repo and push it. We will see the changes are automatically applied via ArgoCD when it next polls the git repo in intervals of 3 minutes.

- We will change the docker image being used. In `deployment.yaml` update the image tag to use `v1.1`. The new image should look like below -

    ```
    containers:
    - image: docker.io/aykhazanchi/argocd-demo-webserver:v1.1
    ```

- Commit the change and push to Github.

- After some time, you will see a new pod created and brought to `running` state and the old pod is destroyed. If you run a `describe` on the new deployment you will see that it's now using our new image.

    ```
    kubectl describe deployment webserver
    
    ### Truncated output below
    Containers:
      webserver:
        Image:        docker.io/aykhazanchi/argocd-demo-webserver:v1.1
    ```

- Similarly, other changes can be performed such as increasing number of replicas in the pods or adding a new feature to the app. All changes will be monitored and ArgoCD will update the environment accordingly.

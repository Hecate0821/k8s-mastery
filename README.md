# Build Images of  webapp and logic

First thing we should do is to build images for all the three services

## Logic

Just run the cmd

```bash
docker build -f Dockerfile -t Hecat0821/sentiment-analysis-logic . `
```

## WebApp

To reduce the time to build and the size of the image, we need to create `.dockerignore` , to keep source code out of building process.

```bash
src
```



Build cmd

```bash
docker build -f Dockerfile -t Hecat0821/sentiment-analysis-webapp . `
```



# Modify  Yaml Files

You have to modify the deployment and service yaml, to meke them pull the images you built and push to your own docker hub instead of the author's.

Here is an example:

``` yaml
    spec:
      containers:
        - image: hecate0821/sentiment-analysis-logic:latest # your own docker hub images
          imagePullPolicy: Always
          name: sa-logic
          ports:
            - containerPort: 5000
```

# Deployment

Use Google Cloud CLI to deploy your services.

## 1. Set your Project and Zone

```
gcloud config set project [YOUR_PROJECT_ID]
gcloud config set compute/zone [YOUR_COMPUTE_ZONE]
```

## 2. Create Cluster

There are actually two way to create a cluster on GCP.

### GUI

Just click Create, you will get a user friendly auto-resize cluster.



### Command Line

You can also create your customize cluster. Here is an example:

``` cmd
gcloud beta container --project "cloud-infra-435418" clusters create-auto "autopilot-cluster-1" --region "us-central1" --release-channel "regular" --network "projects/cloud-infra-435418/global/networks/default" --subnetwork "projects/cloud-infra-435418/regions/us-central1/subnetworks/default" --cluster-ipv4-cidr "/17" --binauthz-evaluation-mode=DISABLED
```



## 3. Get Cluster Credential

You have to get authenticated to use kubectl.

> note: you may need to install some plugin during this phase

```cmd
gcloud container clusters get-credentials [CLUSTER_NAME]
```

#### 4.Use Local  Yaml

```cmd
# deployment
kubectl apply -f resource-manifests\sa-logic-deployment.yaml
kubectl apply -f resource-manifests\sa-web-app-deployment.yaml

# service
kubectl apply -f resource-manifests\service-sa-logic.yaml
kubectl apply -f resource-manifests\service-sa-web-app-lb.yaml
```

#### 5. Check Your Deployment and Service 

```
kubectl get deployments
kubectl get services
```

It should be like this:

``` cmd
C:\Program Files (x86)\Google\Cloud SDK>kubectl get service
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
kubernetes       ClusterIP      34.118.224.1     <none>          443/TCP          124m
sa-frontend-lb   LoadBalancer   34.118.233.238   34.31.104.12    80:31378/TCP     7m40s
sa-logic         ClusterIP      34.118.229.130   <none>          80/TCP           87m
sa-web-app-lb    LoadBalancer   34.118.226.170   34.66.195.222   8080:32059/TCP   62m

C:\Program Files (x86)\Google\Cloud SDK>kubectl get deployment
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
sa-frontend   2/2     2            2           7m49s
sa-logic      2/2     2            2           79m
sa-web-app    2/2     2            2           62m
```



# Modify and Build Frontend Service

## Check External IP of Webapp

You can access your webapp from frontend using external ip. Here is the cmd to check ip:

```
kubectl get services
```

You will get:

```
sa-web-app-lb    LoadBalancer   34.118.226.170   34.66.195.222   8080:32059/TCP   62m
```



## Modify Frontend

Modify the address in `sa-frontend\src\App.js `

``` js
analyzeSentence() {
        fetch('http://34.66.195.222:8080/sentiment', { // The external IP adress
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({sentence: this.textField.getValue()})
        })
            .then(response => response.json())
            .then(data => this.setState(data));
    }
```

## Build

Rebuild JS Project

```
npm run build
```

Use .dockerignore

```
node_modules
src
public
```



Rebuild and Repush Docker

``` 
docker build -t hecate0821/sentiment-analysis-frontend:stable .
docker push hecate0821/sentiment-analysis-frontend:stable
```

## Deploy Frontend

Pretty much the same as webapp and logic.

```cmd
# deployment
kubectl apply -f resource-manifests\sa-frontend-deployment-green.yaml

# service
kubectl apply -f resource-manifests\service-sa-frontend-lb.yaml
```

# Test

Get the external IP of frontend service

```
sa-frontend-lb   LoadBalancer   34.118.233.238   34.31.104.12    80:31378/TCP     7m40s
```

Access `34.31.104.12:80`

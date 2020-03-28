# [DevOps] [GSP114] Continuous Delivery Pipelines with Spinnaker and Kubernetes Engine

Lab:  
https://www.qwiklabs.com/focuses/552?parent=catalog

Promo - 1 month free:  
https://go.qwiklabs.com/qwiklabs-free


<br/>


<br/>

![Application](/img/pic1.png?raw=true)


**Application delivery pipeline**

<br/>

![Application](/img/pic2.png?raw=true)


<br/>

### Set up your environment

    $ gcloud config set compute/zone us-central1-f

    $ gcloud container clusters create spinnaker-tutorial \
    --machine-type=n1-standard-2


    // Create the service account:

    $ gcloud iam service-accounts create spinnaker-account \
        --display-name spinnaker-account

    // Store the service account email address and your current project ID in environment variables for use in later commands:

    $ export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-account" \
    --format='value(email)')

    $ export PROJECT=$(gcloud info --format='value(config.project)')

    // Bind the storage.admin role to your service account:
    $ gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin \
    --member serviceAccount:$SA_EMAIL

    // Download the service account key. In a later step, you will install Spinnaker and upload this key to Kubernetes Engine:

    $ gcloud iam service-accounts keys create spinnaker-sa.json \
     --iam-account $SA_EMAIL

<br/>

### Set up Cloud Pub/Sub to trigger Spinnaker pipelines

    // Create the Cloud Pub/Sub topic for notifications from Container Registry.
    $ gcloud pubsub topics create projects/$PROJECT/topics/gcr

    // Create a subscription that Spinnaker can read from to receive notifications of images being pushed.
    $ gcloud pubsub subscriptions create gcr-triggers \
    --topic projects/${PROJECT}/topics/gcr

    // Give Spinnaker's service account permissions to read from the gcr-triggers subscription.

    $ export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-account" \
    --format='value(email)')

    $ gcloud beta pubsub subscriptions add-iam-policy-binding gcr-triggers \
    --role roles/pubsub.subscriber --member serviceAccount:$SA_EMAIL

<br/>

### Deploying Spinnaker using Helm

    $ wget https://get.helm.sh/helm-v3.1.0-linux-amd64.tar.gz

    $ tar zxfv helm-v3.1.0-linux-amd64.tar.gz

    $ cp linux-amd64/helm .

    // Grant Helm the cluster-admin role in your cluster:
    $ kubectl create clusterrolebinding user-admin-binding \
    --clusterrole=cluster-admin --user=$(gcloud config get-value account)

    // Grant Spinnaker the cluster-admin role so it can deploy resources across all namespaces:
    $ kubectl create clusterrolebinding --clusterrole=cluster-admin \
    --serviceaccount=default:default spinnaker-admin

    // Add the stable charts deployments to Helm's usable repositories (includes Spinnaker):

    $ ./helm repo add stable https://kubernetes-charts.storage.googleapis.com

    $ ./helm repo update

<br/>

### Configure Spinnaker

    // Still in Cloud Shell, create a bucket for Spinnaker to store its pipeline configuration:
    $ export PROJECT=$(gcloud info \
    --format='value(config.project)')

    $ export BUCKET=$PROJECT-spinnaker-config

    $ gsutil mb -c regional -l us-central1 gs://$BUCKET

    // Run the following command to create a spinnaker-config.yaml file, which describes how Helm should install Spinnaker:

    $ export SA_JSON=$(cat spinnaker-sa.json)
    $ export PROJECT=$(gcloud info --format='value(config.project)')

    $ export BUCKET=$PROJECT-spinnaker-config

```
$ cat > spinnaker-config.yaml <<EOF
gcs:
  enabled: true
  bucket: $BUCKET
  project: $PROJECT
  jsonKey: '$SA_JSON'

dockerRegistries:
- name: gcr
  address: https://gcr.io
  username: _json_key
  password: '$SA_JSON'
  email: 1234@5678.com

# Disable minio as the default storage backend
minio:
  enabled: false

# Configure Spinnaker to enable GCP services
halyard:
  additionalScripts:
    create: true
    data:
      enable_gcs_artifacts.sh: |-
        \$HAL_COMMAND config artifact gcs account add gcs-$PROJECT --json-path /opt/gcs/key.json
        \$HAL_COMMAND config artifact gcs enable
      enable_pubsub_triggers.sh: |-
        \$HAL_COMMAND config pubsub google enable
        \$HAL_COMMAND config pubsub google subscription add gcr-triggers \
          --subscription-name gcr-triggers \
          --json-path /opt/gcs/key.json \
          --project $PROJECT \
          --message-format GCR
EOF
```

// Deploy the Spinnaker chart

The installation typically takes 5-8 minutes to complete.

    $ ./helm install -n default cd stable/spinnaker -f spinnaker-config.yaml \
    --version 1.23.0 --timeout 10m0s --wait


    // Set up port forwarding to Spinnaker from Cloud Shell:
    $ export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" \
    -o jsonpath="{.items[0].metadata.name}")

    $ kubectl port-forward --namespace default $DECK_POD 8080:9000 >> /dev/null &

<br/>

Preview on port 8080.

<br/>

![Application](/img/pic3.png?raw=true)



<br/>

### Building the Docker image


    $ wget https://gke-spinnaker.storage.googleapis.com/sample-app-v2.tgz

    $ tar xzfv sample-app-v2.tgz

    $ cd sample-app

    $ git config --global user.email "$(gcloud config get-value core/account)"

    $ git config --global user.name "marley"

    $ git init
    $ git add .
    $ git commit -m "Initial commit"

    // Create a repository to host your code:
    $ gcloud source repos create sample-app

    $ git config credential.helper gcloud.sh

    $ export PROJECT=$(gcloud info --format='value(config.project)')

    $ git remote add origin https://source.developers.google.com/p/$PROJECT/r/sample-app

    $ git push origin master


    Navigation Menu --> Source Repositories --> View All Repositories and select sample-app.

<br/>

![Application](/img/pic4.png?raw=true)

<br/>

### Configure your build triggers

Navigation menu --> Cloud Build --> Triggers --> Create trigger.


<br/>

![Application](/img/pic5.png?raw=true)



<br/>

### Prepare your Kubernetes Manifests for use in Spinnaker


    // Create the bucket:
    $ export PROJECT=$(gcloud info --format='value(config.project)')

    $ gsutil mb -l us-central1 gs://$PROJECT-kubernetes-manifests

    // Enable versioning on the bucket so that you have a history of your manifests:
    $ gsutil versioning set on gs://$PROJECT-kubernetes-manifests


    // Set the correct project ID in your kubernetes deployment manifests:
    $ sed -i s/PROJECT/$PROJECT/g k8s/deployments/*

    // Commit the changes to the repository:
    $ git commit -a -m "Set project ID"

<br/>

### Build your image

    $ git tag v1.0.0
    $ git push --tags

<br/>

![Application](/img/pic6.png?raw=true)    

<br/>

### Configuring your deployment pipelines

Now that your images are building automatically, you need to deploy them to the Kubernetes cluster.


Install the spin CLI for managing Spinnaker

    $ curl -LO https://storage.googleapis.com/spinnaker-artifacts/spin/1.14.0/linux/amd64/spin

    $ chmod +x spin

<br/>

**Create the deployment pipeline**

Use spin to create an app called sample in Spinnaker. Set the owner email address for the app in Spinnaker:

    $ ./spin application save --application-name sample \
    --owner-email "$(gcloud config get-value core/account)" \
    --cloud-providers kubernetes \
    --gate-endpoint http://localhost:8080/gate

Next, you create the continuous delivery pipeline. In this tutorial, the pipeline is configured to detect when a Docker image with a tag prefixed with "v" has arrived in your Container Registry.


    $ export PROJECT=$(gcloud info --format='value(config.project)')

    $ sed s/PROJECT/$PROJECT/g spinnaker/pipeline-deploy.json > pipeline.json

    $ ./spin pipeline save --gate-endpoint http://localhost:8080/gate -f pipeline.json


<br/>

### Manually Trigger and View your pipeline execution

Spinnaker UI  --> Applications 

<br/>

![Application](/img/pic7.png?raw=true)

sample --> Pipelines --> Start Manual Execution --> Run


<br/>

![Application](/img/pic8.png?raw=true)

After 3 to 5 minutes the integration test phase completes and the pipeline requires manual approval to continue the deployment.


Hover over the yellow "person" icon and click Continue.


Infrastructure > Load Balancers

Scroll down the list of load balancers and click Default, under service sample-frontend-production.

<br/>

![Application](/img/pic9.png?raw=true)

Ctrl + F5

<br/>

![Application](/img/pic10.png?raw=true)

<br/>

![Application](/img/pic11.png?raw=true)


<br/>

### Triggering your pipeline from code changes

    $ sed -i 's/orange/blue/g' cmd/gke-info/common-service.go

    $ git commit -a -m "Change color to blue"

    $ git tag v1.0.1

    $ git push --tags

<br/>

![Application](/img/pic12.png?raw=true)

<br/>

![Application](/img/pic13.png?raw=true)


<br/>

![Application](/img/pic14.png?raw=true)

<br/>

### Observe the canary deployments


    $ git revert v1.0.1
    $ git tag v1.0.2
    $ git push --tags


<br/>

![Application](/img/pic15.png?raw=true)

<br/>

![Application](/img/pic16.png?raw=true)

---


<a href="https://marley.org"><strong>Marley</strong></a>

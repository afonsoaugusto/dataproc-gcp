## Lab 5 : Cluster automation using CLI commands

### Task 1: Preparation

```sh
export BUCKET=qwiklabs-gcp-9f1a8225d2a5d5f7
export MYREGION=us-central1
export MYZONE=us-central1-a
export PROJECT_ID=qwiklabs-gcp-9f1a8225d2a5d5f7
export BROWSER_IP=177.43.102.10

cd
cp -r /training/training-data-analyst .
ls
```

### Task 2: Customize the Dataproc initialization action

```sh
cd ~/training-data-analyst/courses/unstructured/
cat init-script.sh
```

```sh
#!/bin/bash

# install Google Python client on all nodes
apt-get update
apt-get install -y python-pip
pip install --upgrade google-api-python-client

ROLE=$(/usr/share/google/get_metadata_value attributes/dataproc-role)
if [[ "${ROLE}" == 'Master' ]]; then
   echo "Only on master node ..."
fi
```

```sh
#!/bin/bash

# install Google Python client on all nodes
apt-get update
apt-get install -y python-pip
pip install --upgrade google-api-python-client

ROLE=$(/usr/share/google/get_metadata_value attributes/dataproc-role)
if [[ "${ROLE}" == 'Master' ]]; then
   git clone https://github.com/GoogleCloudPlatform/training-data-analyst
fi
```

### Task 3: Create the Dataproc cluster

```sh
export BUCKET=qwiklabs-gcp-9f1a8225d2a5d5f7
export MYREGION=us-central1
export MYZONE=us-central1-a
export PROJECT_ID=qwiklabs-gcp-9f1a8225d2a5d5f7
echo $BUCKET $MYREGION $MYZONE
echo $PROJECT_ID
gsutil cp init-script.sh gs://$BUCKET
```

```sh
gcloud dataproc clusters create cluster-custom \
--bucket $BUCKET \
--subnet default \
--zone $MYZONE \
--region $MYREGION \
--master-machine-type n1-standard-2 \
--master-boot-disk-size 100 \
--num-workers 2 \
--worker-machine-type n1-standard-1 \
--worker-boot-disk-size 50 \
--num-preemptible-workers 2 \
--image-version 1.2 \
--scopes 'https://www.googleapis.com/auth/cloud-platform' \
--tags customaccess \
--project $PROJECT_ID \
--initialization-actions 'gs://qwiklabs-gcp-9f1a8225d2a5d5f7/init-script.sh','gs://dataproc-initialization-actions/datalab/datalab.sh'

gsutil cp gs://qwiklabs-gcp-9f1a8225d2a5d5f7/init-script.sh .
gsutil cp gs://dataproc-initialization-actions/datalab/datalab.sh .
gsutil cp  gs://qwiklabs-gcp-9f1a8225d2a5d5f7/google-cloud-dataproc-metainfo/a952d592-7beb-47ee-a7db-2c21ee47e700/cluster-custom2-m/dataproc-initialization-script-1_output .
```

### Task 4: Verify Cluster Customization

```sh
echo $BROWSER_IP
gcloud compute \
--project=$PROJECT_ID \
firewall-rules create allow-custom \
--direction=INGRESS \
--priority=1000 \
--network=default \
--action=ALLOW \
--rules=tcp:9870,tcp:8088,tcp:8080 \
--source-ranges=$BROWSER_IP/32 \
--target-tags=customaccess
```

## Module 3 Quiz

### Which of the following will you typically NOT use an initialization action script for?

* Copy over custom configuration files to the cluster
* Install software libraries on the master
* Install software libraries on the workers
* __Change the number of workers in the cluster__

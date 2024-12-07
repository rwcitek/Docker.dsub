# Notes

## Setup

In part, based on ...
- https://cloud.google.com/batch/docs/dsub


### create a VM in GCP
- https://console.cloud.google.com/compute/instancesAdd

Setup
- E2-micro
- Boot disk:
  - Image: Ubuntu 24.04 LTS88, x86/64, amd64
  - Type: standard persistent disk
- Access scopes:
  - Allow full access to all Cloud APIs

### connect to the VM using web SSH
- https://console.cloud.google.com/compute/instances


### update system packages
```bash
sudo apt-get update
sudo apt-get -y dist-upgrade
sudo reboot
```

### install docker
```bash
curl -fSL -o /tmp/get-docker.sh https://get.docker.com
sed -i -e 's/sleep 20/sleep 1/' /tmp/get-docker.sh
sudo sh /tmp/get-docker.sh
sudo usermod -aG docker ${USER}
exec sudo su - ${USER} 
```

### pull an image from Docker Hub
```bash
docker image list -a
docker pull ubuntu:24.04
docker image list -a
```

### set up GCR


```bash


```

### push an image to GCR
```bash
docker push us.gcr.io:   example:latest


```

### pull an image from GCR
```bash
```

### set up GCS
```bash
```

### create a bucket in GCR
```bash
gcloud storage buckets list
gcloud storage buckets create gs://dds-cohort-15
gcloud storage buckets list
gcloud storage ls
gcloud storage ls gs://dds-cohort-15/
```

### copy a file to a bucket in GCS
```bash
gsutil cp /tmp/input.txt gs://dds-cohort-15/
gsutil ls gs://dds-cohort-15/**
```

### install python
```bash
sudo apt-get install -y python3-pip python3-venv
python3 -m venv dsub
source dsub/bin/activate
```

### install dsub
```bash
pip3 install dsub
```

## Running dsub
```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project my-project-colab-366203 \
  --regions us-central1 \
  --logging gs://dds-cohort-15/logging/ \
  --output OUT=gs://dds-cohort-15/output/out.txt \
  --command 'echo "Hello World" > "${OUT}"' \
  --wait
```

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project my-project-colab-366203 \
  --regions us-central1 \
  --logging gs://dds-cohort-15/logging/ \
  --output OUT=gs://dds-cohort-15/output/out.script.txt \
  --script hello.sh \
  --wait
```

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project my-project-colab-366203 \
  --regions us-central1 \
  --logging gs://dds-cohort-15/logging/ \
  --script ./multi-job.sh \
  --tasks ./run.tsv \
  --wait
```

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project my-project-colab-366203 \
  --preemptible \
  --retries 3 \
  --regions us-central1 \
  --logging gs://dds-cohort-15/logging/ \
  --script ./multi-job.sh \
  --tasks ./run.tsv \
  --wait \
  &
```

```bash
dsub \
  --provider google-cls-v2 \
  --image us.gcr.io/my-project-colab-366203/rwcitek:latest \
  --project my-project-colab-366203 \
  --preemptible \
  --retries 3 \
  --regions us-central1 \
  --logging gs://dds-cohort-15/logging/ \
  --script ./multi-job.sh \
  --tasks ./run.tsv \
  --wait \
  &
```

```bash
dsub \
  --provider google-cls-v2 \
  --image us.gcr.io/my-project-colab-366203/rwcitek:latest \
  --project my-project-colab-366203 \
  --preemptible \
  --retries 3 \
  --regions us-central1 \
  --logging gs://dds-cohort-15/logging/ \
  --script ./multi-job-recon.sh \
  --tasks ./run.recon.tsv \
  --wait \
  &
```

# Notes

## Setup

In part, based on ...
- https://cloud.google.com/batch/docs/dsub

### create a project ( if needed )
- https://console.cloud.google.com/projectcreate

### create a VM in GCP
- https://console.cloud.google.com/compute/instancesAdd

Setup
- enable Compute Engine API
  - https://console.cloud.google.com/marketplace/product/google/compute.googleapis.com
- machine configuration
  - E2-micro
- os and storage
  - Boot disk:
    - Image: Ubuntu 24.04 LTS, x86/64, amd64
    - Type: standard persistent disk
- security
  - Access scopes:
    - Allow full access to all Cloud APIs
  - manage access:
    - Add manually generated SSH keys
      - add item
- Create


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
docker version
```

### start a dsub Docker instance

Pull the image
```bash
docker image pull rwcitek/dsub
```

Launch the container instance
```bash
( dsub_tmp=/tmp/dsub-$( date +%s ) &&
docker container run \
  --rm \
  --detach \
  --name    dsub \
  --volume  /var/run/docker.sock:/var/run/docker.sock \
  --volume  /tmp:/tmp \
  --env     TMPDIR=${dsub_tmp} \
  --workdir ${dsub_tmp} \
  rwcitek/dsub sleep inf
)
```

Exec into the instance
```bash
docker container exec -it dsub /bin/bash
```

Update the software in the container instance ( optional )
```bash
apt-get update
apt-get -y dist-upgrade
pip install dsub --upgrade
```

## authenticate to GCP

Not needed if running on a GCP VM.

```bash
gcloud auth login --no-launch-browser
```

## set up GCP project

Skip to the next code block if running on a GCP VM.

```bash
# list all projects
gcloud projects list

# pick one
my_project=$ ( gcloud projects list | cut -d' ' -f1 | tail -n +2 | shuf -n 1 )
my_project=abc

# set default project
gcloud config set project "${my_project}"
gcloud config get-value project
```

assign project to a variable

```bash
my_project=$( gcloud config get-value project )
```

## GCS 

### set up GCS
```bash
gsutil ls
```

### create a bucket in GCS
```bash
gsutil ls
gsutil mb gs://"${my_project}"
gsutil ls
gsutil ls gs://"${my_project}"/
```

### copy a file to a bucket in GCS

```bash
echo 'Hello, world!' > /"${TMPDIR}"/input.txt
```

```bash
gsutil cp /"${TMPDIR}"/input.txt gs://"${my_project}"/
gsutil ls gs://"${my_project}"/**
```

## application authenticate

Likely not needed in GCP VM

```bash
gcloud auth application-default login --no-launch-browser
```

## run dsub - trial 1

Run dsub

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project "${my_project}" \
  --regions us-central1 \
  --logging gs://"${my_project}"/logging/ \
  --command 'echo "Hello World"' \
  --wait \
  &
```

Look at bucket

```bash
gsutil ls gs://"${my_project}"/**
echo ===
gsutil ls gs://"${my_project}"/logging/**
```

Look at logs

```bash
gsutil cat gs://"${my_project}"/logging/echo--root*.log
```


## run dsub - trial 2

Run dsub capturing output

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project "${my_project}" \
  --regions us-central1 \
  --logging gs://"${my_project}"/logging/ \
  --output OUT=gs://"${my_project}"/output/out.txt \
  --command 'echo "Hello World" > "${OUT}"' \
  --wait \
  &
```

Look at bucket

```bash
gsutil ls gs://"${my_project}"/**
echo ===
gsutil ls gs://"${my_project}"/output/**
```

Look at output

```bash
gsutil cat gs://"${my_project}"/output/out.txt
```

## run dsub - trial 3

Run dsub running a script and capturing output

```bash
echo 'echo "Hello World" > "${OUT}"' > hello.sh
```

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project "${my_project}" \
  --regions us-central1 \
  --logging gs://"${my_project}"/logging/ \
  --output OUT=gs://"${my_project}"/output/out.script.txt \
  --script hello.sh \
  --wait \
  &
```

Look at bucket

```bash
gsutil ls gs://"${my_project}"/**
echo ===
gsutil ls gs://"${my_project}"/output/**
```

Look at output

```bash
gsutil cat gs://"${my_project}"/output/out.script.txt
```

## run dsub - trial 4

Run dsub running a script with input and capturing output

Create input data

```bash
echo 'Hello, world!' > /"${TMPDIR}"/input.txt

gsutil cp /"${TMPDIR}"/input.txt gs://"${my_project}"/input/input.txt
gsutil cat gs://"${my_project}"/input/input.txt
```

Create a script that generates output from input

```bash
<<'eof' cat > hello-script.sh
#!/bin/bash
sed -e 's/Hello/Greetings/' "${INPUT}" > "${OUTPUT}"
date >> "${OUTPUT}"
eof
```

run dsub

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project "${my_project}" \
  --regions us-central1 \
  --logging gs://"${my_project}"/logging/ \
  --input INPUT=gs://"${my_project}"/input/input.txt \
  --output OUTPUT=gs://"${my_project}"/output/in-out.script.txt \
  --script ./hello-script.sh \
  --wait \
  &
```

Look at bucket

```bash
gsutil ls gs://"${my_project}"/**
echo ===
gsutil ls gs://"${my_project}"/output/**
```

Look at output

```bash
gsutil cat gs://"${my_project}"/output/in-out.script.txt
```

## run dsub - trial 5

Run dsub running script multiple times with input and capturing output

Create input data

```bash
echo 'Hello, world!' > /"${TMPDIR}"/input.txt

gsutil cp /"${TMPDIR}"/input.txt gs://"${my_project}"/input/input.txt
gsutil cat gs://"${my_project}"/input/input.txt
```

Create a script that generates output from input

```bash
<<'eof' cat > multi-job.sh
#!/bin/bash
sed -e 's/Hello/Greetings/' "${INPUT}" > "${OUTPUT}"
date >> "${OUTPUT}"
cat /etc/os-release >> "${OUTPUT}"
eof
```

Create the TSV file with inputs and outputs

```bash
n=20
<<eof sed -e's/ *{tab} */\t/g' > run.tsv
--input INPUT  {tab} --output OUTPUT
$(
seq -w 1 10000 |
head -${n} |
while read seq ; do
  echo gs://"${my_project}"/input/input.txt {tab} gs://"${my_project}"/output/out_"${seq}".multi.txt
done
)
eof

cat -vetn run.tsv
```

run dsub

```bash
dsub \
  --provider google-cls-v2 \
  --image ubuntu:24.04 \
  --project "${my_project}" \
  --regions us-central1 \
  --logging gs://"${my_project}"/logging/ \
  --script ./multi-job.sh \
  --tasks ./run.tsv \
  --wait \
  &
```

Look at bucket

```bash
gsutil ls gs://"${my_project}"/**
echo ===
gsutil ls gs://"${my_project}"/output/*multi.txt
```

Look at output

```bash
gsutil cat gs://"${my_project}"/output/*multi.txt
```

## GCR

### FIXME: GCP adjustments
- add private VPC
- add gateway, maybe


### pull an image from Docker Hub

Pull

```bash
docker image list -a
docker pull alpine:latest
docker image list -a
```

Create modified image

```bash
cat <<'eof' > Dockerfile
FROM alpine:latest

# Install bash
RUN apk update && apk add bash

# Set bash as the default shell (optional)
CMD ["bash"]

COPY Dockerfile /
eof

docker build -t alpine-with-bash:example .
docker image list -a
```

### set up GCR

Set up to use only us.gcr.io

```bash
gcloud auth configure-docker

# vi ~/.docker/config.json
```

tag the image

```bash
my_project=$( gcloud config get-value project )
docker image tag alpine-with-bash:example us.gcr.io/"${my_project}"/dsub:example
docker image list -a | grep gcr.io
```

### push an image to GCR

```bash
docker image push us.gcr.io/"${my_project}"/dsub:example
```

### pull an image from GCR
```bash
docker image rm us.gcr.io/"${my_project}"/dsub:example
docker image pull us.gcr.io/"${my_project}"/dsub:example
```

## run dsub - trial 6

Run dsub using a small spot VM instance with a custom image

```bash
dsub \
  --provider google-cls-v2 \
  --image us.gcr.io/"${my_project}"/dsub:example \
  --project "${my_project}" \
  --machine-type e2-micro \
  --disk-size 10 \
  --preemptible \
  --retries 1 \
  --use-private-address \
  --regions us-central1 \
  --logging gs://"${my_project}"/logging/ \
  --script ./multi-job.sh \
  --tasks ./run.tsv \
  --wait \
  &
```

Look at bucket

```bash
gsutil ls gs://"${my_project}"/**
echo ===
gsutil ls gs://"${my_project}"/output/*multi.txt
```

Look at output

```bash
gsutil cat gs://"${my_project}"/output/*multi.txt
```



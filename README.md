# Docker.dsub
## Running dsub in Docker

[dsub](https://github.com/DataBiosphere/dsub) is a great way to run a bunch of commands in parallel in the cloud, currently [Google Cloud Platform](https://console.cloud.google.com/).
But dsub requires several other tools for it to work.
Those tools have been wrapped up in a [dsub Docker container](https://hub.docker.com/r/rwcitek/dsub)
to keep the host environment uncluttered.
The end result is that you can quickly get started with dsub by launching a Docker container and running dsub commands.

Below are some examples of running dsub using the local provider.

## Start a detached instance
Wrapping the commands in parentheses keeps the current shell uncluttered.
The `dsub_tmp` variable is created to provide a unique temporary folder in /tmp/ ( was easier than mktemp ).
And we mount the Docker socket ( `docker.sock` ) to create a "Docker-out-of-Docker" environment.
This enables the instance to launch other Docker instances and to clean up after itself.


Options:
- --detach  : starts the dsub instance in a detached ( or service or daemon ) mode
- --volume  : mounts both the docker.sock and the host /tmp folder.
- --workdir : ensures the tempoary folder is created within the instance
- --env     : passes the dsub_tmp variable into the instance, accessable as TMPDIR

```bash
# Optional -- pull down the image
docker image pull rwcitek/dsub
```

```bash
( dsub_tmp=/tmp/dsub-$( date +%s ) &&
docker container run \
  --detach \
  --name    dsub \
  --volume  /var/run/docker.sock:/var/run/docker.sock \
  --volume  /tmp:/tmp \
  --env     TMPDIR=${dsub_tmp} \
  --workdir ${dsub_tmp} \
  rwcitek/dsub sleep inf
)
```

## Exec into it
```bash
docker container exec -it dsub /bin/bash
```

## Setup environment
This example is using Ubuntu 22.04 and tags it with `:dsub` to aid in later identifying stopped instances.
```bash
docker image pull ubuntu:22.04
docker image tag ubuntu:22.04 ubuntu:dsub
```


## Run a command
```bash
# Run dsub
dsub \
  --provider local \
  --image ubuntu:dsub \
  --logging "${TMPDIR}/dsub-test/logging/" \
  --output OUT="${TMPDIR}/dsub-test/output/out.command.txt" \
  --command 'echo "Hello World" > "${OUT}"' \
  --wait
```

## Run a script
```bash
# Create a script
echo 'echo "Hello World" > "${OUT}"' > hello.sh

# Run dsub
dsub \
  --provider local \
  --image ubuntu:dsub \
  --logging "${TMPDIR}/dsub-test/logging/" \
  --output OUT="${TMPDIR}/dsub-test/output/out.script.txt" \
  --script hello.sh \
  --wait
```


## Run a script multiple times
From https://github.com/DataBiosphere/dsub/tree/main/examples/custom_scripts#create-a-tsv-file

```bash
# Create a mock input file
echo 'Hello, world!' > /tmp/input.txt

# Create the TSV file with inputs and outputs
<<eof sed -e's/ *{tab} */\t/g' > run.tsv
--input INPUT  {tab} --output OUTPUT
/tmp/input.txt {tab} ${TMPDIR}/dsub-test/output/out1.multi.txt
/tmp/input.txt {tab} ${TMPDIR}/dsub-test/output/out2.multi.txt
/tmp/input.txt {tab} ${TMPDIR}/dsub-test/output/out3.multi.txt
eof

# Create a script that generates output from input
<<'eof' cat > multi-job.sh
#!/bin/bash
sed -e 's/Hello/Greetings/' "${INPUT}" > "${OUTPUT}"
date >> "${OUTPUT}"
eof

# Run dsub
dsub \
  --provider local \
  --image ubuntu:dsub \
  --logging "${TMPDIR}/dsub-test/logging/" \
  --script ./multi-job.sh \
  --tasks ./run.tsv \
  --wait
```

## Clean up
dsub does not remove its containers when finished.
That's why it's nice to use an image that is tagged with "dsub"

```bash
# Remove finished dsub instances
docker container list -a |
  awk '$2 ~ ":dsub" {print $1}' |
  xargs -r docker container rm

# Remove dsub tagged image
docker image rm ubuntu:dsub

## Remove the temporary folders: output and logging
cd && rm -rf ${TMPDIR}

# unfortunately there are still zombie processes
# the only solution is to exit, stop, and restart the dsub container
ps faux | grep [d]efunct

```

Once out of the container, remove the container.
```bash
docker container stop dsub
docker container rm dsub
```

## Dockerfile
Details on how the Dockerfile was created are in a [gist](https://gist.github.com/rwcitek/b3dbb57c56d3d450bdef374f643604d5)

To view the Dockerfile that created the image.
```bash
docker container run --rm rwcitek/dsub cat /Dockerfile
```

## Run the gist
Build the image.
```bash
curl -s https://gist.github.com/rwcitek/b3dbb57c56d3d450bdef374f643604d5 |
  grep -o '/.*/raw/.*.sh' |
  xargs -r -I {} curl -L -s https://gist.githubusercontent.com/{} > dsub.docker.build.sh
bash dsub.docker.build.sh
docker image list | grep dsub
```

Push to Dockerhub.
```bash
# create tags
tag=$( date +%s )
docker image tag dsub:build-latest dsub:${tag}
docker image tag dsub:build-latest rwcitek/dsub:${tag}
docker image tag dsub:build-latest rwcitek/dsub
docker image list | grep dsub

# log in
docker login

# push to Dockerhub
docker push rwcitek/dsub:${tag}
docker push rwcitek/dsub:latest
```






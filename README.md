# Docker.dsub
## Running dsub in Docker

[dsub](https://github.com/DataBiosphere/dsub) is a great way to run a bunch of commands in parallel in the cloud.
But dsub requires a bunch of other tools for it to work.
Those tools have been wrapped up in a [dsub Docker container](https://hub.docker.com/r/rwcitek/dsub)
to keep the host environment uncluttered.
The end result is that you can simply run dsub commands from within the Docker container.

Below are some examples of running dsub using the local provider.

## Start a detached instance
Wrapping the commands in parentheses keeps the current shell uncluttered.
The dsub_tmp is created to provide a unique temporary folder in /tmp/ ( was easier than mktemp ).

Options:
- -d : starts the dsub instance in a detached ( or service or daemon ) mode
- -e : passes the dsub_tmp variable into the instance, accessable as TMPDIR
- -w : ensures the tempoary folder is created within the instance
- -v : mounts both the docker.sock ( for accessing the Docker service from within the instance ) and the host /tmp folder.

```bash
( dsub_tmp=/tmp/dsub-$( date +%s ) &&
docker container run \
  -d \
  --name dsub \
  -e TMPDIR=${dsub_tmp} \
  -w ${dsub_tmp} \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp:/tmp \
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
  --logging "${TMPDIR:-/tmp}/dsub-test/logging/" \
  --output OUT="${TMPDIR:-/tmp}/dsub-test/output/out.txt" \
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
  --logging "${TMPDIR:-/tmp}/dsub-test/logging/" \
  --output OUT="${TMPDIR:-/tmp}/dsub-test/output/out.txt" \
  --script hello.sh \
  --wait
```


## Run a script multiple times
From https://github.com/DataBiosphere/dsub/tree/main/examples/custom_scripts#create-a-tsv-file

```bash
# Create a mock input file
date > /tmp/input.txt
```

```bash
# Create the TSV file with inputs and outputs
<<eof sed -e's/{tab}/\t/' > run.tsv
--input INPUT{tab}--output OUTPUT
/tmp/input.txt{tab}/tmp/output1.txt
/tmp/input.txt{tab}/tmp/output2.txt
/tmp/input.txt{tab}/tmp/output3.txt
eof
```

```bash
# Create a script that generates output from input
<<'eof' cat > multi-job.sh
#!/bin/bash
cat ${INPUT} > ${OUTPUT}
date >> ${OUTPUT}
eof
```

```bash
# Run dsub
dsub \
  --provider local \
  --image ubuntu:dsub \
  --logging "${TMPDIR:-/tmp}/dsub-test/logging/" \
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
  xargs docker container rm 

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







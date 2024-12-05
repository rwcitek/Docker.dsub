# Notes

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

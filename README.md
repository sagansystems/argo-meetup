# Description

This repository contains the code used during the August 30, 2018 meetup describing how to build a custom reporting feature using `argo`, `helm`, and `codefresh`.

Following the steps below you will:
- Enable `kubernetes` in docker
- Deploy `argo`, `minio`, and `postgres` to your local cluster
- Populate the database with some event data
- Create an `argo` workflow to generate reports based on the data
- Create a `kubernetes` `CronJob` to run the workflow on a schedule
- Deploy the report using `helm`

# Setup

## Enable kubernetes in docker
Enable `kubernetes` in `docker` by navigating to `preferences` then the `kubernetes` tab and enabling it:

![](resources/k8s-docker.png)

Read more: https://docs.docker.com/docker-for-mac/kubernetes/

## Set up helm
If you are not familiar, `helm` is a package manager for `kubernetes`. We'll use it to deploy our services to the cluster.

```bash
$ brew install kubernetes-helm
$ helm init
```

Note: `tiller`, the service which powers `helm` on the cluster side, can take a bit of time to start up. You can check that it's running with:

```bash
$ kubectl -n kube-system get pod -l name=tiller
```

Read more: https://helm.sh/

## Deploy argo
Argo is a `kubernetes` native workflow engine. It is a very powerful tool which allows you to build complex pipelines from simple steps. It will power our custom report generation.

```bash
$ brew install argoproj/tap/argo
$ argo install
$ kubectl patch svc argo-ui -n kube-system -p '{"spec": {"type": "LoadBalancer"}}'
```

By default, this will expose the argo-ui at http://localhost:80. If this conflicts with other locally running services you can edit the service to change its port:

```bash
$ kubectl -n kube-system edit svc argo-ui
# edit spec.ports[0].port
```

Read more: https://github.com/argoproj/argo

## Deploy minio
`minio` is an `AWS S3` compatible object store which you can deploy locally.

```bash
$ helm install stable/minio --name argo-artifacts --set service.type=LoadBalancer
```

The default access key and secret key are:
AccessKey: `AKIAIOSFODNN7EXAMPLE`
SecretKey: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`

### Install minio client
```bash
$ brew install minio/stable/mc
```

You'll need to edit `~/.mc/config.json` adding access and secret from keys above.

### Create a bucket for custom reports
```bash
$ mc mb local/custom-reports
```

Read more: https://www.minio.io/

### [optional] Add minio secrets to argo
This step is not explicitly required to run this tutorial, but you'll want it if you play around with `argo` more.
Modify `workflow-controller-configmap` to add `minio` secrets.

```bash
$ kubectl edit configmap workflow-controller-configmap -n kube-system
```

See: https://github.com/argoproj/argo/blob/master/demo.md#6-reconfigure-the-workflow-controller-to-use-the-minio-artifact-repository


## Deploy postgres
We'll use `postgres` as our database.

```bash
$ helm install stable/postgresql -n dev-postgresql --set postgresPassword=supersecurehaxor --set service.type=LoadBalancer
```

# Run the workflow
The workflow is `argo`'s unit of work. You can view it in `runner/workflow.yaml`. The workflow here is very simple, it only involves a single "script" step which runs a SQL query, passed as a parameter, and stores the resulting CSV file in `minio`. To learn more about this workflow, or how to write your own, read the documentation at https://github.com/argoproj/argo/tree/master/examples.

## Initialize the db with some data
A provided script will generate some `SQL` to create a new schema, an `events` table, as well as some rows for the table.

The generated events simulate a very simple scenario where the system can send and receive messages in the context of a "session". There are two corresponding events `message_received` and `message_sent`. The report we run will will compute the duration of each session.

```bash
$ python scripts/gen-data.py | psql -d postgres://postgres:supersecurehaxor@db:5432/postgres?sslmode=disable -a
```

## Build the "runner" image
If you so choose you can build your own runner image. You will need to do this if you want to make any changes to the workflow.

```bash
$ cd runner
$ docker build . -t $YOUR_DOCKERHUB_USER/meetup-runner:$SOME_VERSION
$ docker push $YOUR_DOCKERHUB_USER/meetup-runner:$SOME_VERSION
```

You'll also need to modify line 20 of `report/templates/cron-job.yaml` with the image you just built.

## Run the workflow
If you followed the instructions up to this point, all that's left is to deploy the `CronJob` to your local k8s cluster.

You can look at the `CronJob` template in `report/templates/cron-job.yaml`. With `helm` the templated variables are replaced with the contents of a values file. In this case, that file is `environments/dev/handl-time.yaml`.

```bash
$ helm upgrade --install handle-time report -f environments/dev/handle-time.yaml
```

You can also override the cron schedule right on the command line. For example to run on the 12th minute of every hour:

```bash
$ helm upgrade --install handle-time report -f environments/dev/handle-time.yaml --set schedule="12 * * * *"
```

You can modify the query in `handle-time.yaml` to see how you can generate different reports, changing only that file.

## Verify results
You can view the workflow by navigating your browser to http://localhost:80, or whatever port you configured `argo-ui` to use.

To see the output of the report navigate to http://http://localhost:9000/minio/custom-reports/meetup/handle-time/. You'll see a timestamped subdirectory for every run of the report. The subdirectory will contain `data.tgz` which is a compressed archive containing your output CSV.
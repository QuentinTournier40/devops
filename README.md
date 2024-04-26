# Table of Content

  * [Docker project](#docker-project)
  * [Kubernetes project](#kubernetes-project)
  * [Terraform project](#terraform-project)

# Docker project

The goal of the project is to deploy the following application by using Docker and Docker compose. You will give us a GitHub or Gitlab link for your project.

![image](login-nuage-voting.drawio.svg)

## Mandatory version
- one Dockerfile per service that exposes the internal port of the container
- one docker-compose.yml file to deploy the stack
- adequate `depends_on`
- two networks will be defined: `front-net` and `back-net`
- a README file explaining how to configure and deploy the stack

## Optional extended version
- healthchecks on vote, redis and db services (some scripts are given in `healthcheck` directory)
- reducing the size of images
- multistage build on the worker service (.NET)


## Some elements

The base image will contain the basic tools for the language the application is written in.
You should use a tag to specify which version of the image you want to pull.
For building purposes, it is good practice to use a `slim` version of the image.



### `vote` service

This is a Python web server using the Flask framework. It presents a front-end for the user to submit their votes, then write them into the Redis key-value store.

For building the Dockerfile, before starting `app.py`:
- requirements have to be copied and installed in the container
- all necessary files and directories have to be copied in the container

Port mapping:
`5000` is used inside the container (see Python code). Each instance of vote will use the external port `500x` where `x` is the instance number

Healthcheck:


### `result` service

This is a Node.js web server. The front-end presents the results of the votes. The result values are taken from the PostgreSQL database.

In the Dockerfile, before running the code, make the working directory to `/usr/local/app` and
- copy package files into the container,
- install `nodemon` with `npm install -g nodemon`
- install more requirements:
```
npm ci
npm cache clean --force
mv /usr/local/app/node_modules /node_modules
```
- set the `PORT` environment variable

Finally, run the code with `node server.js`.


### `seed` service

This is a Python and bash program used to virtually send many vote requests to the `vote` server.

First the file `make-data.py` has to be executed in the container. Second, the file `generate-votes.sh` has to be executed when starting the container.

For benchmarking, `generate-votes.sh` uses the `ab` utility which needs to be installed, through the `apache2-utils` `apt` package.

### `worker` service

This is a .NET (C#) program that reads vote submissions from Redis store, compute the result and store it in the PostgreSQL database.

It requires a little bit more work to compile and run:
- use this as a base image
```
mcr.microsoft.com/dotnet/sdk:7.0
```
  with the argument `--platform=${BUILDPLATFORM}`
- use `ARG` to define build arguments `TARGETPLATFORM`, `TARGETARCH` and `BUILDPLATFORM`. Print their values with `echo`.
- in the `source/` directory, copy all worker files form this repo and run
```
dotnet restore -a $TARGETARCH
dotnet publish -c release -o /app -a $TARGETARCH --self-contained false --no-restore
```
The application will be built inside the `/app` directory, launch with `dotnet Worker.dll`.

For the multistage build, use this image: `mcr.microsoft.com/dotnet/runtime:7.0`.


### Redis service

This is a simple Redis service. Redis is a NOSQL database software focused on availability used for storing large volumes of data-structures (typically key-value pairs).

In order to perform healthchecks while Redis is running, there must be a volume attached to the container. You will need to mount local the repo directory `./healthchecks/` into the `/healthchecks/` directory of the container.

The check is done by executing the `redis.sh` script.


### PostgreSQL database service

This is a simple PostgreSQL service.

The same logic applies for healthchecks, mount a volume, use `postgres.sh` for running checks.

Moreover, in order to persist the data that comes from the votes, you need to create a Docker volume and attach it to the container.
The volume will be named `db-data` and attached to the `/var/lib/postgresql/data` directory inside the container.

### Nginx loadbalancer service

This is a simple Nginx service. At its core, Nginx is a web-server but it can also be used for other purposes such as loadbalancing, HTTP cache, reverse proxy, etc.

To configure Nginx as a loadbalancer (LB), you first need to edit accordingly the `./nginx/nginx.conf` file from this repo.
Then in the Dockerfile:
- remove the default Nginx configuration located at `/etc/nginx/conf.d/default.conf`,
- copy `./nginx/nginx.conf` into the container at the above location.


### Networking

* The Redis store, the `worker` service and the PostgreSQL database are only available inside the `back-tier` network.
* The `vote` and `result` services are on both the `front-tier` and `back-tier` network in order to (1) expose the frontend to users, and (2) communicate with the databases.
* Finally, the `seed` and Nginx loadbalancer are on the `front-tier`.



# Kubernetes project

The goal of this project is to deploy the previous application to a Kubernetes cluster. 

## Preliminary phase: push your Docker images into a GCP container registry

1. In the GCP dashboard, go to *Artifact Registry* and create a *Repository*.
Give it a name e.g. `voting-images`, and a *region* e.g. `europe-west9`.
Once created, inspect the repository and copy its path, it should look something like `europe-west9-docker.pkg.dev/your-gcp-project/voting-image`.

1. Before pushing to the registry, issue the following command in order to authenticate your laptop to the registry:
```
gcloud auth configure-docker europe-west9-docker.pkg.dev
```
Note that this command can be found in the "Setup Instructions" button in the registry repo.

1. We then need to tag the images with the corresponding registry path, followed by their original name. We can do it *either* in bulk with bare Docker Compose or one by one with Docker.

    * Option 1: Within `docker-compose.yml` file, for each service that `build`s an image, add the `image` field. E.g. for the `result` service:
      ```
      result:
        image: europe-west9-docker.pkg.dev/your-gcp-project/voting-image/result
        build:
          context: ./result
      ```
      Re-build the images with `docker compose build` and verify with `docker image ls`.

    * Option 2: For each service we need to build the image and tag the resulting hash. E.g. with `result`:
        * `docker build result/`
        * `docker tag 0cc5784ad220 europe-west9-docker.pkg.dev/nuage-k8s/login-nuage-images/result`

1. Finally, push the images. Either
    * with Docker Compose: `docker compose push`
    * or with Docker, e.g. `docker push europe-west9-docker.pkg.dev/your-gcp-project/voting-image/result`


* In case you struggle making this work, you can use our public images from Docker Hub during the session (not in the repo you will send us).
  The tags are prefixed with `eloip13009`, for example `eloip13009/result`.
  The suffixes are `vote`, `result`, `worker`, `seed-data`, `postgres-hs` and `redis-hs`.

You will notice that for the `seed-data` image, the public version only sends a total of 300 votes, instead of 3000 previously.


## Mandatory version

<!-- -->

Deploy a working application (with temporary database store)

* `vote`, `result`, `redis` and `db`, each with a `Deployment` and a `Service`.
* `worker` only needs a `Deployment`.
* `seed` will be run as a `Job` that is *not restarted*.

## Optional extensions

The two extensions are independent.

1. Make the data of the `db` persist even if the associated pod is deleted.
    * Use a `PersistentVolumeClaim`.
    * In the corresponding `Deployment`, under `volumeMounts`, there should be `subPath: data`.

1. Add `livenessProbe`s to reflect the `healthchecks` of last week's Docker project.
    * `result` and `vote` use the `httpGet` probe.
    * `redis` and `db` use the `exec` probe to run the `healthchecks/{redis.sh,postgres}.sh` scripts.

<!-- -->

## Appendix: Useful commands

Print the list of resources we can declare in a manifest, i.e. available values of `apiVersion` and `kind`

    kubectl api-resources


Print the documentation of a resource, i.e. the accepted fields in the manifest

    kubectl explain deployment.spec.template.spec.containers.livenessProbe


Print the logs of a container. When selecting a pod, `kubectl` will choose one. The `-f` options means "follow", i.e. print the logs continuously, akin to the Linux `tail -f` command.

    kubectl logs -f pods/vote<TAB>


Apply *all* manifests of a repository

    kubectl apply -f k8s-manifests/


Show resources continuously (refreshed every one second) with `watch`. Beware, `all` do not mean *all* resources, only "user" resources. E.g. `PersistentVolume`s and `StorageClass`es are not showed.

    watch -n1 kubectl get all


Execute a command in a container. E.g. dump the `votes` table in the Postgres pod.

    kubectl exec pods/db<TAB> -- pg_dump -U postgres -t public/votes


For commands that handle resources, the `-l` option applies it on resources holding the specified `labels`. E.g. to delete all resources related to the voting app (those with `metadata.labels.app = vote`)

    kubectl delete all -lapp=vote

# Terraform project

## Objectives

The objective is to use Terraform to deploy the voting app.

The tutorial on Terraform did not give you _all_ elements for this project: it was on purpose.
The point is for you to learn how to seek information in providers and other documentations.
But most elements in the tutorials can be directly applied.

Different levels are possible, the more advancement you make the better. **Part 1 and Part 2 are mandatory.**

## Part 1 mandatory - Docker

In this first part, you must write code that deploys the application with the Terraform Docker provider.
The app will thus be deployed locally inside containers on your machine.

**TIP**: start from your previous `docker-compose.yml`.

## Part 2 mandatory - GKE and Kubernetes

In this second part, you must write code that deploys the application onto a Kubernetes cluster provisioned with Terraform on GKE.
Google and Kubernetes providers will be thus be used.

**TIP**: you can start form the GKE tutorial and from your previous Kubernetes manifests.

**IMPORTANT**: Make sure to organize your Terraform code well. Attention will be given to your organization (modules, directories, files)

## Part 3 optional - GKE, Kubernetes and OpenStack

In this last part, you must deploy with Terraform the `Redis` database inside a VM on the school's OpenStack platform.
This database must then communicate with the other components of the application located on the GKE cluster.

### Changes to make this work

By default, Redis is supposed to be used only locally and does not have a password.
You must thus modify the application code that uses Redis so that they connect with a password.

#### Inside `vote/app.py`

On line 21, change
```
    g.redis = Redis(host="redis", db=0, socket_timeout=5)
```
to
```
    g.redis = Redis(host="redis", password=redis_pw, db=0, socket_timeout=5)
```

#### Inside `worker/Program.cs`

On line 116, change
```
    return ConnectionMultiplexer.Connect(ipAddress);
```
to
```
  return ConnectionMultiplexer.Connect("redis,password=changeit0");
```

#### cloud-init script to install Redis on a VM

Use this script as in cloud-init to install Redis.
```
  #!/usr/bin/env bash

  DEBIAN_FRONTEND=noninteractive apt update -q
  DEBIAN_FRONTEND=noninteractive apt install -q -y redis
```

#### Redis configuration

Finally we need to configure Redis to allow external connections with a password from the outside.

Connect to your VM in SSH, open open the file `/etc/redis/redis.conf`, find the uncommented `bind` line and replace it with
```
  bind 0.0.0.0
```

Then, look for the line with `requirepass`, remove the comment and change the password:
```
  requirepass XYZ
```

Ideally, this last configuration would be done in the cloud-init script, using `sed` for example.

## Destroy everything

To keep some credits, make sure you execute `terraform destroy`.

There is a surprise here: GKE clusters cannot be destroyed by default, we need to modify the state by hand to tell terraform it is OK to delete it.

Open `terraform.tfstate`, look for the property `deletion_protection` and set its value to `false`.

Alternatively, use `sed`:
```
    sed -e '/deletion_protection/s/true/false/' -i terraform.tfstate
```

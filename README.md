# Table of Content

  * [Docker project](#docker-project)
  * [Kubernetes project](#kubernetes-project)

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

The goal of this project is to deploy the previous application to a Kubernetes cluster. Here is a diagram of the expected infrastructure.

<!-- ![image](login-nuage-voting-k8s.drawio.svg) -->

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


## Mandatory version

<!-- Deploy a working application (with temporary database store) -->

<!-- * `vote`, `result`, `redis` and `db`, each with a `Deployment` and a `Service`. -->
<!-- * `worker` only needs a `Deployment`. -->
<!-- * `seed` is only a `Pod` that is *not restarted*. -->

## Optional extension: Persistent data on `db`

<!-- * Use a `PersistentVolumeClaim`. -->
<!--   * In the corresponding `Deployment`, under `volumeMounts`, there should be `subPath: data`. -->


# Annexe: Commandes `kubectl` utiles

Afficher la liste des resources que l'on peut déclarer dans un manifeste (`apiVersion` et `kind`)

    kubectl api-resources


Afficher la documentation d'une resource, i.e. les propriétés acceptées dans un manifeste

    kubectl explain pod.spec.containers.livenessProbe.httpGet


Afficher les logs d'un pod/conteneur en continue (`-f`)

    kubectl logs -f pods/vote<TAB>


Appliquer d'un seul coup tous les manifestes d'un répertoire.

    kubectl apply -f k8s-manifests/


Voir la météo du cluster en continue. Attention `all` ne signifie pas *toutes* les resources, seulement celles "utilisateurs", notamment les `StorageClass`es ne sont pas concernées.

    watch -n1 kubectl get all


Exécuter une commande dans un conteneur. E.g. dump la table `votes` de Postgres

    kubectl exec pods/db<TAB> -- pg_dump -U postgres -t public/votes


Pour les commandes qui manipulent des resources, l'option `-l` applique la commande uniquement sur les resources ayant les `labels` spécifiés. E.g. pour supprimer toutes les resources liés à l'application `vote` (celles qui ont `metadata.labels.app = vote`)

    kubectl delete all -lapp=vote

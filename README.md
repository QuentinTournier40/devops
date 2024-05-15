# Setup

## GCP config

You first have to setup your authentication to GCP.

```shell
gcloud auth login
```

You then have to set the project on which you want to work.

```shell
gcloud config set project PROJECT-ID
```

You can now create a Kubernetes cluster by using the command gcloud container clusters create (documentation)

```shell
gcloud container clusters create NAME-CLUSTER --machine-type n1-standard-2 --num-nodes 3 --zone us-central1-c
```

## Push your Docker images into a GCP container registry

1. In the GCP dashboard, go to *Artifact Registry* and create a *Repository*.
Give it a name e.g. `voting-images`, and a *region* e.g. `europe-west9`.
Once created, inspect the repository and copy its path, it should look something like `europe-west9-docker.pkg.dev/your-gcp-project/voting-image`.

1. Before pushing to the registry, issue the following command in order to authenticate your laptop to the registry:

```shell
gcloud auth configure-docker europe-west9-docker.pkg.dev
```

1. We then need to tag the images with the corresponding registry path, followed by their original name.

    Within `docker-compose.yml` file, for each service that `build`s an image, add the `image` field. E.g. for the `result` service:
      ```
      result:
        image: europe-west9-docker.pkg.dev/your-gcp-project/voting-image/result
        build:
          context: ./result
      ```
      Re-build the images with `docker compose build` and verify with `docker image ls`.

 2. Finally, push the images. Either
    * with Docker Compose: `docker compose push`
    * or with Docker, e.g. `docker push europe-west9-docker.pkg.dev/your-gcp-project/voting-image/result`


## Ansible

1. In the GCP dashboard, go to *Add VM Instance* and create a *VM*.

2. Check if you can acces in SSH to the VM.

```shell
gcloud compute ssh INSTANCE_NAME
```

3. Add to you VM a tag for the firewall. E.g. `db-server`.

```shell
gcloud compute instances add-tags INSTANCE_NAME --tags=db-server
```

4. In the GCP dashboard, go to *Firewall* and choose *Create Firewall Rule*.

  * Choose a name for the rule.
  * In *Target tags*, put your tag. E.g. db-server.
  * For *Source IPv4 ranges* fill with 0.0.0.0/0.
  * In "Protocols and ports", select "TCP" and enter 5432 for Postgres.

5. Change the value in the ansible inventory (`ansible/inventories/gcp.yaml`) with your ssh key and VM IP address.

6. Run the playbook:

```shell
cd ansible
ansible-playbook pg_install.yaml
```

## K8s

1. Add to each kubernetes deployement the path to the image pushed in GCP.

```yaml
spec:
  containers:
  - image: europe-west9-docker.pkg.dev/verdant-descent-421106/vote-image/redis
```

2. Add the public address of you VM in the file `pgsql-es.yaml`

# Start the application

Run these commands.

```shell
kubectl create -f ./k8s/pgsql-svc.yaml
kubectl create -f ./k8s/pgsql-es.yaml
kubectl create -f ./k8s/redis-deployment.yaml
kubectl create -f ./k8s/redis-svc.yaml
kubectl create -f ./k8s/worker-deployment.yaml
kubectl create -f ./k8s/vote-deployment.yaml
kubectl create -f ./k8s/vote-svc.yaml
kubectl create -f ./k8s/result-deployment.yaml
kubectl create -f ./k8s/result-svc.yaml
kubectl create -f ./k8s/seed-job.yaml
```

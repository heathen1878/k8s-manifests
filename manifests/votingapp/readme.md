# Voting Application

## Namespace

The namespace must be created first [using](namespace/deployment.yml). 

```shell
kubectl apply -f manifests/votingapp/namespace/deployment.yml
```

## Database

The PostgreSQL database has a number of secrets i.e. user, password...so I've used kubeseal to encrypt them; i susppose it'd be better using Key Vault or some other vault but I wanted to try kubeseal.

```shell
# Install the Kubeseal controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
```

The Kubeseal CLI is part of the [K8s-tooling](https://github.com/heathen1878/k8s-tooling) using DevBox.

```shell
# Create the secrets using kubectl with dry-run=client; i.e. print the object for kubeseal to encrypt.
# Database Admin
kubectl create secret generic postgres-secret \
--from-literal=POSTGRES_USER=admin \
--from-literal=POSTGRES_PASSWORD=letmein123 \
--from-literal=REPLICATOR_PASSWORD=passtoandfro \
--namespace=votingapp \
--dry-run=client \
--output yaml > tmp/secret.yml
```

This will create a secret.yaml file with your secrets above base64 encoded. Pass this file to kubeseal to encrypt and discard secret.yaml

```shell
kubeseal --controller-namespace=kube-system --format=yaml < tmp/secret.yml > manifests/votingapp/database/postgres-secrets.yml
```

```shell
kubectl apply -f manifests/votingapp/database/postgres-secrets.yml
```

Repeat for the application user

```shell
# Application user
kubectl create secret generic app-user-secret \
--from-literal=APP_USER=postgres \
--from-literal=APP_PASSWORD=postgres \
--namespace=votingapp \
--dry-run=client \
--output yaml > tmp/app-user-secret.yml
```

```shell
kubeseal --controller-namespace=kube-system --format=yaml < tmp/app-user-secret.yml > manifests/votingapp/database/app-user-secret.yml
```

```shell
kubectl apply -f manifests/votingapp/database/app-user-secret.yml
```

Now you can deploy the PostgreSQL database using the secrets sealed above.

```shell
kubectl apply -f manifests/votingapp/database/deployment.yml
```

## Redis

The Redis instance can be deployed using the command below; this will deploy a service and two instances of redis - I really need to understand Redis a little more in terms of resilience etc.

```shell
kubectl apply -f manifests/votingapp/redis/deployment.yml
```

## .NET Worker

The .NET worker processes key / values found in Redis and persists them into the database. It uses the services of the Database and Redis to communicate with the backend pods.

To deploy the worker run.

```shell
kubectl apply -f manifests/votingapp/worker/deployment.yml
```


# Deploy on Minikube

!!! abstract
    This note descriptes how to install and run the solution on minikube instead of docker-compose so developer has the same experience as running on any kubernetes platform.

## Pre-requisites

See the common pre-requisites from [this note](../pre-requisites.md).

You need to have minikube installed and helm - tiller deployed on it, (see [these instructions](https://docs.bitnami.com/kubernetes/get-started-kubernetes/#step-4-install-and-configure-helm-and-tiller) for this installation).

Install `kubectl` CLI.

Verify Tiller is running:

```
kubectl --namespace kube-system get pods | grep tiller
```

Add Bitnami Helm Repository:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Be sure to have updated the helm repository with:

```
helm repo update
```

## Setting up Kafka and Zookeeper

* Create a namespace.

```
  kubectl create namespace greencompute
```

* Deploy kafka and zookeeper using bitnami/kafka helm

  ```
  helm install --name kafka --set persistence.enabled=false bitnami/kafka --namespace greencompute
  ```

It will take minutes to get the 2 pods ready.

* Create the needed kafka topics

  ```
  ./scripts/createMinikubeLocalTopics.sh
  ```

  Get the kafka pod name:

  ```
  export POD_NAME=$(kubectl get pods --namespace greencompute -l "app.kubernetes.io/name=kafka,app.kubernetes.io/instance=kafkabitnami,app.kubernetes.io/component=kafka" -o jsonpath="{.items[0].metadata.name}")
  ```

  Get the list of topics

  ```
  kubectl exec  -ti $POD_NAME  -n greencompute -- kafka-topics.sh --list --zookeeper kafkabitnami-zookeeper:2181
  ```

---

## Deploy each component of the solution

To summarize the MVP solution looks like:

![](../design/kc-mvp-components.png)


### Deploy postgresql using Helm

!!! warning
    Deploy postgresql only if you plan to use the container-ms service.

```
helm install --name postgre-db \
  --set postgresqlPassword=supersecret \
  --set persistence.enabled=false \
    stable/postgresql --namespace greencompute
```

Once the pod is running access to the password using

```
export POSTGRES_PASSWORD=$(kubectl get secret --namespace greencompute postgre-db-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)
```

And then use the `psql` command line interface to interact with postgresql. For that we are using a docker image as client to the postgresql server running in Minikube:

```
kubectl run postgre-db-postgresql-client --rm --tty -i --restart='Never' --namespace greencompute --image docker.io/bitnami/postgresql:11.3.0-debian-9-r38 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgre-db-postgresql -U postgres -p 5432
```

The two above commands are defined in the bash script named `scripts/startPsql.sh`.

!!! note
    `psql` will be use to verify tests execution

Also to connect to your database from outside the cluster execute the following commands:

```
    kubectl port-forward --namespace greencompute svc/postgre-db-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -p 5432
```

### Deploy the Voyage microservice using Helm

The *Voyage microservice* is a simple nodejs app to mockup schedule of vessels between two harbors. It is here to illustrate Kafka integration with nodejs app.

* Go to the repo

```
$ cd refarch-kc-ms/voyages-ms
```

** Build the image

```
$ ./scripts/buildDocker.sh MINIKUBE
```

* Deploy on minikube

```
helm install chart/voyagesms/ --name voyages --set image.repository=ibmcase/kc-voyagesms --set image.pullSecret= --set image.pullPolicy=IfNotPresent --set eventstreams.brokers=kafkabitnami:9092 --set eventstreams.env=local --namespace greencompute
```

* Verify it is correctly running

```
curl http://localhost:31000/voyage
```

### Deploy Order command microservice with Helm

!!! note
    Order command microservice implements the Command part of the CQRS pattern. It is done in Java and use Kafka API.

* Go to the repo

```
$ cd refarch-kc-order-ms/order-command-ms
```

* Build the image

```
$ ./scripts/buildDocker.sh MINIKUBE
```

* Deploy on minikube

```
helm install chart/ordercommandms/ --name ordercmd --set image.repository=ibmcase/kc-ordercommandms --set image.pullSecret= --set image.pullPolicy=IfNotPresent --set eventstreams.brokers=kafkabitnami:9092 --set eventstreams.env=MINIKUBE --namespace greencompute
```

* Verify service runs

Without any previously tests done, the call below should return an empty array: `[]`
```
curl http://localhost:31200/orders
```


### Order query microservice with Helm

!!! note
    Order command microservice implements the Query part of the CQRS pattern. It is done in Java and use Kafka API.

* Go to the repo

```
$ cd refarch-kc-order-ms/order-query-ms
```

* Build the image

```
$ ./scripts/buildDocker.sh MINIKUBE
```

* Deploy on minikube

```
helm install chart/orderqueryms/ --name orderquery --set image.repository=ibmcase/kc-orderqueryms --set image.pullSecret= --set image.pullPolicy=IfNotPresent --set eventstreams.brokers=kafkabitnami:9092 --set eventstreams.env=MINIKUBE --namespace greencompute
```

* Verify service runs

At the beginning the call below should return an empty array: `[]`
```
curl http://localhost:31100/orders
```

### Deploy the Fleet simulator with Helm

!!! note
    The fleet simulator is to move vessels from one harbors to another, and send container metrics while the containers are on a vessel. It has some predefined simulation to trigger some events.

* Go to the repo

```
$ cd refarch-kc-ms/fleet-ms
```

* Build the image

```
$ ./scripts/buildDocker.sh MINIKUBE
```


* Deploy on minikube

```
helm install chart/fleetms/ --name fleetms --set image.repository=ibmcase/kc-fleetms --set image.pullSecret= --set image.pullPolicy=IfNotPresent --set eventstreams.brokers=kafkabitnami:9092 --set eventstreams.env=MINIKUBE --namespace greencompute
```

* Verify service runs

At the beginning the call below should return an empty array: `[]`
```
curl http://localhost:31300/fleetms/fleets
```

### Deploy the Container management microservice with Helm

The container microservice manage the Reefer container inventory and listen to order created event to assign a container to an order.

!!! warning
    There are multiple different implementations of this service. This note is for the Springboot / Postgresql / Kafka implementation.

#### Getting the certs for IBM CLoud PostgreSQL

!!! warning
    If you run with postgresql running locally, you have nothing to do.

If you run the container microservice connected to Postgresql running on IBM Cloud, you need to get the SSL certificate. To do so perform the following commands:

* From the IBM Cloud Console get the name of your postgresql service
  For example the name we have is: Green-DB-PostgreSQL
* Then get the certificates

```
ibmcloud cdb deployment-cacert <your-PostgreSQL-service-name> > postgresql.crt
# if you do not have the cloud database plugin does the following and rerun previous command:
ibmcloud plugin install cloud-databases
```

* Transform the certificate format

```
openssl x509 -in postgressql.crt -out postgressql.crt.der -outform der
```

* Save it in Java keystore

```
keytool -keystore clienttruststore -alias postgresql -import -file postgressql.crt.der -storepass changeit
```

* Create required secrets.

```
kubectl create secret generic postgresql-url --from-literal=binding='jdbc:postgresql://postgre-db-postgresql:5432/postgres' -n greencompute

kubectl create secret generic postgresql-user --from-literal=binding='postgres' -n greencompute

kubectl create secret generic postgresql-pwd --from-literal=binding='supersecret' -n greencompute
```

### Build and deploy the container manager microservice

* Go to the repo

```
cd refarch-kc-container-ms/SpringContainerMS
```

* Build the image

```
$ ./scripts/buildDocker.sh MINIKUBE
```

* Deploy on minikube

```
helm install chart/springcontainerms/ --name containerms --set image.repository=ibmcase/kc-springcontainerms --set image.pullSecret= --set image.pullPolicy=IfNotPresent --set eventstreams.brokers=kafkabitnami:9092 --set eventstreams.env=MINIKUBE --namespace greencompute
```

* Verify the deployed service:

```
curl http://localhost:30626/containers
```

### Deploy the User Interface app used for demonstration

The last app is a web application to expose an user interface to run the demonstration end to end.

* Go to the repo

```
cd refarch-kc-ui/
```

* Build the image

```
./scripts/buildDocker.sh MINIKUBE
```

* Deploy on minikube

```
helm install chart/kc-ui/ --name kcsolution --set image.repository=ibmcase/kc-ui --set image.pullSecret= --set image.pullPolicy=Always --set eventstreams.brokers=kafkabitnami:9092 --set eventstreams.env=MINIKUBE --namespace greencompute
```

* Verify the installed app

Point your web browser to [http://localhost:31010](http://localhost:31010) and login with username: eddie@email.com and password Eddie.


## Troubleshouting

In case of issue try accessing the pod logs.

Use busybox if you want to test that KubeDNS is working as expected and hosts are resolving and can connect:

```
kubectl run -i --tty --rm debug -n greencompute --image=busybox --restart=Never -- sh
```

In the shell prompt you have access to ping command to test name resolution.

### Zookeeper or kafka bitnami pod in ImagePullBackOff

The issue may happen if you are not connected to the internet (yes we code in planes !). The statefulsets are using images with docker.io/bitnami/zookeeper or docker.io/bitnami/kafka, but when the images are local, they look like:

```
bitnami/zookeeper                    3.4.14-debian-9-r25  
bitnami/kafka                        2.2.1-debian-9-r12 
```

So you need to edit the image tag in the stateful yaml. To do so perform the kubectl:

```
kubectl edit statefuleset kafkabitnami
kubectl edit statefuleset kafkabitnami-zookeeper
```
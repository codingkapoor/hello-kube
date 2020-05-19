# hello-kube project

The original hello project has been generated by the lagom/lagom-scala.g8 template. Kubernetes configuration has been added to show how a Lagom project can be deployed in a kubernetes cluster.

### Deployment

The steps to follow for deploying this project in kubernetes are:

- Start a kubernetes cluster. For test purposes you can use minikube
  ```
  $ minikube --vm-driver=virtualbox start
  ```
- Connect docker cli to minikube 
  ```
  $ eval $(minikube -p minikube docker-env)
  ```
- Create and publish the Docker container with the service. Since we have already connected docker cli to minikube in last step the newly published images will be made available directly within minikube docker instance reach 
  ```
  $ sbt docker:publishLocal
  ```
- Create the secrets with database and play secret
  ```
  $ kubectl create secret generic db-secret --from-literal=username=<user> --from-literal=password=<password>
  $ kubectl create secret generic hello-application-secret --from-literal=secret="$(openssl rand -base64 48)"
  ```
- Deploy rbac configuration role
  ```
  $ kubectl apply -f kubernetes/discovery-role.yaml
  ```
- Fetch IP of host machine addressable within the kubernetes cluster
  ```
  $ minikube ssh "route -n | grep ^0.0.0.0 | awk '{ print \$2 }'"
  ```
- Change the IP for Cassandra services and deploy them
  ```
  $ kubectl apply -f kubernetes/cassandra-service.yaml
  ```
- Create a new namespace for strimzi kafka cluster, install strimzi operator and create strimzi kafka cluster
  ```
  $ kubectl create namespace kafka
  $ kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
  $ kubectl apply -f kubernetes/kafka-service.yaml -n kafka 
  ```
- Use following SRV in hello service deployment config
  ```
  - name: KAFKA_SERVICE_NAME
    value: "_tcp-clients._tcp.strimzi-kafka-brokers.kafka.svc.cluster.local"
  ```
  The TCP name could be fetched from the service broker definition:
  ```
  $ kubectl -n kafka get service strimzi-kafka-brokers -o yaml
  
  spec:
  clusterIP: None
  ports:
  - name: tcp-replication
    port: 9091
    protocol: TCP
    targetPort: 9091
  - name: tcp-clients
    port: 9092
    protocol: TCP
    targetPort: 9092
  - name: tcp-clientstls
    port: 9093
    protocol: TCP
    targetPort: 9093

  ```
- Verify if the kafka cluster is accessible to both producers and consumer
  ```
  $ kubectl -n kafka run kafka-producer -ti --image=strimzi/kafka:0.17.0-kafka-2.4.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
  $ kubectl -n kafka run kafka-consumer -ti --image=strimzi/kafka:0.17.0-kafka-2.4.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
  $ kubectl -n kafka exec -ti strimzi-kafka-0 -- bin/kafka-topics.sh --zookeeper localhost:2181 --list
  ```
- Deploy the microservice.<br>
  ```
  $ kubectl apply -f kubernetes/hello-deployment.yaml
  ```
- Enable ingress addon and create ingress service
  ```
  $ minikube addons enable ingress
  $ kubectl apply -f kubernetes/ingress-service.yaml
  ```
- Fetch minikube cluster address
```
$ minikube ip
$ curl -H "Content-Type: application/json" -X POST -d '{"message": "Hola"}' http://192.168.99.101/api/hello/Alice
$ curl -H "Content-Type: application/json" -X GET http://192.168.99.101/api/hello/Alice
```

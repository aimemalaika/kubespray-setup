Installing the dependencies

This quickstart assumes that you have the latest version of the minikube binary, which you can get from the minikube website.

Minikube requires a container or virtual machine manager. The Minikube documentation includes a list of suggested options in the getting started guide.

You’ll also need the kubectl binary, which you can get by following the kubectl installation instructions from the Kubernetes website.

Once you have all the binaries installed, make sure everything works:

# Validate minikube
minikube version

# Validate kubectl
kubectl version

Starting the Kubernetes cluster

Start a local development cluster of Minikube that runs in a container or virtual machine manager.

minikube start --memory=4096 # 2GB default memory isn't always enough

Deploy Strimzi using installation files

Before deploying the Strimzi cluster operator, create a namespace called kafka:

kubectl create namespace kafka

Apply the Strimzi install files, including ClusterRoles, ClusterRoleBindings and some Custom Resource Definitions (CRDs). The CRDs define the schemas used for the custom resources (CRs, such as Kafka, KafkaTopic and so on) you will be using to manage Kafka clusters, topics and users.

kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

The YAML files for ClusterRoles and ClusterRoleBindings downloaded from strimzi.io contain a default namespace of myproject. The query parameter namespace=kafka updates these files to use kafka instead. By specifying -n kafka when running kubectl create, the definitions and configurations without a namespace reference are also installed in the kafka namespace. If there is a mismatch between namespaces, then the Strimzi cluster operator will not have the necessary permissions to perform its operations.

Follow the deployment of the Strimzi cluster operator:

kubectl get pod -n kafka --watch

You can also follow the operator’s log:

kubectl logs deployment/strimzi-cluster-operator -n kafka -f

Once the operator is running it will watch for new custom resources and create the Kafka cluster, topics or users that correspond to those custom resources.
Create an Apache Kafka cluster

Create a new Kafka custom resource to get a single node Apache Kafka cluster:

# Apply the `Kafka` Cluster CR file
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka 

Wait while Kubernetes starts the required pods, services, and so on:

kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka 

The above command might timeout if you’re downloading images over a slow connection. If that happens you can always run it again.
Send and receive messages

With the cluster running, run a simple producer to send messages to a Kafka topic (the topic is automatically created):

kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.48.0-kafka-4.1.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic

Once everything is set up correctly, you’ll see a prompt where you can type in your messages:

If you don't see a command prompt, try pressing enter.

>Hello Strimzi!

And to receive them in a different terminal, run:

kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.48.0-kafka-4.1.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning

If everything works as expected, you’ll be able to see the message you produced in the previous step:

If you don't see a command prompt, try pressing enter.

>Hello Strimzi!

Enjoy your Apache Kafka cluster, running on Minikube!
Deleting your Apache Kafka cluster

When you are finished with your Apache Kafka cluster, you can delete it by running:

kubectl -n kafka delete $(kubectl get strimzi -o name -n kafka)

This will remove all Strimzi custom resources, including the Apache Kafka cluster and any KafkaTopic custom resources but leave the Strimzi cluster operator running so that it can respond to new Kafka custom resources.

Next, delete the Persistent Volume Claim (PVC) that was used by the cluster:

kubectl delete pvc -l strimzi.io/name=my-cluster-kafka -n kafka

Without deleting the PVC, the next Kafka cluster you might start will fail as it will try to use the volume that belonged to the previous Apache Kafka cluster.
Deleting the Strimzi cluster operator

When you want to fully remove the Strimzi cluster operator and associated definitions, you can run:

kubectl -n kafka delete -f 'https://strimzi.io/install/latest?namespace=kafka'

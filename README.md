
### Installation Instructions:

# Note: as part of of this installation, we will be installing: 
        - 3 node RabbitMQ Cluster (with Clustering & HA)
        - 1 node ActiveMQ Instance (No Clustering, No HA)

# make sure your machine is connected with Oracle Cloud Account

# make sure your machine has docker installed & configured with OCIR registry & repository (container registry & repository)


# Download Zip File (contains Kubernetes Artifacts):
  download zip file: <MQ-08-July-2022 path>
  extract it & open it in IDE


# Pull Public RabbitMQ Image, tag it with OCIR Repository, push it to OCIR Repository
  docker pull rabbitmq:3.10.5-management
  docker tag rabbitmq:3.10.5-management <REPLACE-ME-OCIR-REPO>/rabbitmq:3.10.5-management
  docker push <REPLACE-ME-OCIR-REPO>/rabbitmq:3.10.5-management


# Pull Public Busybox Image, tag it with OCIR Repository, push it to OCIR Repository
  docker pull busybox:1.35.0
  docker tag rabbitmq:3.10.5-management <REPLACE-ME-OCIR-REPO>/busybox:1.35.0
  docker push <REPLACE-ME-OCIR-REPO>/busybox:1.35.0


# Build a new ActiveMQ Image, tag it with OCIR Repository, push it to OCIR Repository
  cd MQ-08-July-2022/02-activemq/01-docker-image
  docker build -t <REPLACE-ME-OCIR-REPO>/apache-activemq:5.10.0 .
  docker push <REPLACE-ME-OCIR-REPO>/apache-activemq:5.10.0


# Replace new Container Images in Kubernetes YAML files (for RabbitMQ):
  cd MQ-08-July-2022/01-rabbitmq/02-deployment-through-plain-k8s-artifacts
  open 06-statefulset.yaml & replace below lines:
        image: busybox:1.35.0 --> <REPLACE-ME-OCIR-REPO>/busybox:1.35.0
        image: rabbitmq:3.10.5-management --> <REPLACE-ME-OCIR-REPO>/rabbitmq:3.10.5-management


# Replace new Container Images in Kubernetes YAML files (for ActiveMQ):
  cd MQ-08-July-2022/02-activemq/02-deployment-through-plain-k8s-artifacts
  open 06-deployment.yaml & replace below lines:
        image: akashdocker29/apache-activemq:5.10.0 --> <REPLACE-ME-OCIR-REPO>/apache-activemq:5.10.0


# Verify all changes in Kubernetes YAML files (for both RabbitMQ & ActiveMQ):
    - Resources (CPU/Memory/Disk)
    - Container Images


# make sure your machine has kubectl installed & configured it "FrontOffice" Kubernetes Cluster. Then apply changes for RabbitMQ:
    - cd MQ-08-July-2022/01-rabbitmq/02-deployment-through-plain-k8s-artifacts
    - kubectl apply -f .

    - Verify:
            - kubectl get pods -n rabbitmq-system (it takes some time to get all 3 statefulset pods running and  ready status = 1/1)
            - We can validate it:
                    - by describing pod: kubectl describe pod <REPLACE-ME-POD-NAME> -n rabbitmq-system
                    - by checking logs: kubectl logs <REPLACE-ME-POD-NAME> -n rabbitmq-system
            - Note down Load Balancer IP of RabbitMQ Management Service:
                    - kubectl get svc -n rabbitmq-system (note down EXTERNAL-IP of rabbitmq-management)
            - Go to machine's browser (chrome etc.) type: <EXTERNAL_IP>:15672 (it should open RabbitMQ management console) & enter username/password:
                    - username: Rabbit@123
                    - password: Rabbit@123
            
            - Broker, Queue & Message Durability Testing:
                   
                    - Navigate to Queue Tab

                    - Create Queues:
                            - Create a new Classic Queue:
                                    - Name: test-classic-queue-durable
                                    - Type: Classic
                                    - Durability: Durable
                                    - Node: rabbitmq-cluster-server-0.rabbitmq-cluster-nodes.rabbitmq-system.svc.- cluster.local
                            - Create a new Classic Queue:
                                    - Name: test-classic-queue-transient
                                    - Type: Classic
                                    - Durability: Transient
                                    - Node: rabbitmq-cluster-server-0.rabbitmq-cluster-nodes.rabbitmq-system.svc.cluster.local
                            - Create a new Quorum Queue:
                                    - Name: test-quorum-queue
                                    - Type: Quorum
                                    - Node: rabbitmq-cluster-server-0.rabbitmq-cluster-nodes.rabbitmq-system.svc.cluster.local

                    - Create Exchanges:
                            - Create a new direct exchange for Queue (test-classic-queue-durable):
                                    - Name: test-classic-exchange-durable
                                    - Type: Direct
                                    - Durability: Durable
                            - Create a new direct exchange for Queue (test-classic-queue-transient):
                                    - Name: test-classic-exchange-transient
                                    - Type: Direct
                                    - Durability: Transient
                            - Create a new direct exchange for Quorum Queue (test-quorum-queue):
                                    - Name: test-quorum-exchange
                                    - Type: Direct
                                    - Durability: Durable

                    - Create Bindings (between exchange & queues):
                            - Navigate to Exchanges Tab
                            - Click Exchange (test-classic-exchange-durable):
                                    - Click Bindings:
                                            - To queue: test-classic-queue-durable
                                            - Routing key: test-classic-queue-durable-routing
                                    - Publish Messages:
                                            - Routing key: test-classic-queue-durable-routing
                                            - Payload: testing classic queue and exchange (durable)
                                    - Validate:
                                            - Navigate to Queues Tab
                                            - Queue (test-classic-queue-durable) should have 1 message (For Ready & Total)
                            
                            - Navigate to Exchanges Tab
                            - Click Exchange (test-classic-exchange-transient):
                                    - Click Bindings:
                                            - To queue: test-classic-queue-transient
                                            - Routing key: test-classic-queue-transient-routing
                                    - Publish Messages:
                                            - Routing key: test-classic-queue-transient-routing
                                            - Payload: testing classic queue and exchange (transient)
                                    - Validate:
                                            - Navigate to Queues Tab
                                            - Queue (test-classic-queue-transient) should have 1 message (For Ready & Total)
                            
                            - Navigate to Exchanges Tab
                            - Click Exchange (test-quorum-exchange):
                                    - Click Bindings:
                                            - To queue: test-quorum-queue
                                            - Routing key: test-quorum-queue-routing
                                    - Publish Messages:
                                            - Routing key: test-quorum-queue-routing
                                            - Payload: testing quorum queue and exchange
                                    - Validate:
                                            - Navigate to Queues Tab
                                            - Queue (test-quorum-queue) should have 1 message (For Ready & Total)
                    
                     - Navigate to Overview Tab
                            - from command line delete first pod:
                                    - kubectl delete pod rabbitmq-cluster-server-0 -n rabbitmq-system
                                    - verify:
                                            - from command line: kubectl get pods -n rabbitmq-system 
                                            - from rabbitmq admin console: You will be able to see first pod is down, kubernetes is creating a new one (same we can verify on Overview Tab).
                            
                            - do the same from command line for second pod:
                                    - kubectl delete pod rabbitmq-cluster-server-1 -n rabbitmq-system
                                    - verify:
                                            - from command line: kubectl get pods -n rabbitmq-system 
                                            - You will be able to see second pod is down, kubernetes is creating a new one (same we can verify on Overview Tab).
                            
                            - do the same from command line for third pod:
                                    - kubectl delete pod rabbitmq-cluster-server-2 -n rabbitmq-system
                                     - verify:
                                            - from command line: kubectl get pods -n rabbitmq-system 
                                            - You will be able to see third pod is down, kubernetes is creating a new one (same we can verify on Overview Tab).

                            

# make sure your machine has kubectl installed & configured it "BackOffice" Kubernetes Cluster. Then apply changes for ActiveMQ:
    - cd MQ-08-July-2022/02-activemq/02-deployment-through-plain-k8s-artifacts
    - kubectl apply -f .

    - Verify:
            - kubectl get pods -n activemq-system (wait till pod reach in running state and ready status = 1/1)
            - We can validate it:
                    - by describing pod: kubectl describe pod <REPLACE-ME-POD-NAME> -n activemq-system
                    - by checking logs: kubectl logs <REPLACE-ME-POD-NAME> -n activemq-system
            - Note down Load Balancer IP of ActiveMQ Management Service:
                    - kubectl get svc -n activemq-system (note down EXTERNAL-IP of activemq-management)
            - Go to machine's browser (chrome etc.) type: <EXTERNAL_IP>:8161 (it should open ActiveMQ management console) & enter username/password:
                    - username: admin
                    - password: ActiveAdmin@123

            - Broker, Queue & Message Durability Testing:

                    - Navigate to Queue Tab:
                            - Create Queue:
                                    - Queue Name: test-queue
                                    - click "Create" button

                    - Navigate to Send Tab:
                            - Send one message to the queue:
                                    - Destination: test-queue
                                    - Queue or Topic dropdown: Queue
                                    - Persistent Delivery checkbox: checked
                                    - Message Body: "Test Message - Durable"
                                    - click "Send" button
                            - Send another message to the queue:
                                    - Destination: test-queue
                                    - Queue or Topic dropdown: Queue
                                    - Persistent Delivery checkbox: unchecked
                                    - Message Body: "Test Message - Transient"
                                    - click "Send" button
                    
                    - from command line delete the activemq pod:
                            - kubectl get pods -n activemq-system (note down POD-NAME)
                            - kubectl delete pod <REPLACE-POD-NAME-FROM-ABOVE-STEP> -n activemq-system
                            - verify:
                                    - from activemq admin console in browser:
                                            - broker will be down (because HA has not yet implemented for ActiveMQ)
                                    - from command line: kubectl get pods -n rabbitmq-system:
                                            - (wait till pod reach in running state and ready status = 1/1)
                                    - from activemq admin console in browser:
                                            - broker will be up
                                            - Queue servive - broker restart
                                            - Durable Message - servive broker restart
                                            - Transient Message lost while broker restart



# RabbitMQ & ActiveMQ Broker details for MQ Clients / Publisher / Consumer (probably Intellect team needs this):

- RabbitMQ:
        - host = rabbitmq-client.rabbitmq-system (kubernetes-service.kubernetes-namespace)
        - port = 5672
        - username = Rabbit@123
        - password = Rabbit@123

- ActiveMQ:
        - host = activemq-client.activemq-system (kubernetes-service.kubernetes-namespace)
        - port = 5672
        - username = admin
        - password = ActiveAdmin@123


# RabbitMQ & ActiveMQ Management UI details (for Oracle / Intellect / ICICI):

- RabbitMQ:
        - run following command from command line:
                - kubectl get svc -n rabbitmq-system
                - note down "EXTERNAL-IP" of "rabbitmq-management" service
                - go to browser (chrome etc.) & type: EXTERNAL-IP:15672 
                - Username & Password for Login:
                        - username = Rabbit@123
                        - password = Rabbit@123

- ActiveMQ:
         - run following command from command line:
                - kubectl get svc -n activemq-system
                - note down "EXTERNAL-IP" of "activemq-management" service
                - go to browser (chrome etc.) & type: EXTERNAL-IP:8161 
                - Username & Password for Login:
                        - username = admin
                        - password = ActiveAdmin@123


# Note: for Oracle Team:
        - This is the initial setup, which is done in very short span of time-frame, so work can progress. This setup will be improved in next couple of weeks (with more testing). Adding this note so there won't be any communication gap.
[![Build Status](https://travis-ci.org/ballerina-guides/messaging-with-jms-queues.svg?branch=master)](https://travis-ci.org/ballerina-guides/messaging-with-jms-queues)

# Inter-Process Communication in a Microservices Architecture

In this guide we are focusing on building applications with a microservices architecture. In this article, we are considering a look at how the services within a system communicate with one another.

> This guide walks you through the process of describing implementing inter-process communication using Ballerina programming language.

The following are the sections available in this guide.

- [What you'll build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
- [Testing](#testing)
- [Deployment](#deployment)
- [Observability](#observability)

## What you’ll build
In a monolithic application process, components interactions are designed in a way that invoke one another via language‑level method or function calls. On the other hand, a microservices‑based application fully focused on distributed system running on multiple singular containers. Each service instance is typically a process. Consequently, as the following diagram shows, services must interact using an inter‑process communication (IPC) mechanism.

When selecting an Inter process communication mechanism for a service, it is always useful to think first about how services interact. There are a variety of client⇔service interaction styles. They can be categorized along two dimensions. The first dimension is whether the interaction is one‑to‑one or one‑to‑many:

- One‑to‑one – Each client request is processed by exactly one service instance.
- One‑to‑many – Each request is processed by multiple service instances.
- The second dimension is whether the interaction is synchronous or asynchronous:

- Synchronous – The client expects a timely response from the service and might even block while it waits.
- Asynchronous – The client doesn’t block while waiting for a response, and the response, if any, isn’t necessarily sent immediately.

Each service typically uses a combination of these interaction styles. For some services, a single IPC mechanism is sufficient. Other services might need to use a combination of IPC mechanisms. The following diagram shows how services in a taxi-hailing application might interact when the user requests a trip.

![alt text](/images/Richardson-microservices-part3-monolith-vs-microservices-1024x518.png)

In Microservices architecture, The services use a combination of notifications, request/response, and publish/subscribe. For example, the passenger’s smartphone sends a notification to the Trip Management service to request a pickup. The Trip Management service verifies that the passenger’s account is active by using request/response to invoke the Passenger Service. The Trip Management service then creates the trip and uses publish/subscribe to notify other services including the Dispatcher, which locates an available driver.
![alt text](/images/Richardson-microservices-part3-taxi-service.png)


So, lets try to demonstrate the scenario,
In this example `Apache ActiveMQ` has been used as the JMS broker for inter process communication through notification channel. Ballerina JMS Connector is used to connect Ballerina 
and JMS Message Broker. With this JMS Connector, Ballerina can act as both JMS Message Consumer and JMS Message 
Producer.

## Prerequisites
 
- [Ballerina Distribution](https://ballerina.io/learn/getting-started/)
- A JMS Broker (Example: [Apache ActiveMQ](http://activemq.apache.org/getting-started.html))
  * After installing the JMS broker, copy its .jar files into the `<BALLERINA_HOME>/bre/lib` folder
    * For ActiveMQ 5.12.0: Copy `activemq-client-5.12.0.jar` and `geronimo-j2ee-management_1.1_spec-1.0.1.jar`
- A Text Editor or an IDE 

### Optional Requirements
- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina), [Atom](https://atom.io/packages/language-ballerina))
- [Docker](https://docs.docker.com/engine/installation/)
- [Kubernetes](https://kubernetes.io/docs/setup/)

## Implementation

> If you want to skip the basics, you can download the git repo and directly move to the "Testing" section by skipping "Implementation" section.    

### Create the project structure

Ballerina is a complete programming language that supports custom project structures. Use the following package structure for this guide.
```
├── dispatcher
│   └── dispatcher.bal
├── driver-management
│   └── driver-management.bal
├── notification
│   └── notification.bal
├── passenger-management
│   └── passenger-management.bal
└── trip-management
    └── trip-management.bal
```

- Create the above directories in your local machine and also create empty `.bal` files.

- Then open the terminal and navigate to `inter-prceosss-communication/guide` and run Ballerina project initializing toolkit.
```bash
   $ ballerina init
```

### Developing the service

Let's get started with the implementation of the `trip-management.bal`, which acts as the middle man to coordinate the interactions between passenger management and dispatcher. 
Refer to the code attached below. Inline comments added for better understanding.

##### trip-management.bal
```ballerina
import ballerina/log;
import ballerina/http;
import ballerina/jms;
import ballerinax/docker;

// Type definition for a book order
type pickup {
    string customerName;
    string address;
    string phonenumber;
};


// Initialize a JMS connection with the provider
// 'providerUrl' and 'initialContextFactory' vary based on the JMS provider you use
// 'Apache ActiveMQ' has been used as the message broker in this example
jms:Connection jmsConnection = new({
        initialContextFactory: "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
        providerUrl: "tcp://localhost:61616"
    });

// Initialize a JMS session on top of the created connection
jms:Session jmsSession = new(jmsConnection, {
        acknowledgementMode: "AUTO_ACKNOWLEDGE"
    });

// Initialize a queue sender using the created session
endpoint jms:QueueSender jmsTripDispatchOrder {
    session:jmsSession,
    queueName:"trip-dispatcher"
};

// Client endpoint to communicate with passager management service
endpoint http:Client passengerMgtEP {
    url:"http://localhost:9091/passenger-management"
};


// Service endpoint
endpoint http:Listener listener {
    port:9090
};


// Trip management serice, which will take client pickup request
@http:ServiceConfig {basePath:"/trip-manager"}
service<http:Service> TripManagement bind listener {
    // Resource that allows users to place an order for a book
    @http:ResourceConfig { methods: ["POST"], consumes: ["application/json"],
        produces: ["application/json"], path : "/pickup" }
    pickup(endpoint caller, http:Request request) {
        http:Response response;
        pickup pickup;
        json reqPayload;

        // Try parsing the JSON payload from the request
        match request.getJsonPayload() {
            // Valid JSON payload
            json payload => reqPayload = payload;
            // NOT a valid JSON payload
            any => {
                response.statusCode = 400;
                response.setJsonPayload({"Message":"Invalid payload - Not a valid JSON payload"});
                _ = caller -> respond(response);
                done;
            }
        }

        json name = reqPayload.Name;
        json address = reqPayload.pickupaddr;
        json contact = reqPayload.ContactNumber;


        // If payload parsing fails, send a "Bad Request" message as the response
        if (name == null || address == null || contact == null) {
            response.statusCode = 400;
            response.setJsonPayload({"Message":"Bad Request - Invalid Trip Request payload"});
            _ = caller -> respond(response);
            done;
        }

        // Order details
        pickup.customerName = name.toString();
        pickup.address = address.toString();
        pickup.phonenumber = contact.toString();
    
        log:printInfo("Calling passenger management service:");
      
        // call passanger-management and get passagner orginization claims
        json responseMessage;
        http:Request passangermanagerReq;
        json pickupjson = check <json>pickup;
        passangermanagerReq.setJsonPayload(pickupjson);
        http:Response passangerResponse= check passengerMgtEP -> post("/claims", request = passangermanagerReq);
        json passangerResponseJSON = check passangerResponse.getJsonPayload();

        // Dispatch to the dispatcher service
        // Create a JMS message
        jms:Message queueMessage = check jmsSession.createTextMessage(passangerResponseJSON.toString());
            // Send the message to the JMS queue
        
        
        log:printInfo("Hand over to the trip dispatcher to coordinate driver and  passenger:");
        _ = jmsTripDispatchOrder -> send(queueMessage);

        log:printInfo("passanger-magement response:"+passangerResponseJSON.toString());
        // Send response to the user
        responseMessage = {"Message":"Trip information received"};
        response.setJsonPayload(responseMessage);
        _ = caller -> respond(response);
    }

}
```

##### dispatcher.bal
```ballerina
import ballerina/log;
import ballerina/io;
import ballerina/jms;
import ballerina/http;

type Trip{
    string tripID;
    Driver driver;
    Person person;
    string time;
};
type Driver{
    string driverID;
    string drivername;

};

type Person {
    string name;
    string address;
    string phonenumber;
    string registerID;
    string email;
};

// Initialize a JMS connection with the provider
// 'Apache ActiveMQ' has been used as the message broker
jms:Connection conn = new({
        initialContextFactory: "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
        providerUrl: "tcp://localhost:61616"
    });

// Client endpoint to communicate with Airline reservation service
endpoint http:Client courierEP {
    url:"http://localhost:9095/courier"
};

// Initialize a JMS session on top of the created connection
jms:Session jmsSession = new(conn, {
        // Optional property. Defaults to AUTO_ACKNOWLEDGE
        acknowledgementMode: "AUTO_ACKNOWLEDGE"
    });

// Initialize a queue receiver using the created session
endpoint jms:QueueReceiver jmsConsumer {
    session:jmsSession,
    queueName:"trip-dispatcher"
};



// Initialize a queue sender using the created session
endpoint jms:QueueSender jmsPassengerMgtNotifer {
    session:jmsSession,
    queueName:"trip-passanger-notify"
};

// Initialize a queue sender using the created session
endpoint jms:QueueSender jmsDriverMgtNotifer {
    session:jmsSession,
    queueName:"trip-driver-notify"
};

// JMS service that consumes messages from the JMS queue
// Bind the created consumer to the listener service
service<jms:Consumer> TripDispatcher bind jmsConsumer {
    // Triggered whenever an order is added to the 'OrderQueue'
    onMessage(endpoint consumer, jms:Message message) {
        log:printInfo("New Trip request ready to process from JMS Queue");
        http:Request orderToDeliver;
        // Retrieve the string payload using native function
        string personDetail = check message.getTextMessageContent();
        log:printInfo("person Details: " + personDetail);
        json person = <json>personDetail;
        orderToDeliver.setJsonPayload(person);
        string name = person.name.toString();
        //TODO fix the way to extract JSON path message from JMS message
        log:printInfo("name dd" + name);
        //TODO fix the way to extract JSON path message from JMS message
        Trip trip;
        
        trip.person.name = "dushan";
        trip.person.address="1817";
        trip.person.phonenumber="0014089881345";
        trip. person.email="dushan@wso2.com";
        trip.person.registerID="AB0001222";
        trip.driver.driverID="driver001";
        trip.driver.drivername="Adeel Sign";
        trip.tripID="0001";
        trip.time="2018 Jan 6 10:10:20";

        json tripjson = check <json>trip;

        log:printInfo("Driver Contacted Trip notification dispatching " + tripjson.toString());

        jms:Message queueMessage = check jmsSession.createTextMessage(tripjson.toString());
         
        fork {
            worker w1 {
                _ = jmsPassengerMgtNotifer -> send(queueMessage);
            }
            worker w2 {
               _ = jmsDriverMgtNotifer -> send(queueMessage);
            }

        } join (all) (map collector) {
    }

    }
    
}
```


##### passenger-management.bal
```ballerina
import ballerina/http;
import ballerina/log;
import ballerina/jms;

type Person {
    string name;
    string address;
    string phonenumber;
    string registerID;
    string email;
};

endpoint http:Listener listener {
    port:9091
};

// Initialize a JMS connection with the provider
// 'Apache ActiveMQ' has been used as the message broker
jms:Connection conn = new({
        initialContextFactory: "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
        providerUrl: "tcp://localhost:61616"
    });

// Initialize a JMS session on top of the created connection
jms:Session jmsSession = new(conn, {
        // Optional property. Defaults to AUTO_ACKNOWLEDGE
        acknowledgementMode: "AUTO_ACKNOWLEDGE"
    });

// Initialize a queue receiver using the created session
endpoint jms:QueueReceiver jmsConsumer {
    session:jmsSession,
    queueName:"trip-passanger-notify"
};


@http:ServiceConfig { basePath: "/passenger-management" }
service<http:Service> PassengerManagement bind listener {
    @http:ResourceConfig {
        path : "/claims",
        methods : ["POST"]
    }
    claims (endpoint caller, http:Request request) {
        Person person;
        // create an empty response object 
        http:Response res = new;
        // check will cause the service to send back an error 
        // if the payload is not JSON
        json responseMessage;
        json passangerInfoJSON = check request.getJsonPayload();
        
        log:printInfo("JSON :::" + passangerInfoJSON.toString());
        
        string customerName = passangerInfoJSON.customerName.toString();
        string address = passangerInfoJSON.address.toString();
        string contact = passangerInfoJSON.phonenumber.toString();

        person.name = customerName;
        person.address=address;
        person.phonenumber=contact;
        person.email="dushan@wso2.com";
        person.registerID="AB0001222";
        
        log:printInfo("customerName:" + customerName);
        log:printInfo("address:" + address);
        log:printInfo("contact:" + contact);

        //TODO prepare client response
        json personjson = check <json>person;
        responseMessage = personjson;
        log:printInfo("Passanger claims included in the response:" + personjson.toString());
        res.setJsonPayload(personjson);
        _ = caller -> respond (res);
    }
}


// JMS service that consumes messages from the JMS queue
// Bind the created consumer to the listener service
service<jms:Consumer> PassengerNotificationService bind jmsConsumer {
    // Triggered whenever an order is added to the 'OrderQueue'
    onMessage(endpoint consumer, jms:Message message) {
        log:printInfo("Trip information received passenger notification service notifying to the client");
        http:Request orderToDeliver;
        // Retrieve the string payload using native function
        string personDetail = check message.getTextMessageContent();
        log:printInfo("Trip Details:" + personDetail);
       
    }
    
}
```



##### driver-management.bal
```ballerina
import ballerina/http;
import ballerina/log;
import ballerina/jms;

type Person {
    string name;
    string address;
    string phonenumber;
    string registerID;
    string email;
};

endpoint http:Listener listener {
    port:9091
};

// Initialize a JMS connection with the provider
// 'Apache ActiveMQ' has been used as the message broker
jms:Connection conn = new({
        initialContextFactory: "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
        providerUrl: "tcp://localhost:61616"
    });

// Initialize a JMS session on top of the created connection
jms:Session jmsSession = new(conn, {
        // Optional property. Defaults to AUTO_ACKNOWLEDGE
        acknowledgementMode: "AUTO_ACKNOWLEDGE"
    });

// Initialize a queue receiver using the created session
endpoint jms:QueueReceiver jmsConsumer {
    session:jmsSession,
    queueName:"trip-driver-notify"
};




// JMS service that consumes messages from the JMS queue
// Bind the created consumer to the listener service
service<jms:Consumer> DriverNotificationService bind jmsConsumer {
    // Triggered whenever an order is added to the 'OrderQueue'
    onMessage(endpoint consumer, jms:Message message) {
        log:printInfo("Trip information received for Driver notification service notifying coordinating with Driver the trip info");
        http:Request orderToDeliver;
        // Retrieve the string payload using native function
        string personDetail = check message.getTextMessageContent();
        log:printInfo("Trip Details: " + personDetail);
       
    }
    
}
```

Similar to the JMS consumer, here also we require to provide JMS configuration details when defining the `jms:QueueSender` endpoint. We need to provide the JMS session and the queue to which the producer pushes the messages.   

To see the complete implementation of the above, refer to the [bookstore_service.bal](https://github.com/ballerina-guides/messaging-with-jms-queues/blob/master/guide/bookstore_service/bookstore_service.bal).

## Testing 

### Invoking the service

- First, start the `Apache ActiveMQ` server by entering the following command in a terminal from `<ActiveMQ_BIN_DIRECTORY>`.
```bash
   $ ./activemq start
```

- Navigate to `inter-prceosss-communication/guide` and run the following commands in separate terminals to start  `trip-management`,  `passenger-management` ,`dispatcher`, `driver-management` microservices.
```bash
   $ ballerina run trip-management
```
```bash
   $ ballerina run dispatcher
```

```bash
   $ ballerina run passenger-management
```

```bash
   $ ballerina run driver-management
```
   
- Then you may simulate requesting pickup call to the  `trip-management` .

```bash
   curl -v -X POST -d \
   '{"Name":"Dushan", "pickupaddr":"1817, Anchor Way, San Jose, US", 
   "ContactNumber":"0014089881345"}' \
   "http://localhost:9090/trip-manager/pickup" -H "Content-Type:application/json" 
```

  The Trip management sends a response similar to the following which is similar to the acknowledged to the pickup request followed by he will receiving
  trip notification information through different channel
```
< HTTP/1.1 200 OK
< content-type: application/json
< content-length: 39
< server: ballerina/0.970.1
< date: Fri, 8 Jun 2018 21:25:29 -0700
<
* Connection #0 to host localhost left intact
{"Message":"Trip information received"}
```

Driver management service would receive trip notification, the notification service can will render relevant information in the customer handheld device through notification

```
2018-06-08 22:08:38,361 INFO  [driver-management] - Trip Details: {"tripID":"0001","driver":{"driverID":"driver001","drivername":"Adeel Sign"},"person":{"name":"dushan","address":"1817","phonenumber":"0014089881345","registerID":"AB0001222","email":"dushan@wso2.com"},"time":"2018 Jan 6 10:10:20"}
```

Similarly passanger would get his trip notification
```
2018-06-08 22:08:38,346 INFO  [passenger-management] - Trip Details:{"tripID":"0001","driver":{"driverID":"driver001","drivername":"Adeel Sign"},"person":{"name":"dushan","address":"1817","phonenumber":"0014089881345","registerID":"AB0001222","email":"dushan@wso2.com"},"time":"2018 Jan 6 10:10:20"}
```

***************  SECTION BELOW HERE STILL UNDER CONTRUCTION *****

### Writing unit tests 

In Ballerina, the unit test cases should be in the same package inside a folder named as 'tests'.  When writing the test functions the below convention should be followed.
- Test functions should be annotated with `@test:Config`. See the below example.
```ballerina
   @test:Config
   function testResourcePlaceOrder() {
```
  
This guide contains unit test cases for each resource available in the 'bookstore_service' implemented above. 

To run the unit tests, navigate to `inter-prceosss-communication/guide/guide` and run the following command. 
```bash
   $ ballerina test
```

When running these unit tests, make sure that the `ActiveMQ` is up and running.

To check the implementation of the test file, refer to the [bookstore_service_test.bal](https://github.com/ballerina-guides/messaging-with-jms-queues/blob/master/guide/bookstore_service/tests/bookstore_service_test.bal).

## Deployment

Once you are done with the development, you can deploy the services using any of the methods that we listed below. 

### Deploying locally

As the first step, you can build Ballerina executable archives (.balx) of the services that we developed above. Navigate to `messaging-with-jms-queues/guide` and run the following command.
```bash
   $ ballerina build
```

- Once the .balx files are created inside the target folder, you can run them using the following command. 
```bash
   $ ballerina run target/<Exec_Archive_File_Name>
```

- The successful execution of a service will show us something similar to the following output.
```
   ballerina: initiating service(s) in 'target/bookstore_service.balx'
   ballerina: started HTTP/WS endpoint 0.0.0.0:9090
```

### Deploying on Docker

You can run the service that we developed above as a docker container.
As Ballerina platform includes [Ballerina_Docker_Extension](https://github.com/ballerinax/docker), which offers native support for running ballerina programs on containers,
you just need to put the corresponding docker annotations on your service code.
Since this guide requires `ActiveMQ` as a prerequisite, you need a couple of more steps to configure it in docker container.   

First let's see how to configure `ActiveMQ` in docker container.

- Initially, you need to pull the `ActiveMQ` docker image using the below command.
```bash
   $ docker pull webcenter/activemq
```

- Then launch the pulled image using the below command. This will start the `ActiveMQ` server in docker with default configurations.
```bash
   $ docker run -d --name='activemq' -it --rm -P webcenter/activemq:latest
```

- Check whether the `ActiveMQ` container is up and running using the following command.
```bash
   $ docker ps
```

Now let's see how we can deploy the `bookstore_service` we developed above on docker. We need to import  `ballerinax/docker` and use the annotation `@docker:Config` as shown below to enable docker image generation during the build time. 

##### bookstore_service.bal
```ballerina
import ballerinax/docker;
// Other imports

// Type definition for a book order

json[] bookInventory = ["Tom Jones", "The Rainbow", "Lolita", "Atonement", "Hamlet"];

// 'jms:Connection' definition

// 'jms:Session' definition

// 'jms:QueueSender' endpoint definition

@docker:Config {
    registry:"ballerina.guides.io",
    name:"bookstore_service",
    tag:"v1.0"
}

@docker:CopyFiles {
    files:[{source:<path_to_JMS_broker_jars>,
            target:"/ballerina/runtime/bre/lib"}]
}

@docker:Expose{}
endpoint http:Listener listener {
    port:9090
};

@http:ServiceConfig {basePath:"/bookstore"}
service<http:Service> bookstoreService bind listener {
``` 

- `@docker:Config` annotation is used to provide the basic docker image configurations for the sample. `@docker:CopyFiles` is used to copy the JMS broker jar files into the ballerina bre/lib folder. You can provide multiple files as an array to field `files` of CopyFiles docker annotation. `@docker:Expose {}` is used to expose the port. 

- Now you can build a Ballerina executable archive (.balx) of the service that we developed above, using the following command. This will also create the corresponding docker image using the docker annotations that you have configured above. Navigate to `messaging-with-jms-queues/guide` and run the following command.  
  
```
   $ballerina build bookstore_service
  
   Run following command to start docker container: 
   docker run -d -p 9090:9090 ballerina.guides.io/bookstore_service:v1.0
```

- Once you successfully build the docker image, you can run it with the `` docker run`` command that is shown in the previous step.  

```bash
   $ docker run -d -p 9090:9090 ballerina.guides.io/bookstore_service:v1.0
```

   Here we run the docker image with flag`` -p <host_port>:<container_port>`` so that we use the host port 9090 and the container port 9090. Therefore you can access the service through the host port. 

- Verify docker container is running with the use of `` $ docker ps``. The status of the docker container should be shown as 'Up'. 

- You can access the service using the same curl commands that we've used above.
```bash
   curl -v -X POST -d \
   '{"Name":"Bob", "Address":"20, Palm Grove, Colombo, Sri Lanka", 
   "ContactNumber":"+94777123456", "BookName":"The Rainbow"}' \
   "http://localhost:9090/bookstore/placeOrder" -H "Content-Type:application/json"
```


### Deploying on Kubernetes

- You can run the service that we developed above, on Kubernetes. The Ballerina language offers native support for running a ballerina programs on Kubernetes, with the use of Kubernetes annotations that you can include as part of your service code. Also, it will take care of the creation of the docker images. So you don't need to explicitly create docker images prior to deploying it on Kubernetes. Refer to [Ballerina_Kubernetes_Extension](https://github.com/ballerinax/kubernetes) for more details and samples on Kubernetes deployment with Ballerina. You can also find details on using Minikube to deploy Ballerina programs. 

- Since this guide requires `ActiveMQ` as a prerequisite, you need an additional step to create a pod for `ActiveMQ` and use it with our sample.  

- Navigate to `messaging-with-jms-queues/resources` directory and run the below command to create the ActiveMQ pod by creating a deployment and service for ActiveMQ. You can find the deployment descriptor and service descriptor in the `./resources/kubernetes` folder.
```bash
   $ kubectl create -f ./kubernetes/
```

- Now let's see how we can deploy the `bookstore_service` on Kubernetes. We need to import `` ballerinax/kubernetes; `` and use `` @kubernetes `` annotations as shown below to enable kubernetes deployment.

##### bookstore_service.bal

```ballerina

``` 

- Here we have used ``  @kubernetes:Deployment `` to specify the docker image name which will be created as part of building this service. `copyFiles` field is used to copy required JMS broker jar files into the ballerina bre/lib folder. You can provide multiple files as an array to this field.
- We have also specified `` @kubernetes:Service `` so that it will create a Kubernetes service, which will expose the Ballerina service that is running on a Pod.  
- In addition we have used `` @kubernetes:Ingress ``, which is the external interface to access your service (with path `` /`` and host name ``ballerina.guides.io``)

- Now you can build a Ballerina executable archive (.balx) of the service that we developed above, using the following command. This will also create the corresponding docker image and the Kubernetes artifacts using the Kubernetes annotations that you have configured above.
  
```
   $ ballerina build bookstore_service
  
   Run following command to deploy kubernetes artifacts:  
   kubectl apply -f ./target/bookstore_service/kubernetes
```

- You can verify that the docker image that we specified in `` @kubernetes:Deployment `` is created, by using `` docker images ``. 
- Also the Kubernetes artifacts related our service, will be generated under `` ./target/bookstore_service/kubernetes``. 
- Now you can create the Kubernetes deployment using:

```bash
   $ kubectl apply -f ./target/bookstore_service/kubernetes 
 
   deployment.extensions "ballerina-guides-bookstore-service" created
   ingress.extensions "ballerina-guides-bookstore-service" created
   service "ballerina-guides-bookstore-service" created
```

- You can verify Kubernetes deployment, service and ingress are running properly, by using following Kubernetes commands. 

```bash
   $ kubectl get service
   $ kubectl get deploy
   $ kubectl get pods
   $ kubectl get ingress
```

- If everything is successfully deployed, you can invoke the service either via Node port or ingress. 

Node Port:
```bash
     
```

Ingress:

Add `/etc/hosts` entry to match hostname. 
``` 
   127.0.0.1 ballerina.guides.io
```

Access the service 
```bash
   
```


## Observability 
Ballerina is by default observable. Meaning you can easily observe your services, resources, etc.
However, observability is disabled by default via configuration. Observability can be enabled by adding following configurations to `ballerina.conf` file in `messaging-with-jms-queues/guide/`.

```ballerina
[b7a.observability]

[b7a.observability.metrics]
# Flag to enable Metrics
enabled=true

[b7a.observability.tracing]
# Flag to enable Tracing
enabled=true
```

NOTE: The above configuration is the minimum configuration needed to enable tracing and metrics. With these configurations default values are load as the other configuration parameters of metrics and tracing.

### Tracing 

You can monitor ballerina services using in built tracing capabilities of Ballerina. We'll use [Jaeger](https://github.com/jaegertracing/jaeger) as the distributed tracing system.
Follow the following steps to use tracing with Ballerina.

- You can add the following configurations for tracing. Note that these configurations are optional if you already have the basic configuration in `ballerina.conf` as described above.
```
   [b7a.observability]

   [b7a.observability.tracing]
   enabled=true
   name="jaeger"

   [b7a.observability.tracing.jaeger]
   reporter.hostname="localhost"
   reporter.port=5775
   sampler.param=1.0
   sampler.type="const"
   reporter.flush.interval.ms=2000
   reporter.log.spans=true
   reporter.max.buffer.spans=1000
```

- Run Jaeger docker image using the following command
```bash
   $ docker run -d -p5775:5775/udp -p6831:6831/udp -p6832:6832/udp -p5778:5778 \
   -p16686:16686 p14268:14268 jaegertracing/all-in-one:latest
```

- Navigate to `messaging-with-jms-queues/guide` and run the `bookstore_service` using following command 
```
   $ ballerina run bookstore_service/
```

- Observe the tracing using Jaeger UI using following URL
```
   http://localhost:16686
```

### Metrics
Metrics and alerts are built-in with ballerina. We will use Prometheus as the monitoring tool.
Follow the below steps to set up Prometheus and view metrics for bookstore_service service.

- You can add the following configurations for metrics. Note that these configurations are optional if you already have the basic configuration in `ballerina.conf` as described under `Observability` section.

```ballerina
   [b7a.observability.metrics]
   enabled=true
   provider="micrometer"

   [b7a.observability.metrics.micrometer]
   registry.name="prometheus"

   [b7a.observability.metrics.prometheus]
   port=9700
   hostname="0.0.0.0"
   descriptions=false
   step="PT1M"
```

- Create a file `prometheus.yml` inside `/tmp/` location. Add the below configurations to the `prometheus.yml` file.
```
   global:
     scrape_interval:     15s
     evaluation_interval: 15s

   scrape_configs:
     - job_name: prometheus
       static_configs:
         - targets: ['172.17.0.1:9797']
```

   NOTE : Replace `172.17.0.1` if your local docker IP differs from `172.17.0.1`
   
- Run the Prometheus docker image using the following command
```
   $ docker run -p 19090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml \
   prom/prometheus
```
   
- You can access Prometheus at the following URL
```
   http://localhost:19090/
```

NOTE:  Ballerina will by default have following metrics for HTTP server connector. You can enter following expression in Prometheus UI
-  http_requests_total
-  http_response_time


### Logging

Ballerina has a log package for logging to the console. You can import ballerina/log package and start logging. The following section will describe how to search, analyze, and visualize logs in real time using Elastic Stack.

- Start the Ballerina Service with the following command from `messaging-with-jms-queues/guide`
```
   $ nohup ballerina run bookstore_service/ &>> ballerina.log&
```
   NOTE: This will write the console log to the `ballerina.log` file in the `messaging-with-jms-queues/guide` directory

- Start Elasticsearch using the following command

- Start Elasticsearch using the following command
```
   $ docker run -p 9200:9200 -p 9300:9300 -it -h elasticsearch --name \
   elasticsearch docker.elastic.co/elasticsearch/elasticsearch:6.2.2 
```

   NOTE: Linux users might need to run `sudo sysctl -w vm.max_map_count=262144` to increase `vm.max_map_count` 
   
- Start Kibana plugin for data visualization with Elasticsearch
```
   $ docker run -p 5601:5601 -h kibana --name kibana --link \
   elasticsearch:elasticsearch docker.elastic.co/kibana/kibana:6.2.2     
```

- Configure logstash to format the ballerina logs

i) Create a file named `logstash.conf` with the following content
```
input {  
 beats{ 
     port => 5044 
 }  
}

filter {  
 grok{  
     match => { 
	 "message" => "%{TIMESTAMP_ISO8601:date}%{SPACE}%{WORD:logLevel}%{SPACE}
	 \[%{GREEDYDATA:package}\]%{SPACE}\-%{SPACE}%{GREEDYDATA:logMessage}"
     }  
 }  
}   

output {  
 elasticsearch{  
     hosts => "elasticsearch:9200"  
     index => "store"  
     document_type => "store_logs"  
 }  
}  
```

ii) Save the above `logstash.conf` inside a directory named as `{SAMPLE_ROOT}\pipeline`
     
iii) Start the logstash container, replace the {SAMPLE_ROOT} with your directory name
     
```
$ docker run -h logstash --name logstash --link elasticsearch:elasticsearch \
-it --rm -v ~/{SAMPLE_ROOT}/pipeline:/usr/share/logstash/pipeline/ \
-p 5044:5044 docker.elastic.co/logstash/logstash:6.2.2
```
  
 - Configure filebeat to ship the ballerina logs
    
i) Create a file named `filebeat.yml` with the following content
```
filebeat.prospectors:
- type: log
  paths:
    - /usr/share/filebeat/ballerina.log
output.logstash:
  hosts: ["logstash:5044"]  
```
NOTE : Modify the ownership of filebeat.yml file using `$chmod go-w filebeat.yml` 

ii) Save the above `filebeat.yml` inside a directory named as `{SAMPLE_ROOT}\filebeat`   
        
iii) Start the logstash container, replace the {SAMPLE_ROOT} with your directory name
     
```
$ docker run -v {SAMPLE_ROOT}/filbeat/filebeat.yml:/usr/share/filebeat/filebeat.yml \
-v {SAMPLE_ROOT}/guide/bookstore_service/ballerina.log:/usr/share\
/filebeat/ballerina.log --link logstash:logstash docker.elastic.co/beats/filebeat:6.2.2
```
 
 - Access Kibana to visualize the logs using following URL
```
   http://localhost:5601 
```
  

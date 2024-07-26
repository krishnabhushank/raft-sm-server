# RAFT server with State Machine

To create a Spring Boot application that sets up a Raft server cluster using JRaft, you can follow these steps:

1. **Set Up Your Spring Boot Application:**

   First, create a new Spring Boot project. You can use Spring Initializr or your preferred IDE to generate the basic structure.

   ```shell
   spring init --dependencies=web,data-jpa your-project-name
   ```

2. **Add JRaft Dependencies:**

   Add JRaft dependencies to your `pom.xml` or `build.gradle` file. For Maven:

   ```xml
    <!-- Spring Boot dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
      
    <!-- jraft dependencies -->
    <dependency>
        <groupId>com.alipay.sofa</groupId>
        <artifactId>jraft-core</artifactId>
        <version>1.3.8</version>
    </dependency>
    <dependency>
        <groupId>com.alipay.sofa</groupId>
        <artifactId>jraft-rheakv-core</artifactId>
        <version>1.3.8</version>
    </dependency>
   ```

   Make sure to check for the latest version of JRaft on [Maven Central](https://search.maven.org/search?q=g:com.alipay.sofa%20a:jraft-core).

3. **Create Raft Node Configuration:**

   Define a configuration class for your Raft nodes. This class will configure the Raft environment and initialize the Raft nodes.

   ```java
   import com.alipay.sofa.jraft.JRaftUtils;
   import com.alipay.sofa.jraft.Node;
   import com.alipay.sofa.jraft.RaftGroupService;
   import com.alipay.sofa.jraft.conf.Configuration;
   import com.alipay.sofa.jraft.entity.PeerId;
   import com.alipay.sofa.jraft.option.NodeOptions;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;

   @Configuration
   public class RaftConfiguration {

       @Value("${raft.groupId}")
       private String groupId;

       @Value("${raft.dataPath}")
       private String dataPath;

       @Value("${raft.serverAddress}")
       private String serverAddress;

       @Value("${raft.initialServers}")
       private String initialServers;

       @Bean
       public Node raftNode() {
           NodeOptions nodeOptions = new NodeOptions();
           nodeOptions.setElectionTimeoutMs(5000);
           nodeOptions.setDisableCli(false);

           // Define Raft group
           Configuration conf = JRaftUtils.getConfiguration(initialServers);
           nodeOptions.setInitialConf(conf);

           PeerId serverId = JRaftUtils.getPeerId(serverAddress);
           RaftGroupService raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, dataPath);
           return raftGroupService.start();
       }
    @Bean
    public Node raftNode() {
        // Define the Raft group and node configurations
        String groupId = "raft-group";
        String currentPeerIdStr = "localhost:8081"; // Update with the current pod's address
        String initialConfStr = "localhost:8081,localhost:8082,localhost:8083"; // Update with the cluster addresses

        PeerId currentPeerId = new PeerId();
        currentPeerId.parse(currentPeerIdStr);
        Configuration conf = new Configuration();
        if (!conf.parse(initialConfStr)) {
            throw new IllegalArgumentException("Invalid initial configuration: " + initialConfStr);
        }

        NodeOptions nodeOptions = new NodeOptions();
        nodeOptions.setElectionTimeoutMs(1000);
        nodeOptions.setLogUri("data/log");
        nodeOptions.setRaftMetaUri("data/raft_meta");
        nodeOptions.setSnapshotUri("data/snapshot");
        nodeOptions.setInitialConf(conf);

        // Start the Raft node
        RaftGroupService raftGroupService = new RaftGroupService(groupId, currentPeerId, nodeOptions);
        return raftGroupService.start();
    }
   }
   ```

4. **Application Properties:**

   Define the properties in your `application.properties` or `application.yml` file to configure the Raft nodes:

   ```properties
   raft.groupId=example_group
   raft.dataPath=/path/to/data
   raft.serverAddress=localhost:8081
   raft.initialServers=localhost:8081,localhost:8082,localhost:8083
   ```

5. **Controller to Test Raft Cluster:**

   Create a simple controller to interact with the Raft cluster.

   ```java
   import com.alipay.sofa.jraft.Node;
   import com.alipay.sofa.jraft.entity.PeerId;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.*;
   
   @RestController
   public class RaftController {
   
       @Autowired
       private Node raftNode;
   
       @GetMapping("/isLeader")
       public String isLeader() {
           if (raftNode != null && raftNode.isLeader()) {
               return "true";
           } else {
               return "false";
           }
       }
   
       @GetMapping("/health")
       public String health() {
           if (raftNode != null) {
               return "Node is healthy";
           } else {
               return "Node is unhealthy";
           }
       }
   
       @GetMapping("/leader")
       public String leader() {
           if (raftNode != null) {
               PeerId leader = raftNode.getLeaderId();
               if (leader != null) {
                   return leader.toString();
               } else {
                   return "No leader";
               }
           } else {
               return "Node is not part of the cluster";
           }
       }
   
       @GetMapping("/configuration")
       public String configuration() {
           if (raftNode != null) {
               return raftNode.getOptions().getInitialConf().toString();
           } else {
               return "Node is not part of the cluster";
           }
       }
   
       @GetMapping("/role")
       public String role() {
           if (raftNode != null) {
               if (raftNode.isLeader()) {
                   return "leader";
               } else if (raftNode.isFollower()) {
                   return "follower";
               } else {
                   return "candidate";
               }
           } else {
               return "Node is not part of the cluster";
           }
       }
   
       @PostMapping("/addPeer")
       public String addPeer(@RequestParam String peer) {
           if (raftNode != null && peer != null) {
               PeerId newPeer = new PeerId();
               newPeer.parse(peer);
               raftNode.addPeer(newPeer, status -> {
                   if (status.isOk()) {
                       return "Peer added successfully";
                   } else {
                       return "Failed to add peer: " + status.getErrorMsg();
                   }
               });
               return "Adding peer...";
           } else {
               return "Invalid request";
           }
       }
   
       @PostMapping("/removePeer")
       public String removePeer(@RequestParam String peer) {
           if (raftNode != null && peer != null) {
               PeerId removePeer = new PeerId();
               removePeer.parse(peer);
               raftNode.removePeer(removePeer, status -> {
                   if (status.isOk()) {
                       return "Peer removed successfully";
                   } else {
                       return "Failed to remove peer: " + status.getErrorMsg();
                   }
               });
               return "Removing peer...";
           } else {
               return "Invalid request";
           }
       }
   
       @GetMapping("/metrics")
       public String metrics() {
           if (raftNode != null) {
               StringBuilder metrics = new StringBuilder();
               metrics.append("Term: ").append(raftNode.getCurrentTerm()).append("\n");
               metrics.append("Last Log Index: ").append(raftNode.getLastLogIndex()).append("\n");
               metrics.append("Committed Index: ").append(raftNode.getCommittedIndex()).append("\n");
               metrics.append("Election Timeout: ").append(raftNode.getOptions().getElectionTimeoutMs()).append(" ms\n");
               return metrics.toString();
           } else {
               return "Node is not part of the cluster";
           }
       }
   }
   ```

6. **Run the Application:**

   You need to start multiple instances of your Spring Boot application to form the Raft cluster. Each instance should have a unique `serverAddress` in the `application.properties` file.

   For example:
   
   ```properties
   # Instance 1
   server.port=8081
   raft.serverAddress=localhost:8081
   raft.initialServers=localhost:8081,localhost:8082,localhost:8083

   # Instance 2
   server.port=8082
   raft.serverAddress=localhost:8082
   raft.initialServers=localhost:8081,localhost:8082,localhost:8083

   # Instance 3
   server.port=8083
   raft.serverAddress=localhost:8083
   raft.initialServers=localhost:8081,localhost:8082,localhost:8083
   ```

By following these steps, you will have a basic Spring Boot application running a JRaft server cluster. You can enhance this setup by adding more endpoints to interact with the Raft cluster, handling state machines, and other features as needed.

## Deploy on GKE

To deploy a Spring Boot-based Raft server cluster on Google Kubernetes Engine (GKE), you need to containerize your application, create Kubernetes configurations, and deploy them to your GKE cluster. Here's a step-by-step guide:

### Step 1: Containerize Your Application

1. **Create a Dockerfile:**

   Create a `Dockerfile` in the root directory of your project:

   ```dockerfile
   FROM openjdk:17-jdk-slim
   VOLUME /tmp
   ARG JAR_FILE=target/*.jar
   COPY ${JAR_FILE} app.jar
   ENTRYPOINT ["java","-jar","/app.jar"]
   ```

2. **Build the Docker Image:**

   Build your Spring Boot application and the Docker image:

   ```shell
   ./mvnw clean package
   docker build -t gcr.io/YOUR_PROJECT_ID/raft-server:latest .
   ```

3. **Push the Docker Image to Google Container Registry:**

   Tag and push the image to Google Container Registry (GCR):

   ```shell
   docker tag raft-server:latest gcr.io/YOUR_PROJECT_ID/raft-server:latest
   docker push gcr.io/YOUR_PROJECT_ID/raft-server:latest
   ```

### Step 2: Create Kubernetes Configurations

1. **Create a Deployment Configuration:**

   Create a `deployment.yaml` file to define the deployment of your Raft server nodes:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: raft-server
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: raft-server
     template:
       metadata:
         labels:
           app: raft-server
       spec:
         containers:
         - name: raft-server
           image: gcr.io/YOUR_PROJECT_ID/raft-server:latest
           ports:
           - containerPort: 8080
           env:
           - name: RAFT_GROUP_ID
             value: "example_group"
           - name: RAFT_DATA_PATH
             value: "/path/to/data"
           - name: RAFT_SERVER_ADDRESS
             valueFrom:
               fieldRef:
                 fieldPath: status.podIP
           - name: RAFT_INITIAL_SERVERS
             value: "RAFT_SERVER_SERVICE_NAME:8080"  # Replace with your service name
           volumeMounts:
           - mountPath: "/path/to/data"
             name: raft-data
         volumes:
         - name: raft-data
           emptyDir: {}
   ```

2. **Create a Service Configuration:**

   Create a `service.yaml` file to define a headless service for the Raft servers:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: raft-server
   spec:
     clusterIP: None
     selector:
       app: raft-server
     ports:
     - name: http
       port: 8080
       targetPort: 8080
   ```

### Step 3: Deploy to GKE

1. **Create a GKE Cluster:**

   If you haven't already, create a GKE cluster:

   ```shell
   gcloud container clusters create raft-cluster --num-nodes=3
   ```

2. **Deploy Your Application:**

   Apply the Kubernetes configurations to your GKE cluster:

   ```shell
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   ```

### Step 4: Update Application Configuration

Ensure your Spring Boot application reads the configuration from environment variables. Update your `RaftConfiguration` class to use these environment variables:

```java
import com.alipay.sofa.jraft.JRaftUtils;
import com.alipay.sofa.jraft.Node;
import com.alipay.sofa.jraft.RaftGroupService;
import com.alipay.sofa.jraft.conf.Configuration;
import com.alipay.sofa.jraft.entity.PeerId;
import com.alipay.sofa.jraft.option.NodeOptions;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RaftConfiguration {

    @Value("${RAFT_GROUP_ID}")
    private String groupId;

    @Value("${RAFT_DATA_PATH}")
    private String dataPath;

    @Value("${RAFT_SERVER_ADDRESS}")
    private String serverAddress;

    @Value("${RAFT_INITIAL_SERVERS}")
    private String initialServers;

    @Bean
    public Node raftNode() {
        NodeOptions nodeOptions = new NodeOptions();
        nodeOptions.setElectionTimeoutMs(5000);
        nodeOptions.setDisableCli(false);

        // Define Raft group
        Configuration conf = JRaftUtils.getConfiguration(initialServers);
        nodeOptions.setInitialConf(conf);

        PeerId serverId = JRaftUtils.getPeerId(serverAddress);
        RaftGroupService raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, dataPath);
        return raftGroupService.start();
    }
}
```

### Step 5: Access and Manage the Cluster

1. **Check the Status:**

   Verify that your pods are running correctly:

   ```shell
   kubectl get pods
   ```

2. **Access the Application:**

   You can port-forward to one of the pods to access the application:

   ```shell
   kubectl port-forward pod/<pod-name> 8080:8080
   ```

By following these steps, you should have a Spring Boot-based Raft server cluster running on GKE. Adjust the configurations and deployment as needed to fit your specific requirements.

## To Integrate the App with External App using REST APIs

To integrate your external application (like a Kafka Consumer) with your JRaft-based Raft server to maintain the state, you'll need to modify your JRaft application to expose APIs for state management and integrate with your external application. Here's how you can achieve this:

### Step 1: Define a State Machine for Your Raft Server

In JRaft, the state machine is where you define how state is applied based on log entries. Create a state machine class to handle your application's state.

```java
import com.alipay.sofa.jraft.FSMCaller;
import com.alipay.sofa.jraft.core.StateMachineAdapter;
import com.alipay.sofa.jraft.entity.LogEntry;
import com.alipay.sofa.jraft.entity.Task;
import com.alipay.sofa.jraft.util.Utils;
import java.util.concurrent.ConcurrentHashMap;

public class RaftStateMachine extends StateMachineAdapter {
    private final ConcurrentHashMap<String, String> state = new ConcurrentHashMap<>();

    @Override
    public void onApply(Iterator iter) {
        while (iter.hasNext()) {
            LogEntry entry = iter.next();
            byte[] data = entry.getData().array();
            String command = new String(data);
            String[] parts = command.split(":");
            if (parts.length == 2) {
                state.put(parts[0], parts[1]);
            }
            iter.next();
        }
    }

    public String get(String key) {
        return state.get(key);
    }

    public void apply(String key, String value, FSMCaller.TaskCallback callback) {
        Task task = new Task();
        task.setData(Utils.getBytes(key + ":" + value));
        task.setDone(callback);
        node.apply(task);
    }
}
```

### Step 2: Expose APIs to Interact with the Raft Cluster

Create a REST controller in your Spring Boot application to expose APIs for interacting with the Raft cluster.

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;

@RestController
@RequestMapping("/raft")
public class RaftController {

    private final RaftStateMachine stateMachine;

    @Autowired
    public RaftController(RaftStateMachine stateMachine) {
        this.stateMachine = stateMachine;
    }

    @PostMapping("/apply")
    public ResponseEntity<?> apply(@RequestParam String key, @RequestParam String value) {
        CompletableFuture<Void> future = new CompletableFuture<>();
        stateMachine.apply(key, value, new FSMCaller.TaskCallback() {
            @Override
            public void run(Status status) {
                if (status.isOk()) {
                    future.complete(null);
                } else {
                    future.completeExceptionally(new RuntimeException(status.toString()));
                }
            }
        });

        try {
            future.get();
            return ResponseEntity.ok("Applied successfully");
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Failed to apply: " + e.getMessage());
        }
    }

    @GetMapping("/get")
    public ResponseEntity<String> get(@RequestParam String key) {
        String value = stateMachine.get(key);
        if (value != null) {
            return ResponseEntity.ok(value);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
}
```

### Step 3: Update the Application Configuration

Update your Spring Boot configuration to include the state machine and update the Raft node initialization to use this state machine.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RaftConfiguration {

    @Value("${RAFT_GROUP_ID}")
    private String groupId;

    @Value("${RAFT_DATA_PATH}")
    private String dataPath;

    @Value("${RAFT_SERVER_ADDRESS}")
    private String serverAddress;

    @Value("${RAFT_INITIAL_SERVERS}")
    private String initialServers;

    @Bean
    public RaftStateMachine raftStateMachine() {
        return new RaftStateMachine();
    }

    @Bean
    public Node raftNode(RaftStateMachine stateMachine) {
        NodeOptions nodeOptions = new NodeOptions();
        nodeOptions.setElectionTimeoutMs(5000);
        nodeOptions.setDisableCli(false);
        nodeOptions.setFsm(stateMachine);

        // Define Raft group
        Configuration conf = JRaftUtils.getConfiguration(initialServers);
        nodeOptions.setInitialConf(conf);

        PeerId serverId = JRaftUtils.getPeerId(serverAddress);
        RaftGroupService raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, dataPath);
        return raftGroupService.start();
    }
}
```

### Step 4: Integrate Your Kafka Consumer with the Raft APIs

In your Kafka Consumer application, use the exposed REST APIs to apply state changes and retrieve state from the Raft cluster.

```java
import org.springframework.web.client.RestTemplate;
import org.springframework.http.ResponseEntity;

public class KafkaConsumer {
    private final RestTemplate restTemplate;
    private final String raftServerUrl;

    public KafkaConsumer(String raftServerUrl) {
        this.restTemplate = new RestTemplate();
        this.raftServerUrl = raftServerUrl;
    }

    public void processRecord(String key, String value) {
        ResponseEntity<String> response = restTemplate.postForEntity(raftServerUrl + "/raft/apply?key=" + key + "&value=" + value, null, String.class);
        if (response.getStatusCode().is2xxSuccessful()) {
            System.out.println("Successfully applied state: " + key + "=" + value);
        } else {
            System.err.println("Failed to apply state: " + key + "=" + value);
        }
    }

    public String getState(String key) {
        ResponseEntity<String> response = restTemplate.getForEntity(raftServerUrl + "/raft/get?key=" + key, String.class);
        if (response.getStatusCode().is2xxSuccessful()) {
            return response.getBody();
        } else {
            return null;
        }
    }
}
```

### Step 5: Deploy the Updated Application to GKE

Build and deploy the updated Docker image to GKE following the steps from the previous answer. Ensure the Kafka Consumer can reach the Raft server's REST API endpoints.

By following these steps, your Kafka Consumer can use the JRaft-based Raft server to maintain the state of the application. The Raft server exposes APIs for state management, which the Kafka Consumer calls to apply and retrieve state.

## To Use gRPC instead of REST APIs

To use gRPC instead of REST APIs for interaction between your Kafka Consumer and the JRaft server, you need to modify your JRaft server to expose gRPC services. Here’s how you can do it:

### Step 1: Add gRPC Dependencies

Add the necessary gRPC dependencies to your `pom.xml`:

```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.40.1</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.40.1</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.40.1</version>
</dependency>
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
```

### Step 2: Define gRPC Service

Create a `.proto` file that defines the gRPC service for your Raft server. Save this file as `raft.proto`.

```proto
syntax = "proto3";

option java_package = "com.example.raft";
option java_outer_classname = "RaftServiceProto";

service RaftService {
    rpc Apply (ApplyRequest) returns (ApplyResponse);
    rpc Get (GetRequest) returns (GetResponse);
}

message ApplyRequest {
    string key = 1;
    string value = 2;
}

message ApplyResponse {
    bool success = 1;
}

message GetRequest {
    string key = 1;
}

message GetResponse {
    string value = 1;
}
```

### Step 3: Generate gRPC Code

Generate the gRPC server and client code from your `.proto` file. You can use the `protoc` compiler for this:

```shell
protoc --java_out=src/main/java --grpc-java_out=src/main/java src/main/proto/raft.proto
```

Ensure you have the `protoc` compiler installed and set up correctly. You can also configure your `pom.xml` to automatically generate these files during the build process.

### Step 4: Implement gRPC Service in JRaft Server

Create a gRPC server in your Spring Boot application and implement the service methods.

```java
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.io.IOException;

@Service
public class RaftGrpcServer {

    private Server server;

    @Autowired
    private RaftStateMachine stateMachine;

    @PostConstruct
    public void start() throws IOException {
        server = ServerBuilder.forPort(9090)
                .addService(new RaftServiceImpl(stateMachine))
                .build()
                .start();
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.err.println("*** shutting down gRPC server since JVM is shutting down");
            RaftGrpcServer.this.stop();
            System.err.println("*** server shut down");
        }));
    }

    public void stop() {
        if (server != null) {
            server.shutdown();
        }
    }

    private static class RaftServiceImpl extends RaftServiceGrpc.RaftServiceImplBase {
        private final RaftStateMachine stateMachine;

        public RaftServiceImpl(RaftStateMachine stateMachine) {
            this.stateMachine = stateMachine;
        }

        @Override
        public void apply(ApplyRequest request, StreamObserver<ApplyResponse> responseObserver) {
            stateMachine.apply(request.getKey(), request.getValue(), status -> {
                ApplyResponse response = ApplyResponse.newBuilder().setSuccess(status.isOk()).build();
                responseObserver.onNext(response);
                responseObserver.onCompleted();
            });
        }

        @Override
        public void get(GetRequest request, StreamObserver<GetResponse> responseObserver) {
            String value = stateMachine.get(request.getKey());
            GetResponse response = GetResponse.newBuilder().setValue(value).build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        }
    }
}
```

### Step 5: Update the State Machine Class

Ensure your state machine can handle the apply requests properly. Your `RaftStateMachine` should have methods to apply state changes and retrieve state.

### Step 6: Deploy the Application

Build and deploy your Spring Boot application with the gRPC service to GKE. Follow the steps in the previous response to build the Docker image and apply Kubernetes configurations.

### Step 7: Implement the gRPC Client in Kafka Consumer

Create a gRPC client in your Kafka Consumer application to interact with the Raft server.

```java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;
import com.example.raft.RaftServiceGrpc;
import com.example.raft.ApplyRequest;
import com.example.raft.ApplyResponse;
import com.example.raft.GetRequest;
import com.example.raft.GetResponse;

public class KafkaConsumer {
    private final ManagedChannel channel;
    private final RaftServiceGrpc.RaftServiceBlockingStub blockingStub;

    public KafkaConsumer(String host, int port) {
        this.channel = ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()
                .build();
        this.blockingStub = RaftServiceGrpc.newBlockingStub(channel);
    }

    public void processRecord(String key, String value) {
        ApplyRequest request = ApplyRequest.newBuilder().setKey(key).setValue(value).build();
        ApplyResponse response;
        try {
            response = blockingStub.apply(request);
            if (response.getSuccess()) {
                System.out.println("Successfully applied state: " + key + "=" + value);
            } else {
                System.err.println("Failed to apply state: " + key + "=" + value);
            }
        } catch (StatusRuntimeException e) {
            System.err.println("RPC failed: " + e.getStatus());
        }
    }

    public String getState(String key) {
        GetRequest request = GetRequest.newBuilder().setKey(key).build();
        GetResponse response;
        try {
            response = blockingStub.get(request);
            return response.getValue();
        } catch (StatusRuntimeException e) {
            System.err.println("RPC failed: " + e.getStatus());
            return null;
        }
    }

    public void shutdown() {
        channel.shutdown();
    }
}
```

### Step 8: Deploy and Test

Deploy your Kafka Consumer application and ensure it can communicate with the Raft server via gRPC. Adjust the configurations and deployment steps as needed to fit your specific environment and requirements.

By following these steps, you can replace REST APIs with gRPC for interaction between your Kafka Consumer and the JRaft server, ensuring efficient and robust communication.

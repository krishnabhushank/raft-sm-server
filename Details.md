To enable leader election using Raft with Sofa Jraft in your Spring Boot application, you'll need to follow these steps:

1. **Set up Maven Dependencies**: Ensure you have the necessary dependencies in your `pom.xml`.
2. **Configure JRaft**: Set up the JRaft configuration.
3. **Initialize Raft Node**: Create and initialize the Raft node.
4. **Handle Leader Election Events**: Implement listeners to handle leader election events.

Here’s a step-by-step guide and code examples to help you get started:

### Step 1: Set up Maven Dependencies
Add the necessary dependencies in your `pom.xml`:

```xml
<dependencies>
    <!-- Sofa JRaft dependencies -->
    <dependency>
        <groupId>com.alipay.sofa</groupId>
        <artifactId>jraft-core</artifactId>
        <version>1.3.5</version>
    </dependency>
    <dependency>
        <groupId>com.alipay.sofa</groupId>
        <artifactId>jraft-rhea</artifactId>
        <version>1.3.5</version>
    </dependency>
    <!-- Other necessary dependencies -->
    <!-- Spring Boot dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <!-- etc. -->
</dependencies>
```

### Step 2: Configure JRaft
Create a configuration class for JRaft:

```java
import com.alipay.sofa.jraft.option.NodeOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JRaftConfig {

    @Bean
    public NodeOptions nodeOptions() {
        NodeOptions nodeOptions = new NodeOptions();
        nodeOptions.setElectionTimeoutMs(5000);
        nodeOptions.setDisableCli(false);
        nodeOptions.setDisableMetric(false);
        // Set other options as needed
        return nodeOptions;
    }
}
```

### Step 3: Initialize Raft Node
Create a service to initialize the Raft node:

```java
import com.alipay.sofa.jraft.JRaftUtils;
import com.alipay.sofa.jraft.Node;
import com.alipay.sofa.jraft.RaftGroupService;
import com.alipay.sofa.jraft.conf.Configuration;
import com.alipay.sofa.jraft.entity.PeerId;
import com.alipay.sofa.jraft.option.NodeOptions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class RaftNodeService {

    private final NodeOptions nodeOptions;
    private Node node;

    @Autowired
    public RaftNodeService(NodeOptions nodeOptions) {
        this.nodeOptions = nodeOptions;
    }

    public void start() {
        String groupId = "example_raft_group";
        PeerId serverId = JRaftUtils.getPeerId("localhost:8081");
        Configuration initConf = JRaftUtils.getConfiguration("localhost:8081,localhost:8082,localhost:8083");

        RaftGroupService raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, initConf);
        this.node = raftGroupService.start();
    }

    public Node getNode() {
        return this.node;
    }
}
```

### Step 4: Handle Leader Election Events
Implement a listener to handle leader election events:

```java
import com.alipay.sofa.jraft.Node;
import com.alipay.sofa.jraft.NodeManager;
import com.alipay.sofa.jraft.core.StateMachineAdapter;
import com.alipay.sofa.jraft.entity.LeaderChangeContext;
import com.alipay.sofa.jraft.entity.PeerId;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class LeaderElectionListener extends StateMachineAdapter {

    private final RaftNodeService raftNodeService;

    @Autowired
    public LeaderElectionListener(RaftNodeService raftNodeService) {
        this.raftNodeService = raftNodeService;
    }

    @PostConstruct
    public void init() {
        Node node = raftNodeService.getNode();
        if (node != null) {
            node.addStateMachine(this);
        }
    }

    @Override
    public void onLeaderStart(long term) {
        System.out.println("Node is now the leader. Term: " + term);
    }

    @Override
    public void onLeaderStop(LeaderChangeContext ctx) {
        System.out.println("Node is no longer the leader. Context: " + ctx);
    }

    @Override
    public void onStartFollowing(LeaderChangeContext ctx) {
        System.out.println("Node started following the leader. Context: " + ctx);
    }

    @Override
    public void onStopFollowing(LeaderChangeContext ctx) {
        System.out.println("Node stopped following the leader. Context: " + ctx);
    }
}
```

### Step 5: Start the Raft Node
Make sure to start the Raft node when your application starts:

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class RaftApplication {

    public static void main(String[] args) {
        SpringApplication.run(RaftApplication.class, args);
    }

    @Bean
    public CommandLineRunner commandLineRunner(RaftNodeService raftNodeService) {
        return args -> {
            raftNodeService.start();
        };
    }
}
```

### Summary
This setup includes:
1. **Maven dependencies** to include Sofa JRaft.
2. **JRaft configuration** to define node options.
3. **Raft node initialization** to start the Raft node.
4. **Leader election listener** to handle leader election events.
5. **Application startup** to ensure the Raft node is started.

Now your Spring Boot application is set up to perform leader election using Raft.

**Next Steps:**
- **a.** Add unit tests to verify the Raft node's behavior during leader election.
- **b.** Implement additional Raft features such as log replication.

- Certainly! Below are unit test cases for verifying the behavior of the Raft node during leader election in your Spring Boot application. We'll use JUnit and Mockito for creating these tests.

### Dependencies
Ensure you have the necessary dependencies for JUnit and Mockito in your `pom.xml`:

```xml
<dependencies>
    <!-- Other dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>4.0.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Test Class for RaftNodeService

Create a test class to verify the initialization and start behavior of the Raft node:

```java
import com.alipay.sofa.jraft.JRaftUtils;
import com.alipay.sofa.jraft.Node;
import com.alipay.sofa.jraft.RaftGroupService;
import com.alipay.sofa.jraft.conf.Configuration;
import com.alipay.sofa.jraft.entity.PeerId;
import com.alipay.sofa.jraft.option.NodeOptions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.mockito.Mockito.*;

public class RaftNodeServiceTest {

    @Mock
    private NodeOptions nodeOptions;

    @Mock
    private RaftGroupService raftGroupService;

    @Mock
    private Node node;

    @InjectMocks
    private RaftNodeService raftNodeService;

    @BeforeEach
    public void setUp() {
        MockitoAnnotations.openMocks(this);
        when(nodeOptions.copy()).thenReturn(nodeOptions);
    }

    @Test
    public void testStart() {
        String groupId = "example_raft_group";
        PeerId serverId = JRaftUtils.getPeerId("localhost:8081");
        Configuration initConf = JRaftUtils.getConfiguration("localhost:8081,localhost:8082,localhost:8083");

        when(raftGroupService.start()).thenReturn(node);

        raftNodeService.start();

        Node startedNode = raftNodeService.getNode();
        assertNotNull(startedNode);
        verify(raftGroupService, times(1)).start();
    }
}
```

### Test Class for LeaderElectionListener

Create a test class to verify the leader election event handling:

```java
import com.alipay.sofa.jraft.Node;
import com.alipay.sofa.jraft.core.StateMachine;
import com.alipay.sofa.jraft.entity.LeaderChangeContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import static org.mockito.Mockito.*;

public class LeaderElectionListenerTest {

    @Mock
    private RaftNodeService raftNodeService;

    @Mock
    private Node node;

    @InjectMocks
    private LeaderElectionListener leaderElectionListener;

    @BeforeEach
    public void setUp() {
        MockitoAnnotations.openMocks(this);
        when(raftNodeService.getNode()).thenReturn(node);
    }

    @Test
    public void testOnLeaderStart() {
        leaderElectionListener.onLeaderStart(1L);
        verify(raftNodeService, times(1)).getNode();
    }

    @Test
    public void testOnLeaderStop() {
        LeaderChangeContext ctx = new LeaderChangeContext();
        leaderElectionListener.onLeaderStop(ctx);
        verify(raftNodeService, times(1)).getNode();
    }

    @Test
    public void testOnStartFollowing() {
        LeaderChangeContext ctx = new LeaderChangeContext();
        leaderElectionListener.onStartFollowing(ctx);
        verify(raftNodeService, times(1)).getNode();
    }

    @Test
    public void testOnStopFollowing() {
        LeaderChangeContext ctx = new LeaderChangeContext();
        leaderElectionListener.onStopFollowing(ctx);
        verify(raftNodeService, times(1)).getNode();
    }
}
```

### Summary
The provided unit tests cover:
- **RaftNodeServiceTest**: Verifying that the Raft node starts correctly and the node is not null after starting.
- **LeaderElectionListenerTest**: Ensuring the leader election event handling methods are called correctly.

Make sure to place these test classes in the `src/test/java` directory of your Spring Boot application.

**Next Steps:**
- **a.** Enhance the tests by adding more specific assertions and mock behaviors.
- **b.** Integrate these tests into your CI/CD pipeline for automated testing.

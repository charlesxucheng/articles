
#General
We can have nodes of different "roles" in the cluster.

A member node can have zero or more roles.
Specify roles via the akka.cluster.roles configuration setting: -Dakka.cluster.roles.0=player-registry

For our project, possible roles are:

- RequestDispatcher
-- Keep track of instance run requests
-- Keep track of available workers
-- Send requests to available workers
-- Keep track of requests being served (including request time-outs). 
-- Blacklist workers who may have issue serving requests
-- Trigger alert if requests are stuck (not completed after certain threshold)
- RequestExecutor
-- Receive request for one instance (or basis?) and execute it.
-- Inform requester after completing the request, or failing to complete (best effort basis)
- NotificationSender  
-- Sends out alert if any RequestExecutors are blacklisted, or requests are stuck

We should have enough seed nodes for the different roles: 
- To handle network partition:  minimally 3 RequestDispatchers, 3 RequestExecutors
- Otherwise: minimally 2 RequestDispatchers, 2 Workers

**Note:** Charles River team mentioned that CRD HA set up had issues with network partition before. Ideally STARS should handle network partition scenario.

RequestDispatchers are to handle cluster events (worker up/down) and request events. RequestDispatchers need some durable storage to keep its state. e.g. Akka Persistence, RDBMS

RequestDispatcher State: 
- Requests and their statues
- List of available workers
- List of black-listed workers

RequestDispatcher should be made a Cluster Singleton.
If not, all requests to RequestDispatchers need to be replicated across all nodes. To explore if CRDT is applicable in this case. 

Sending messages to RequestDispatcher from Controller - Currently implemented with creating an actor system in the controller JVM and join the cluster of the dispatchers and executors. There might be other ways. 

#High Level Process
Desired use case will be as follows:

-	There will be a cluster that consists of n RequestDispatcher (manager) nodes, m RequestExecutor (worker) nodes and k NotificationSender nodes (different roles)
-	Instance execution triggering is via the following means:
--	A user submit a request to kick off an instance via web frontend.
-- A instance is triggered by a scheduler
-	Since the manage is a cluster singleton, the request will be received by the currently active manager node in the cluster. The manager will record the request to some persistent storage (e.g. an RDBMS) for queuing. 
-	Periodically the manager try get jobs queued and locate available workers to run the jobs.
-	The job will take about 10 mins to run. It is resource intensive and will produce millions of rows of data, so it should be assigned to one worker only.
-	As the worker is resource intensive, likely there should be one worker instantiated on one node.
-	When the worker completes the run, it will inform any of the manager node, which will then update the status in the persistent storage. 
-	The instance may have multiple steps (e.g. run checks after engine execution). The manager will need to know what step to kick off after current step is completed.
-	Ideally the manager should inform the web front-end about job status changes. To explore how the notification can be done (email, browser notification, etc)
-	If the worker receives a message while it is busy, it should reject the request by informing the manager.
-	The worker may fail while running the job. If so it should inform the manager if it still can. The manager should retry the request after some time (up to a max number of tries).
-	The worker may take too long to complete the request, or simply become unresponsive. The manager will set a time out and if time out happens, it should retry the request with another worker, and the existing worker should abort if it is still possible.
-	If a worker has failed or timed out a number of times (for different requests), the manager should blacklist the worker and send alert to administrator for investigation and recovery.
-	If a request has been executed and failed or encountered time out for a number of times, the manager should blacklist the request and send alert to administrator and users.
-	If a request cannot be completed after a certain time, due to insufficient resource, resource availability, or a surge in requests, the manager should send alert to administrator and users.
- If the singleton manager fails, its state should be migrated to the new singleton manager instance. Either to use Akka Persistence, or reloading/retrieving all the state information at start up manually. e.g. Requests data can be retrieved from RDBMS, and available executors information can be polled from the cluster members. Requests that are still being executed need more effort to track. e.g send Query status message to executors, and executors will need to retry completion messages if they are not delivered.

#RequestExecutor
Need a layer of actors on top of current hierarchy. This layer will be responsible to instantiate the actors doing the actual instance run, and shut them down after the work is done (or time-out) and inform requester of the run completion (or failure). This layer will always be running once the ActorSystem is started.

#RequestDispatcher
Follow the Advanced Akka course examples. Define the states clearly. Take note of the system events and make use of them. Implemented as a normal Actor but possible to be an FSM too.

- ClusterEvent.*
- FSM.StateTimeout$ (Fired when a time-out happens. A time-out can be defined at the when() function. - GameEngine.java)
- Terminated (Death Watch - context().watch())
- Status.Failure (Fired when a time-out happens - for an ask operation? This message will be received instead of FSM.StateTimeout$ for a non-FSM actor.)

#Implementation Notes
**applicatoin.conf:**
```
	provider = akka.cluster.ClusterActorRefProvider

	cluster {
	    metrics.enabled = off // legacy metrics disabled
	    auto-down-unreachable-after = 5 seconds // should be longer for production
	    seed-nodes = ["akka.tcp://akkollect-system@localhost:2551",
      "akka.tcp://akkollect-system@localhost:2552"]
	}
```
**Cluster Event Subscribe/Unsubscribe**
```
  @Override
  public void preStart() throws Exception {
    Cluster.get(context().system()).subscribe(self(), ClusterEvent.initialStateAsEvents(), ClusterEvent.MemberEvent.class);
  }

  @Override
  public void postStop() {
    Cluster.get(context().system()).unsubscribe(self());
  }
```
**Log FSM state transitions**
```
 onTransition(new UnitPFBuilder<Tuple2<GameEngineState, GameEngineState>> ().match(Tuple2.class, t -> {
      log().debug("Transitioning from {} to {}", t._1(), t._2());
    }).build());
```
**Check node role**
```
  private boolean isPlayerRegistry(Member member) {
    return member.hasRole(PlayerRegistry.name);
  }
```
**Ask pattern (Can pass in ActorSelection)**
```
  @Override
  public void preStart() throws Exception {
    final Future<Object> ask = ask(playerRegistry, PlayerRegistry.GetPlayers.Instance, settings.tournament.askTimeout);
    pipe(ask, context().dispatcher()).to(self());
  }
```
**Cluster Aware Routers**
**Group router - look up routees on member nodes**
- Must create the routee actors separately
- Routee can be created after the router node
- Routee is created using the normal actor creation way (context().actorOf(props))
- Router node is created using context().actorOf(new FromConfig().props(), name);
- Router will look for routees specified in routees.paths from all nodes in the cluster.

**Pool router - create routees on member nodes**
- No need to create routee actors separately. They are auto created.
- Router node is created using context().actorOf(new FromConfig().props(), name);

**Circuit Breaker for preventing cascading failures in distributed systems**
- Akka Circuit Breaker pattern can be used to implement the dispatcher behavior of monitoring executor timeouts and blacklisting
- http://doc.akka.io/docs/akka/current/common/circuitbreaker.html
#Sequence Diagram
```sequence
InstanceExecutionRequestSubmissionController->InstanceExecRequestSubmissionService:submitRequest()
InstanceExecRequestSubmissionService->InstanceExecutionRequestRepo:createRequest()
InstanceExecutionRequestRepo-->>InstanceExecRequestSubmissionService:requestCreated
InstanceExecRequestSubmissionService->RequestDispatcher:executeRequest(request)
RequestDispatcher-->>InstanceExecRequestSubmissionService:requestReceived
RequestDispatcher->>RequestDispatcher:DispatchRequests
RequestDispatcher->InstanceExecutionRequestRepo:getReadyRequests()
InstanceExecutionRequestRepo-->>RequestDispatcher:readyRequests
RequestDispatcher->>RequestExecutor:BasisExecutionRequest
RequestDispatcher->RequestDispatcher:scheduleBasisExecutionTimeOut()
RequestDispatcher->RequestDispatcher:cancelExecutionTimeOut()
RequestExecutor->>RequestDispatcher:BasisExecutionCompleted
RequestExecutor->>RequestDispatcher:ExecutorBusy
RequestDispatcher->>RequestExecutor:PostCheckExecutionRequest
RequestExecutor->>RequestDispatcher:PostCheckExecutionCompleted
RequestExecutor->>RequestDispatcher:ExecutorBusy


```

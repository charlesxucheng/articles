
#General
We can have nodes of different "roles" in the cluster.

A member node can have zero or more roles.
Specify roles via the akka.cluster.roles configuration setting: -Dakka.cluster.roles.0=player-registry

For our project, possible roles are:

- Arbiters (Managers)
-- Keep track of instance run requests
-- Keep track of active workers and available workers
-- Send requests to available workers
-- Keep track of requests being served (including request time-outs). 
-- Blacklist workers who may have issue serving requests
- Probes (Workers)
-- Receive request for one instance (or basis?) and execute it.
-- Inform requester after completing the request, or failing to complete (best effort basis)
- Sentries (Notification Senders)  
-- Sends out alert if any Probes are blacklisted

We should have enough seed nodes for the different roles: 
- To handle network partition: 3 Arbiters, 3 Workers minimum
- Otherwise: 2 Arbiters, 2 Workers minimum

Arbiters are to handle cluster events (worker up/down) and request events. Arbiters need some durable storage to keep its state. Akka Persistence? 

Also requests to Arbiters need to be replicated across all nodes. To explore if CRDT is applicable in this case. 

Routing request to Arbiter is required. Use Cluster-ware router?

#Probe
Need a layer of actors on top of current hierarchy. This layer will be responsible to instantiate the actors doing the actual instance run, and shut them down after the work is done (or time-out) and inform requester of the run completion (or failure). This layer will always be running once the ActorSystem is started.

#Arbiter
Follow the Advanced Akka course examples and use FSM. Define the states clearly. Take note of the system events and make use of them. 

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

#### 2.2 Give the answer to the question about the existing problem with the current solution.

> Our solution is currently a bit too simple:
>
> All nodes have to be registered to the same cluster through the haproxy node. This can causes problem if the haproxy is not running or ready, then some node may not be able to join the cluster. Whereas, with Serf usage we want to be able to join the cluster at any time, through multiple machines, not just one haproxy like here.

#### 2.3 Give an explanation on how `Serf` is working. Read the official website to get more details about the `GOSSIP` protocol used in `Serf`. Try to find other solutions that can be used to solve similar situations where we need some auto-discovery mechanism.

> Wikipedia definition:
>
> *A **gossip protocol** is a procedure or process of computer peer-to-peer communication that is based on the way [epidemics](https://en.wikipedia.org/wiki/Epidemic) spread.[[1\]](https://en.wikipedia.org/wiki/Gossip_protocol#cite_note-1) Some [distributed systems](https://en.wikipedia.org/wiki/Distributed_computing) use peer-to-peer gossip to ensure that data is disseminated to all members of a group. Some ad-hoc networks have no central registry and the only way to spread common data is to rely on each member to pass it along to their neighbours.*
>
> So with Serf, there is a usage of gossip protocol to communicate within the nodes. With the peer-to-peer, all Serf agent exchange with each other to transmit their data like a virus in the cluster.
>
> For this each agent want to join (or create if no existing) the load-balancer. Then if a node arrrive/leave the cluster, with the GOSSIP protocol, the Serf agent will tell all the nodes with thoose 2 messages: Join intent and Leave intent
>
> Other solution for auto-discovery mechanism can be:
>
> (found with https://saipraveenblog.wordpress.com/2014/10/06/service-discovery-in-soamsa/)
>
> * Zookeeper
> * Netflix's Eureka
> * Consul.io
>
> (found with http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/)
>
> * Doozer
> * Etcd


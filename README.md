## mongodb-kubernetes-aws
### Why Kubernetes
 * Kubernetes makes a MongoDB cluster Highly Available. If a pod fails, the system will reschedule and restart it. 
 Effectively adding redundancy with minimal human intervention.
 
### MongoDB Cluster via Kubernetes on AWS 
* Docker containers are stateless. While, MongoDB database nodes have to be  stateful. 
In the event that a container faSetupils, and is rescheduled, it's undesirable for the data to be lost. 
To solve this, features such as the Volume abstraction in Kubernetes can be used to map what would otherwise be an ephemeral MongoDB data directory in the container to a persistent location where the data survives container failure and rescheduling.
* Persistence: In AWS this Data Persistence is achieved by using Elastic Block Storage.
* MongoDB database nodes within a replica set must communicate with each other – including after rescheduling. 
All of the nodes within a replica set must know the addresses of all of their peers. However, when a container is rescheduled, it is likely to be restarted with a different IP address. For example, all containers within a Kubernetes Pod share a single IP address, which changes when the pod is rescheduled. With Kubernetes, this can be handled by associating a Kubernetes Service with each MongoDB node, which uses the Kubernetes DNS service to provide a hostname for the service that remains constant through rescheduling.
* Once each of the individual MongoDB nodes is running (each within its own container), the replica set must be initialized and each node added. This is likely to require some additional logic beyond that offered by off the shelf orchestration tools. Specifically, one MongoDB node within the intended replica set must be used to execute the rs.initiate and rs.add commands.

### Howto Deploy on AWS
* Create Cluster using kops and kubectl on AWS.
```shell
export NODE_COUNT=3
kops create cluster --name=$CLUSTER_NAME --cloud=aws --master-size=t2.micro --node-size=t2.micro --node-count=$NODE_COUNT --vpc=$VPC_ID --network-cidr=$NETWORK_CIDR --zones=us-west-2a,us-west-2b,us-west-2c  --master-zones=us-west-2a,us-west-2b,us-west-2c 
```
* Availability Zones: If we want to distribute the master or nodes across multiple availability zones you need to add some parameters to the create clister cli.
For masters in multiple AZs specify `--master-zones=us-west-2a,us-west-2b,us-west-2c`. For nodes in multiple AZs specify for instance `--zones=us-west-2a,us-west-2b,us-west-2c`. 
You can then change the number of nodes you run total by passing `--node-count`. However, the nodes are not guaranteed to be distributed across all AZs. 
If you want to guarantee a count per zone, you need to create [multiple instance groups](https://github.com/kubernetes/kops/blob/master/docs/instance_groups.md) for your nodes, 
one per AZ (instance groups map to ASGs). You can tweak your nodes any time (i.e. after cluster creation).
* Get `VPC_ID` and `NETWORK_CIDR` from AWS Console. Details [here](https://github.com/kubernetes/kops/blob/master/docs/run_in_existing_vpc.md).
* Create a deployment. See [Example YML File](mongo-cluster-deployment.yml)
* Deploy
```
kubectl create -f $DEPLOYMENT_YML_FILE --record
```
* Deploy a LoadBalancer service to front this.
```shell
kubectl expose deployment $DEPLOYMENT_NAME --type=LoadBalancer --name=$SERVICE_NAME
```
* You dont need a replication controller as mentioned [here](https://www.mongodb.com/blog/post/running-mongodb-as-a-microservice-with-docker-and-kubernetes), since deployments already take care of that.
Deployments subsume the role of RCs.
* This is what it looks like. Instead of ReplicationController -- think a deployment.
<p align="center">
<img src="https://webassets.mongodb.com/_com_assets/cms/image00-f5bc4ecaf8.png?raw=true" width="450"/>
</p>
* After adding 3 replicas the cluster looks like this:
<p align="center">
<img src="https://webassets.mongodb.com/_com_assets/cms/image01-b1896be8f6.png">
</p>

* Enabling Replica Sets to Talk to each other.
* Connect to any one of the mongodb instances. Either use the public IP. Use `kubectl get svc` to get the services and the public IP.
However, if the cluster is not externally visible, log into the pods ` kubectl exec -it $POD_ID bash`
* run the inititate command:
```
rs.initiate()
```
* We need to edit the config of the replica set. Run `conf=rs.conf()`. 
This member should be known by the external IP address and the port. Add the primary as such:
```
conf.members[0].host="$EXTERNAL_IP_FIRST_MONGO_NODE:PORT"
```
* Override the configuration data using `rs.reconfig(conf)`
* Add other (two?) members (secondaries) of the replica set using external IP addresses:
```
rs.add($EXTERNAL_IP_SECOND_MONGO_NODE:PORT)
rs.add($EXTERNAL_IP_THIRD_MONGO_NODE:PORT)
```
* Check `rs.status`

##### Credits
* Image / Some Content Credit: [MongoDB Blog](https://www.mongodb.com/blog/post/running-mongodb-as-a-microservice-with-docker-and-kubernetes)
* [Video](https://www.mongodb.com/presentations/webinar-enabling-microservices-with-containers-orchestration-and-mongodb)
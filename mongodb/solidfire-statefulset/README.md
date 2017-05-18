
Following are instructions to deploy a MongoDB 3 replica-set on Kubernetes with StatefulSets and NetApp Trident for dynamic Storage provisioning on NetApp SolidFire.

Note: At this time (13.03.2017) Kubernetes StatefulSets are still in Beta. The SideCar Container used in the deployment is also unsupported.

For this deployment we need the following:
1. NetApp Trident installed and configured with SolidFire Backend, backend-solidfire.json
2. A Kubernetes Storage class for SolidFire Storage, storage-class-gold.yaml
3. A MongoDB headless service as defined in the mongo-solidfire-statefulsets.yaml
4. MongoDB StatefulSet as defined in the mongo-solidfire-statefulsets.yaml


We will start by installing and configuring NetApp Trident. Then we will create a Storage class for SolidFire Trident backend.
After that we can deploy the MongoDB using replica sets. At the end we should be able to connect to MongoDB in our app.

## Install and Configure NetApp Trident

1. Ensure a Kubernetes 1.5+
2. Make sure iscsi utils are available on the kubernetes nodes. See more instructions [here](https://github.com/NetApp/netappdvp#configuring-your-docker-host-for-nfs-or-iscsi).
3. On your SolidFire system, create a volume access group (VAG) named trident and place the IQN of each node in the cluster into it.
4. Download and untar [Trident installer bundle](https://github.com/NetApp/trident/releases/download/v1.0/trident-installer-1.0.tar.gz)
4. Configure a storage backend from which Trident will provision its volumes. Edit file backend-solidfire.json and copy it to setup/backend.json
5. Run Trident installer with <code>./install_trident.sh</code>
6. Wait until deployment is complete. You should see Trident deployment when you execute <code>kubectl get deployment</code> and trident pod when you execute <code>kubectl get pods</code>
7. If you face issues check this troubleshooting section
8. Register you backend with Trident: <code>cat setup/backend.json | kubectl exec -i <trident-pod-name> -- post.sh backend</code>
9. Edit storage-class-gold.yaml to configure your storage class. Make sure you update requiredStorage param.
10. Create storage class in kubernetes: <code>kubectl create -f storage-class-gold.yaml</code>
11. Hurray! Trident is installed and configured.

Find detailed steps and information to install and configure Trident [here](https://github.com/NetApp/trident).

## Deploy MongoDB

Now that we have Trident installed and setup and a storageclass for SolidFire defined we can start with our MongoDB deployment.

With StatefulSets feature in Kubernetes it is not only simpler to deploy MongoDB with replica sets but also to scale it.

To deploy MongoDB with 3 replica sets using SolidFire configured with Gold class :

<code>kubectl create -f mongo-solidfire-statefulsets.yaml</code>

*Note: If using Openshift you must give the serviceaccount additional privileges to list pods as required by the sidecar.  Giving admin access is one way to do it:* `oadm policy add-cluster-role-to-user admin system:serviceaccount:YOUR-PROJECT-NAME:default`

This YAML will deploy a headless service and the MongoDb StatefulSet with  this “sidecar” container which will configure the MongoDB replica set automatically. A “sidecar” is a helper container which helps the main container do its work.

Once you execute the deployment command you will see mongo pods come up 1 by 1. Finally the StatefulSet will create 3 mongo instances 1 for each replica set.

![alt tag](https://raw.githubusercontent.com/kapilarora/images/master/mongo-stateful-pods.png)

You will also notice that 3 PersistenceVolumeClaims have also been created.
![alt tag](https://raw.githubusercontent.com/kapilarora/images/master/mongo-stateful-pvc.png)

And the corresponding Volumes for each mongoDB replica set is also provisioned dynamically by Trident on SolidFire.
![alt tag](https://raw.githubusercontent.com/kapilarora/images/master/mongo-stateful-pv.png)

## Connect with your Database

Each pod in a StatefulSet mongo-n where n is 0,1,2.. and is backed by our mongo Headless Service.
All mongo instances will have a stable DNS name like: <code>&lt;pod-name&gt;.&lt;service-name&gt;</code>

This means the DNS names for the MongoDB replica set are:

* mongo-0.mongo
* mongo-1.mongo
* mongo-2.mongo

You can use these names directly in the connection string URI of your app.

In this case, the connection string URI would be:

<code>“mongodb://mongo-0.mongo,mongo-1.mongo,mongo-2.mongo:27017/dbname_?”</code>

## Scaling

Here is how you can scale your cluster to 5 replica set:

kubectl scale --replicas=5 statefulset mongo

Once the pods are up and running you can use the new instances in your MongoDB connection string. namely, mongo-3.mongo and mongo-4.mongo.

## Clean-up

* Delete the StatefulSet:
<code>kubectl delete statefulset mongo</code>
* Delete the Service:
<code>kubectl delete svc mongo</code>
* Delete the Volumes:
<code>kubectl delete pvc -l role=mongo</code>


## References
* http://blog.kubernetes.io/2017/01/running-mongodb-on-kubernetes-with-statefulsets.html
* https://github.com/NetApp/trident

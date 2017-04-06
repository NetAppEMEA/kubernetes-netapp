# Apache Spark with S3 Support

This example shows how to deploy a stateless Apache Spark cluster with S3 support on Kubernetes. This is based on the "official" [kubernetes/spark](https://github.com/kubernetes/kubernetes/tree/master/examples/spark) example, which also contains a few more details on the deployment steps.

## Deploying Spark on Kubernetes

Create a new namespace:

```
$ kubectl create -f namespace-spark-cluster.yaml
```

Configure `kubectl` to work with the new namespace:

```
$ CURRENT_CONTEXT=$(kubectl config view -o jsonpath='{.current-context}')
$ USER_NAME=$(kubectl config view -o jsonpath='{.contexts[?(@.name == "'"${CURRENT_CONTEXT}"'")].context.user}')
$ CLUSTER_NAME=$(kubectl config view -o jsonpath='{.contexts[?(@.name == "'"${CURRENT_CONTEXT}"'")].context.cluster}')
$ kubectl config set-context spark --namespace=spark-cluster --cluster=${CLUSTER_NAME} --user=${USER_NAME}
$ kubectl config use-context spark
```

Deploy the Spark master Replication Controller and Service:

```
$ kubectl create -f spark-master-controller.yaml
$ kubectl create -f spark-master-service.yaml
```

Next, start your Spark workers:

```
$ kubectl create -f spark-worker-controller.yaml
```

Let's wait until everything is up and running:

```
$ kubectl get all
NAME                               READY     STATUS    RESTARTS   AGE
po/spark-master-controller-5rgz2   1/1       Running   0          9m
po/spark-worker-controller-0pts6   1/1       Running   0          9m
po/spark-worker-controller-cq6ng   1/1       Running   0          9m

NAME                         DESIRED   CURRENT   READY     AGE
rc/spark-master-controller   1         1         1         9m
rc/spark-worker-controller   2         2         2         9m

NAME               CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
svc/spark-master   10.108.94.160   <none>        7077/TCP,8080/TCP   9m
```

## Running queries against S3

Now, let's fire up a Spark shell and try out some commands:

```
$ kubectl exec spark-master-controller-5rgz2 -it spark-shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://192.168.132.147:4040
Spark context available as 'sc' (master = spark://spark-master:7077, app id = app-20170405152342-0000).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.0
      /_/

Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_111)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

Excellent, now let's tell our Spark cluster the details of our S3 target, this will use https by default:

```
scala> sc.hadoopConfiguration.set("fs.s3a.endpoint", "s3.company.com:8082")
scala> sc.hadoopConfiguration.set("fs.s3a.access.key", "94IMPM0VXXXXXXXX")
scala> sc.hadoopConfiguration.set("fs.s3a.secret.key", "L+3B2xXXXXXXXXXXX")
scala> sc.hadoopConfiguration.set("fs.s3a.fast.upload", "true")
```

If you are using a self-signed certifcate (and you haven't put it in the JVM truststore), you can disable SSL certificate verification via:

```
scala> System.setProperty("com.amazonaws.sdk.disableCertChecking", "1")
```

However, please don't do this in production.
Now, let's load some data that is sitting in S3:

```
scala> val movies = sc.textFile("s3a://spark/movies.txt")
movies: org.apache.spark.rdd.RDD[String] = s3a://spark/movies.txt MapPartitionsRDD[1] at textFile at <console>:24

scala> movies.count()
res6: Long = 4245028
```

Looks like it is working, let's do some filtering and then write the results back to S3:

```
scala> val godfather_movies = movies.filter(line => line.contains("Godfather"))
scala> godfather_movies.saveAsTextFile("s3a://spark/godfather.txt")
```

Let's see what Spark wrote to our S3 bucket:

```
$ sgws s3 ls s3://spark/godfather.txt/ --profile spark
2017-04-05 17:46:34          0 _SUCCESS
2017-04-05 17:46:06       1619 part-00000
2017-04-05 17:46:13       2152 part-00001
2017-04-05 17:46:15       1189 part-00002
2017-04-05 17:46:29       6698 part-00003
2017-04-05 17:46:32        856 part-00004
2017-04-05 17:46:33       3565 part-00005
```

As you can see, Spark didn't write a single object, but rather chunked it over multiple objects. While this might not be desirable with a small dataset, it makes sense for larger ones. This is because the overall throughput for writing to S3 improves as all workers can write in parallel. Concating all objects would yield the complete dataset as a single textfile.

## Further notes

This setup is a just an inital introduction on getting S3 working with Apache Spark on Kubernetes. Getting insights out of your data is the next step, but also optimizing performance is an important topic. For example, using Spark's `parallelize` call to parallelize object reads can yield massive performance improvements over using a simple `sc.textFiles(s3a://spark/*)` as used in this example.

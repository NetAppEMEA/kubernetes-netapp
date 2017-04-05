all: spark
push: push-spark
.PHONY: push push-spark

TAG = 2.1.0

spark:
	docker build -t csiebler/spark-s3 .
	docker tag csiebler/spark-s3 csiebler/spark-s3:$(TAG)

push-spark: spark
	docker push csiebler/spark-s3
	docker push csiebler/spark-s3:$(TAG)

clean:
	docker rmi csiebler/spark-s3:$(TAG) || :
	docker rmi csiebler/spark-s3 || :

# Dubbo example with Nacos/ZK

1. Run Dubbo example with Nacos/ZK as registry server.
2. Reference: [https://github.com/binblee/dubbo-docker](https://github.com/binblee/dubbo-docker)

## Build the example images

```
$ mvn clean package
$ cd service-consumer
$ docker build -t consumer .
$ cd ../service-producer
$ docker build -t producer .
```

## Run the example with docker-compose

### Run with docker-compose
```
$ cd docker
$ docker-compose -f docker-compose-zk.yml up
$ docker-compose -f docker-compose-zk.yml down
$ docker-compose -f docker-compose-nacos.yml up
$ docker-compose -f docker-compose-nacos.yml down
```
* Add "-d" to run as background process.


### Verify the example on docker

1. **Access the Nacos admin console in the browser**

	It can show the registered services in web UI.
	
	```
	http://localhost:8848/nacos (nacos/nacos)
	```
	
2. **Access the consumer**
	
	The cosumer will call producer and return a string.

	```
	$ curl http://localhost:8899
	Greetings from Dubbo Docker
	```
	

## Run the example with K8S	
### Run with K8S
```
$ cd k8s
$ kubectl create ns dubbo
$ kubectl apply -f k8s-nacos.yml -n dubbo
$ kuectl delete -f k8s-nacos.yml -n dubbo
$ kubectl apply -f k8s-zk.yml -n dubbo
$ kuectl delete -f k8s-zk.yml -n dubbo
```

### Verify the example on K8S

1. **Access the Nacos admin console in the browser**

	It can show the registered services in web UI.
	
	```
	http://localhost:30848/nacos (nacos/nacos)
	```
	
2. **Access the consumer**
	
	The cosumer will call producer and return a string.

	```
	$ curl http://localhost:30899
	Greetings from Dubbo Docker
	```
	

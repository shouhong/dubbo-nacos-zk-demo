# Dubbo example with Nacos/ZK

1. Run Dubbo example with Nacos/ZK as registry server.
2. Reference: [https://github.com/binblee/dubbo-docker](https://github.com/binblee/dubbo-docker)

## 1. Test on local machine

```
## start nacos
$ ./run-nacos.sh

## compile project 
$ mvn clean package

## run producer
$ cd service-producer/target
$ java -jar dubbo-service-producer-1.0-SNAPSHOT.jar

## run consumer
$ cd service-consumer/target
$ java -jar dubbo-service-consumer-1.0-SNAPSHOT.jar

## verify
$ curl 127.0.0.1:8899
Greetings from Dubbo Docker

## stop nacos
$ ./stop-nacos.sh
```

## 2. Build the example images

```
$ mvn clean package
$ cd service-consumer
$ docker build -t consumer .
$ cd ../service-producer
$ docker build -t producer .
```

## 3. Run the example with docker-compose

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
	

## 4. Run the example with K8S	
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
	
## 5. Run the example with K8S + SOFAMesh

### Notes:
1. Reference: [https://github.com/sofastack/sofa-mesh/tree/x-protocol-quickstart/samples/e2e-dubbo/platform/kube](https://github.com/sofastack/sofa-mesh/tree/x-protocol-quickstart/samples/e2e-dubbo/platform/kube)
2. The SOFAMesh code on this website cannot support x-dubbo protocol properly. So the above exmaple cannot work.
3. The istio-pilot needs to be enhanced to support outbound configuration of x-dubbo. Tenxcloud internally did this enhancement. Below verification is based on the SOFAMesh version developed by Tenxcloud.
4. The consumer and producer images used in this example can replace the return IP from Nacos service discovery with an IP set in PROVIDER\_CLUSER\_IP env (if the env has value). The consumer can leverage this to get producer's service-name/clusterIP, so the service mesh side car (MSON) can control the traffic when parsing the cluster IP to pod IP.

### Deploy the example on K8S
```
$ cd sofa
$ istioctl kube-inject k8s-nacos-all-dubbo.yml > k8s-nacos-all-dubbo-istio.yml
$ kubectl create ns dubbo
$ kubectl apply -f k8s-nacos-all-dubbo-istio.yml -n dubbo
```

### Verify the example without service mesh control
```
$ curl http://localhost:30899
Greetings from Dubbo Docker -- V1
$ curl http://localhost:30899
Greetings from Dubbo Docker -- V2
...
## V1 and V2 appear in randomly
```

### Create the destionation rule
```
$ kubectl apply -f producer-destinationrule.yml -n dubbo
```

### Verify the example with route by version
```
$ kubectl apply -f producer-vs-v1.yml -n dubbo
$ curl http://localhost:30899
## Always return V1
$ kubectl delete -f producer-vs-v1.yml -n dubbo

$ kubectl delete -f producer-vs-v1.yml -n dubbo
$ kubectl apply -f producer-vs-v2.yml -n dubbo
$ curl http://localhost:30899
## Always return V2
$ kubectl delete -f producer-vs-v2.yml -n dubbo
```

### Verify the example with route by weight
```
$ kubectl apply -f producer-vs-weight.yml -n dubbo
$ curl http://localhost:30899
## The portation of V1 and V2 is about 20%:80%
$ kubectl delete -f producer-vs-weight.yml -n dubbo
```
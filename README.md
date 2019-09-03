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
$ istioctl kube-inject -f k8s-dubbo-nacos-all.yml > k8s-dubbo-nacos-all-istio.yml
$ kubectl create ns dubbo
$ kubectl apply -f k8s-dubbo-nacos-all-istio.yml -n dubbo
```

### Verify the example without service mesh control
```
$ curl http://localhost:30899
Greetings from Dubbo Docker -- V1
$ curl http://localhost:30899
Greetings from Dubbo Docker -- V2
...
## V1 and V2 appear in turn
```

### Create the destionation rule
```
$ kubectl apply -f dr-producer.yml -n dubbo
```

### Verify the example with route by version
```
$ kubectl apply -f vs-producer-v1.yml -n dubbo
$ curl http://localhost:30899
## Always return V1
$ kubectl delete -f vs-producer-v1.yml -n dubbo

$ kubectl apply -f vs-producer-v2.yml -n dubbo
$ curl http://localhost:30899
## Always return V2
$ kubectl delete -f vs-producer-v2.yml -n dubbo
```

### Verify the example with route by weight
```
$ kubectl apply -f vs-producer-weight.yml -n dubbo
$ curl http://localhost:30899
## The portation of V1 and V2 is about 20%:80%
$ kubectl delete -f vs-producer-weight.yml -n dubbo
```

### Enable mTLS
```
## Enable mTLS globally
$ kubectl apply -f mtls-enable-dubbo.yml -n dubbo

## Deploy a client pod for test
$ istioctl kube-inject -f sleep.yml > sleep-istio.yml
$ kubectl apply -f sleep-istio.yml -n dubbo

## Verify the mTLS has been configured successfully
$ istioctl authn tls-check <producer1_pod_id> -n dubbo | grep '\.dubbo.svc'
HOST:PORT                                                     STATUS       SERVER     CLIENT     AUTHN POLICY                  
consumer.dubbo.svc.cluster.local:8899                                        OK         mTLS       mTLS       default/dubbo     default/dubbo
nacos.dubbo.svc.cluster.local:8848                                           OK         mTLS       mTLS       default/dubbo     default/dubbo
producer.dubbo.svc.cluster.local:20880                                       OK         mTLS       mTLS       default/dubbo     default/dubbo
sleep.dubbo.svc.cluster.local:80                                             OK         mTLS       mTLS       default/dubbo     default/dubbo

## Verify the sample can be accessed successfully from sleep pod
$ curl http://consumer:8899
Greetings from Dubbo Docker -- V1

## Disable mTLS 
$ kubectl delete -f mtls-enable-dubbo.yml -n dubbo
```

### RBAC Test
```
## To use RBAC function, the mTLS needs to be enabled first.

## Create ClusterRbacConfig to enable authorization for producer service
$ kubectl apply -f rbac-config-producer.yml

## Verify the sample cannot work anymore, access from sleep pod (need wait some time for the change to take effect)
$ http http://consumer.dubbo:8899
HTTP/1.1 500 Internal Server Error
...

## Create ServieRole and ServiceRoleBinding to authorize default service account in dubbo namespace to access producer service
$ kubectl apply -f rbac-policy-producer.yml -n dubbo

## Verify the sample can work again (need wait some time for the change to take effect)
$ http http://consumer.dubbo:8899
Greetings from Dubbo Docker -- V1
```

### 场景1: 不使用 mTLS（HTTP）  
1. 部署测试服务

    ```
    $ kubectl apply -f deploy.yaml
    ```
    这会在 rbactest 空间下部署 sleep1, sleep2 和 httpbin 三个服务。
    
2. 在 rbactest 空间下禁用 mTLS 功能

    ```
    $ kubectl apply -f mtls-disable.yaml
    ```
    
3. 验证 rbactest 空间下的服务在 SERVER 和 CLIENT 的模式都是 HTTP。

    ```
    $ istioctl authn tls-check <sleep1-pod-id> -n rbactest | grep rbactest.svc
    httpbin.rbactest.svc.cluster.local:8000                                        OK           HTTP       HTTP       default/rbactest                mtls-test/rbactest
    ```
    
4. 确认从 sleep1 可以成功访问 httpbin

    ```
    $ kubectl exec -it <sleep1-pod-id> -n rbactest -c sleep1 sh
    / # curl httpbin:8000/ip
    {
      "origin": "127.0.0.1"
    }
    ```
    
### 场景2: 打开 mTLS（HTTP）
1. 打开 rbactest 空间下客户端 mTLS

    ```
    $ kubectl apply -f mtls-enable-client.yaml
    ```

2. 验证服务端和客户端的 mTLS 模式不一样，处于冲突状态

    ```
    $ istioctl authn tls-check <httpbin-pod-id> -n rbactest | grep rbactest.svc
    httpbin.rbactest.svc.cluster.local:8000                                      CONFLICT     HTTP       mTLS       default/rbactest     mtls-test/rbactest
    ```

3. 验证 sleep1 访问httpbin 失败，因为 sleep1 发出的请求加密了，而 httpbin 要求接收到的请求是未加密的

    ```
    $ kubectl exec -it <sleep1-pod-id> -n rbactest -c sleep1 sh
    / # curl httpbin:8000/ip -v
    *   Trying 10.111.97.14...
    * TCP_NODELAY set
    * Connected to httpbin (10.111.97.14) port 8000 (#0)
    > GET /ip HTTP/1.1
    > Host: httpbin:8000
    > User-Agent: curl/7.60.0
    > Accept: */*
    > 
    < HTTP/1.1 502 Bad Gateway
    < Date: Thu, 15 Aug 2019 08:45:36 GMT
    < Content-Length: 0
    < Host: 10.111.97.14:8000
    < User-Agent: curl/7.60.0
    < Accept: */*
    < 
    * Connection #0 to host httpbin left intact
    ```

4. 打开 rbactest 空间下服务端 mTLS

    ```
    $ kubectl apply -f mtls-enable-server.yaml
    ```
    
5. 验证服务端和客户端的 mTLS 模式一样，回到 OK 状态

    ```
    $ istioctl authn tls-check <httpbin-pod-id> -n rbactest | grep rbactest.svc
    httpbin.rbactest.svc.cluster.local:8000                                        OK           mTLS       mTLS       default/rbactest                mtls-test/rbactest
    ```

6. 验证 sleep1 可以通过加密方式访问 httpbin

    ```
    kubectl exec -it <sleep1-pod-id> -n rbactest -c sleep1 sh
    / # curl httpbin:8000/ip
    {
      "origin": "127.0.0.1"
    }
    ```
    
7. **注意**：验证步骤中必须先开启客户端 mTLS，再开启服务端 mTLS。如果反过来则会有问题，显现如下：

    ```
    1. 未开启 mTLS 情况下，可以正常访问
    2. 开启服务端 mTLS, 还是可以正常访问。（非期望的行为）
    3. 重启客户端或服务端 Pod 后，访问返回 “502 Bad Gateway” 错误。（期望的行为）
    ```
    * 原因是服务端的 mTLS 是作用在 Listener 上，对于已经建立的连接修改 Listener 的 mTLS 不会受影响。即修改服务端 mTLS 只对新建连接有效。
    * 实际使用中客户端和服务端 mTLS 会成对配置，不会单独只配置服务器端 mTLS，所以上面的问题可以忽略。
    
### 场景3: 使用 RBAC（HTTP）
1. 本场景在“场景2: 使用 mTLS (HTTP)” 的基础上进行

2. 对 rbactest 命名空间打开 RBAC 检查：

    ```
    $ kubectl apply -f rbac-enable-rbacconfig.yaml
    ```
    
3. 验证从 sleep1 容器中访问 httpbin 报 “403 Forbidden” 错误

    ```
    $ kubectl exec -it <sleep1-pod-id> -n rbactest -c sleep1 sh
    / # curl httpbin:8000/ip -v
    *   Trying 10.99.193.187...
    * TCP_NODELAY set
    * Connected to httpbin (10.99.193.187) port 8000 (#0)
    > GET /ip HTTP/1.1
    > Host: httpbin:8000
    > User-Agent: curl/7.60.0
    > Accept: */*
    > 
    < HTTP/1.1 403 Forbidden
    < Date: Tue, 03 Sep 2019 12:33:32 GMT
    < Content-Length: 0
    < Host: httpbin:8000
    < User-Agent: curl/7.60.0
    < Accept: */*
    < X-Mosn-Host: httpbin:8000
    < Authority: httpbin:8000
    < X-Mosn-Method: GET
    < X-Mosn-Path: /ip
    < 
    * Connection #0 to host httpbin left intact
    ```
    
4. 用 ServiceRole+ServiceRoleBinding 对 sleep1 的 sa（sleep1）授权可以访问 httpbin 服务

    ```
    kubectl apply -f rbac-authorize-sleep1.yaml
    ```
    
5. 验证从 sleep1 可以访问 httpbin

    ```
    $ kubectl exec -it <sleep1-pod-id> -n rbactest -c sleep1 sh
    / # curl httpbin:8000/ip
    {
      "origin": "127.0.0.1"
    }
    ```
    
6. 验证从 sleep2 访问 httpbin 仍然报 “403 Forbidden” 错误
    ```
    $ kubectl exec -it <sleep2-pod-id> -n rbactest -c sleep1 sh
    / # curl httpbin:8000/ip -v
    *   Trying 10.99.193.187...
    * TCP_NODELAY set
    * Connected to httpbin (10.99.193.187) port 8000 (#0)
    > GET /ip HTTP/1.1
    > Host: httpbin:8000
    > User-Agent: curl/7.60.0
    > Accept: */*
    > 
    < HTTP/1.1 403 Forbidden
    < Date: Tue, 03 Sep 2019 12:33:32 GMT
    < Content-Length: 0
    < Host: httpbin:8000
    < User-Agent: curl/7.60.0
    < Accept: */*
    < X-Mosn-Host: httpbin:8000
    < Authority: httpbin:8000
    < X-Mosn-Method: GET
    < X-Mosn-Path: /ip
    < 
    * Connection #0 to host httpbin left intact
    ```
    
### 场景4: 清理 HTTP 测试环境
1. 清理 HTTP 测试环境

    ```
    sh clean.sh
    ```


### 场景5: 不使用 mTLS（Dubbo）  

1. 部署测试服务

    ```
    $ kubectl apply -f k8s-dubbo-nacos-all.yaml
    ```
    这会在 rbactest 空间下部署 nacos, producer, consumer, sleep。producer 和 consumer 注册到 nacos；sleep 以 HTTP 协议调用 consumer，然后触发 consumer 以 dubbo 协议调用 producer。
    
2. 在 rbactest 空间下禁用 mTLS 功能

    ```
    $ kubectl apply -f mtls-disable.yaml
    ```
    
3. 验证 rbactest 空间下的服务在 SERVER 和 CLIENT 的模式都是 HTTP。

    ```
    $ istioctl authn tls-check <sleep-pod-id> -n rbactest | grep rbactest.svc
    consumer.rbactest.svc.cluster.local:8899                                     OK         HTTP       HTTP       default/rbactest     mtls-test/rbactest
    nacos.rbactest.svc.cluster.local:8848                                        OK         HTTP       HTTP       default/rbactest     mtls-test/rbactest
    producer.rbactest.svc.cluster.local:20880                                    OK         HTTP       HTTP       default/rbactest     mtls-test/rbactest
    ```
    
4. 确认从 sleep 访问 consumer 时，consumer 能成功访问 provider 返回数据。

    ```
    $ kubectl exec -it <sleep-pod-id> -n rbactest -c sleep sh
    / # curl consumer:8899
    Greetings from Dubbo Docker -- V1
    ```
    
### 场景6: 打开 mTLS（Dubbo）
1. 打开 rbactest 空间下 producer 服务的客户端 mTLS

    ```
    $ kubectl apply -f mtls-enable-client-producer.yaml
    ```

2. 验证服务端和客户端的 mTLS 模式不一样，处于冲突状态

    ```
    $ istioctl authn tls-check <nacos-pod-id> -n rbactest | grep rbactest.svc
    producer.rbactest.svc.cluster.local:20880                                    CONFLICT     HTTP       mTLS       default/rbactest     mtls-test/rbactest
    ```

3. 验证 sleep 访问 consumer 时，consumer 访问 provider 失败，因为 consumer 发出的请求加密了，而 producer 要求接收到的请求是未加密的

    ```
    $ kubectl exec -it <sleep-pod-id> -n rbactest -c sleep sh
    / # curl consumer:8899 -v
    * Rebuilt URL to: consumer:8899/
    *   Trying 10.101.151.115...
    * TCP_NODELAY set
    * Connected to consumer (10.101.151.115) port 8899 (#0)
    > GET / HTTP/1.1
    > Host: consumer:8899
    > User-Agent: curl/7.61.1
    > Accept: */*
    > 
    < HTTP/1.1 500 Internal Server Error
    < Date: Wed, 04 Sep 2019 08:09:12 GMT
    < Content-Type: application/json;charset=UTF-8
    < Content-Length: 1615
    < Connection: close
    < 
    {"timestamp":"2019-09-04T08:09:12.346+0000","status":500,"error":"Internal Server Error","message":"Failed to invoke the method say in the service com.example.service.Greetings. Tried 3 times of the providers [producer:20880] (1/1) from the registry nacos:8848 on the consumer 172.31.107.235 using the dubbo version 2.7.1. Last error is: Invoke remote method timeout. method: say, provider: dubbo://producer:20880/com.example.service.Greetings?anyhost=true&application=consumer-app&bean.name=com.example.service.Greetings&category=providers&check=false&default.deprecated=false&default.dynamic=false&default.generic=false&default.lazy=false&default.register=true&default.sticky=false&deprecated=false&dubbo=2.0.2&dynamic=false&generic=false&interface=com.example.service.Greetings&lazy=false&methods=say&path=com.example.service.Greetings&pid=1&protocol=dubbo&register=true&register.ip=172.31.107.235&release=2.7.1&remote.application=producer-app&remote.timestamp=1567583748520&revision=1.0-SNAPSHOT&side=consumer&sticky=false&timestamp=1567584534246, cause: Waiting server-side response timeout by scan timer. start time: 2019-09-04 08:09:11.337, end time: 2019-09-04 08:09:12.342, client elapsed: 1 ms, server elapsed: 1003 ms, timeout: 1000 ms, request: Request [id=14, version=2.0.2, twoway=true, event=false, broken=false, data=RpcInvocation [methodName=say, parameterTypes=[class java.lang.String], arguments=[Dubbo Docker], attachments={path=com.example.service.Greetings, interface=com.example.service.Greetings, version=0.0.0}]], channel: /172.31.107.235:32858 -> producer/10.104.114.205:20880","path":"/"}* Closing connection 0
    ```

4. 打开 rbactest 空间下 producer 服务的服务端 mTLS

    ```
    $ kubectl apply -f mtls-enable-server-producer.yaml
    ```
    
5. 验证服务端和客户端的 mTLS 模式一样，回到 OK 状态

    ```
    $ istioctl authn tls-check <sleep-pod-id> -n rbactest | grep rbactest.svc
    producer.rbactest.svc.cluster.local:20880                                    OK         mTLS       mTLS       enable-producer-tls/rbactest     mtls-test/rbactest
    ```

6. 验证 sleep 访问 consumer 时，consumer 可以通过加密方式成功访问 producer

    ```
    kubectl exec -it <sleep-pod-id> -n rbactest -c sleep sh
    / # curl consumer:8899
    Greetings from Dubbo Docker -- V1
    ```
    
### 场景7: 使用 RBAC（Dubbo）
1. 本场景在“场景6: 使用 mTLS (Dubbo)” 的基础上进行

2. 对 rbactest 命名空间打开 RBAC 检查：

    ```
    $ kubectl apply -f rbac-enable-rbacconfig.yaml
    ```
    
3. 验证从 sleep 容器中访问 consumer，然后 consumer 访问 provider 报 “403 Forbidden” 错误

    ```
    $ kubectl exec -it <sleep1-pod-id> -n rbactest -c sleep1 sh
    / # curl consumer:8899 -v
    * Rebuilt URL to: consumer:8899/
    *   Trying 10.101.151.115...
    * TCP_NODELAY set
    * Connected to consumer (10.101.151.115) port 8899 (#0)
    > GET / HTTP/1.1
    > Host: consumer:8899
    > User-Agent: curl/7.61.1
    > Accept: */*
    > 
    < HTTP/1.1 500 Internal Server Error
    < Date: Wed, 04 Sep 2019 09:12:35 GMT
    < Content-Type: application/json;charset=UTF-8
    < Content-Length: 1616
    < Connection: close
    < 
    {"timestamp":"2019-09-04T09:12:35.745+0000","status":500,"error":"Internal Server Error","message":"Failed to invoke the method say in the service com.example.service.Greetings. Tried 3 times of the providers [producer:20880] (1/1) from the registry nacos:8848 on the consumer 172.31.107.235 using the dubbo version 2.7.1. Last error is: Invoke remote method timeout. method: say, provider: dubbo://producer:20880/com.example.service.Greetings?anyhost=true&application=consumer-app&bean.name=com.example.service.Greetings&category=providers&check=false&default.deprecated=false&default.dynamic=false&default.generic=false&default.lazy=false&default.register=true&default.sticky=false&deprecated=false&dubbo=2.0.2&dynamic=false&generic=false&interface=com.example.service.Greetings&lazy=false&methods=say&path=com.example.service.Greetings&pid=1&protocol=dubbo&register=true&register.ip=172.31.107.235&release=2.7.1&remote.application=producer-app&remote.timestamp=1567585678107&revision=1.0-SNAPSHOT&side=consumer&sticky=false&timestamp=1567584534246, cause: Waiting server-side response timeout by scan timer. start time: 2019-09-04 09:12:34.722, end time: 2019-09-04 09:12:35.741, client elapsed: 0 ms, server elapsed: 1019 ms, timeout: 1000 ms, request: Request [id=108, version=2.0.2, twoway=true, event=false, broken=false, data=RpcInvocation [methodName=say, parameterTypes=[class java.lang.String], arguments=[Dubbo Docker], attachments={path=com.example.service.Greetings, interface=com.example.service.Greetings, version=0.0.0}]], channel: /172.31.107.235:32858 -> producer/10.104.114.205:20880","path":"/"}* Closing connection 0
    ```
    
 4. 用 ServiceRole+ServiceRoleBinding 对 consumer 的 sa（consumer）授权可以访问 producer 服务
 
    ```
    kubectl apply -f rbac-authorize-consumer.yaml
    ```
    
5. 验证从 sleep 访问 consuemr, 然后 consumer 可以正常访问 producer

    ```
    $ kubectl exec -it <sleep-pod-id> -n rbactest -c sleep1 sh
    / # curl consumer:8899
    Greetings from Dubbo Docker -- V1
    ```
    
### 场景8: 清理 Dubbo 测试环境
1. 清理 Dubbo 测试环境

    ```
    sh clean.sh
    ```

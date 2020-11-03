---
layout: post
title:  Infinispan Hot Rod client with springboot
date:   2020-11-04
categories: tech
tags: infinispan openshift datagrid springboot
---

This is a extended post from the previous [article](https://wohshon.github.io/tech/2020/11/03/cross-site-infinispan.html) on deploying a cross site infinispan cluster on OpenShift Container Platform (OCP). 

I want to talk a bit on the Hot Rod springboot client I wrote to test the setup. I like to keep some scripts and utility programs handy as my job requires me to test or demo the solutions I work with. In fact, quite a number of them (like AMQ, AMQ Streams, DataGrid) actually needs some form of clients to be interacting with them. That was my motivation for developing some commonly used [application clients](https://github.com/wohshon/application-clients) in various runtimes. I am still in the midst of building that up after AMQ and Datagrid, probably AMQ Streams (kafka) is next.  

Back to today's topic, the little utility springboot program basically uses [Hot Rod](https://infinispan.org/docs/dev/titles/hotrod_java/hotrod_java.html) to communicate with a infinispan cluster; and it exposes a suite of REST based API for end users to invoke the CRUD operations. Hot Rod is a binary based TCP protocol, that promised better performance and provides client side functionalities like loadbalancing and failover etc. 

We will focus on the connectivity as an **external** client (outside of openshift), and will discuss some of the gotchas and differences in a OCP deployment in terms of the Hot Rod connectivity. I will also share some of the changes required for **internal** clients, if we were to deploy the springboot app on OCP, calling the cluster via service endpoint

The repo of the client is [here](https://github.com/wohshon/application-clients/tree/master/rhdg-springboot). The README file should be able to get you going.

Some things to take note of:

#### Sample Code

- I wanted a working sample where Java Objects are used (String based samples are plenty out there), where a sample protobuf usecase can be shown. Again, I prefer to have working 'MVP's where I can extend later, rather than to worry about these details when I need to work on something bigger or more complex.
In this case, a `PersonEntity` is used as the domain object (name, email, age) , nothing fanciful but the writing of correct  protobuf schemas and syntax are out of the way.

#### Connectivity

- Most of the Hot Rod connectivity details are configured in `hotrod-client-ocp-route.properties` and `hotrod-client.properties` and as the names implied, one is more for OpenShift Container Platform (OCP) route based usecases.

The key configs you will need to be aware are

- `infinispan.client.hotrod.server_list`, the endpoint of your cluster, it accepts a semi-colon separated list for multiple instances to handle failover*

*This is another area I need to explore, traditionally, for non OCP deployments, a list refers to the different nodes in a cluster, with a route based endpoint, it already 'is' the entire cluster. My take is it does not makes sense to have multiple routes as endpoints here (it won't work). Like the non OCP deployment,  there needs to be some external help (like LB) needed to automatically fail over to a different cluster.


- `infinispan.client.hotrod.client_intelligence`, BASIC (no cluster aware), TOPOLOGY_AWARE (clients received updated topology), DISTRIBUTION_AWARE (Topology aware and stores consistent hash for keys).*

I tried the `TOPOLOGY_AWARE` client intelligence, noticed that the server info uses the POD IPs which external clients cannot access, so they will not be able to make use of this function and have to stick to the `BASIC` mode. 
How big will this be an impact? For most usecases, I would like to think that the kubernetes service will provide the topology awareness, and distribute the load according to number of nodes in the cluster (same goes for internal clients, which is not wise to use a POD IP as a endpoint). Seems like client side topology awareness and failover is no longer relevant here. So looks like is about leveraging hot rod's superior perforamnce vs text-based protocols. 

The `DISTRIBUTION_AWARE` will not work for external clients as well as it is topology aware. 

(I got another error in this mode :`Unable to convert property [%s] to an enum! `, will need to investigate this, for internal clients ) 


```
2020-11-04 10:25:37.112  INFO 25381 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004014: New server added(172.22.0.70:11222), adding to the pool.
2020-11-04 10:25:37.118  INFO 25381 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004014: New server added(172.22.1.77:11222), adding to the pool.
2020-11-04 10:25:37.123  INFO 25381 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004016: Server not in cluster anymore(example-infinispan-external-rhdg-cluster.apps.ocpcluster2.domain.com:443), removing from the pool.
2020-11-04 10:25:45.984  INFO 25381 --- [nio-8080-exec-2] c.r.a.client.rhdgspringboot.Controller   : Get : 001
2020-11-04 10:26:37.131  WARN 25381 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004015: Failed adding new server 172.22.0.70:11222

io.netty.channel.ConnectTimeoutException: connection timed out: /172.22.0.70:11222
	at io.netty.channel.epoll.AbstractEpollChannel$AbstractEpollUnsafe$2.run(AbstractEpollChannel.java:575) ~[netty-transport-native-epoll-4.1.51.Final-linux-x86_64.jar:4.1.51.Final]

```
- Bunch of authentication / authorization related configs: 
`infinispan.client.hotrod.sasl_mechanism`
`infinispan.client.hotrod.use_auth`
`infinispan.client.hotrod.auth_username`
`infinispan.client.hotrod.auth_password`
- For OCP route based connection, you will need to extract the certificate from the secret (see README) and point to it using `infinispan.client.hotrod.trust_store_path`

You will also need to specify the hostname of the route via `infinispan.client.hotrod.sni_host_name`

For more details, the apidocs are [here](https://docs.jboss.org/infinispan/10.1/apidocs/org/infinispan/client/hotrod/configuration/package-summary.html)


#### Running the app

Some things to take note:

1. You have access to a cluster.
2. Ensure your Hot Rod client properties are configured with the correct settings according to your environment. Refer to the previous section.
3. Update the `application.properties` file with
   -  `app.cacheName`, the name of the cache
   - `hotrod.properties`, the Hot Rod client properties file you are using

Running the app is straightforward, like any springboot app.

`./mvnw clean compile spring-boot:run`


```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.3.RELEASE)

2020-11-04 09:31:13.094  INFO 24396 --- [           main] c.r.a.c.r.RhdgSpringbootApplication      : Starting RhdgSpringbootApplication on ocpclient1.domain.com with PID 24396 (/home/wohshon/workspace/application-clients/rhdg-springboot/target/classes started by wohshon in /home/wohshon/workspace/application-clients/rhdg-springboot)
2020-11-04 09:31:13.096  INFO 24396 --- [           main] c.r.a.c.r.RhdgSpringbootApplication      : No active profile set, falling back to default profiles: default
2020-11-04 09:31:13.692  INFO 24396 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-11-04 09:31:13.699  INFO 24396 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-11-04 09:31:13.700  INFO 24396 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.37]
2020-11-04 09:31:13.754  INFO 24396 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-11-04 09:31:13.754  INFO 24396 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 614 ms
2020-11-04 09:31:13.901  INFO 24396 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-11-04 09:31:14.014  INFO 24396 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-11-04 09:31:14.022  INFO 24396 --- [           main] c.r.a.c.r.RhdgSpringbootApplication      : Started RhdgSpringbootApplication in 1.214 seconds (JVM running for 1.501)
```

Once the app is running, use the REST API endpoint to insert, get or remove entries, details are in the README file for the repo.

Inserting a value

```
$ curl -X PUT -H 'Content-type: application/json' -d '{"id":"001","name":"ws","email": "a@b.com"}' localhost:8080/api/put/001
{"id":"001","name":"ws","email":"a@b.com"}
```

Getting the value out 
```
$ curl -X GET -H 'Content-type: application/json'  localhost:8080/api/get/001
{"id":"001","name":"ws","email":"a@b.com"}
```

#### Testing as internal clients, running springboot on OCP

For details check out the `internal-client` branch. 

In summary:

- change the server list in hot rod connection properties file to point to service endpot, 

        infinispan.client.hotrod.server_list = example-infinispan.rhdg-cluster.svc.cluster.local:11222

- create a secret of the cert, the cluster still listens on tls , so hotrod needs this to be mounted into the Pod running springboot

        $ oc create secret generic dg-crt --from-file=/tmp/tls-2.crt

- deploy the app, dirty hack to generate the `deployment` object 

        $ oc new-app openjdk-11-rhel8:1.0~https://github.com/wohshon/application-clients#internal-client --context-dir=rhdg-springboot --name=client
        $ oc expose svc client

- patch the `deployment` object to include the secret in the pod as a volumemount.

```
    spec:
      containers:
      - image: image-registry.openshift-image-registry.svc:5000/apps/<truncated>
        imagePullPolicy: IfNotPresent
        name: client
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 8443
          protocol: TCP
        - containerPort: 8778
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /tmp/crt
          name: mydir
      volumes:
      - name: mydir
        secret:
          defaultMode: 420
          secretName: dg-crt
```

- try using the route

$ curl -X GET -H 'Content-type: application/json'  http://client-apps.apps.ocpcluster2.domain.com/api/get/001

#### Client Intelligence with internal clients

Just an observation, I swtiched over to `TOPOLOGY_AWARE` mode, and can see the topology info sent back to clients successfully.

Went a step further to increase the cluster size to 3, and can see the info coming back.



```
2020-11-04 05:49:37.730  INFO 1 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004006: Server sent new topology view (id=10, age=0) containing 2 addresses: [172.22.1.77:11222, 172.22.0.78:11222]
2020-11-04 05:49:37.730  INFO 1 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004014: New server added(172.22.1.77:11222), adding to the pool.
2020-11-04 05:49:37.740  INFO 1 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004014: New server added(172.22.0.78:11222), adding to the pool.
2020-11-04 05:49:37.749  INFO 1 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004016: Server not in cluster anymore(example-infinispan.rhdg-cluster.svc.cluster.local:11222), removing from the pool.
2020-11-04 05:49:54.439  INFO 1 --- [nio-8080-exec-3] c.r.a.client.rhdgspringboot.Controller   : Get : 003
2020-11-04 05:50:03.117  INFO 1 --- [nio-8080-exec-5] c.r.a.client.rhdgspringboot.Controller   : Get : 001
2020-11-04 05:52:51.466  INFO 1 --- [nio-8080-exec-7] c.r.a.client.rhdgspringboot.Controller   : Get : 001
2020-11-04 05:52:51.491  INFO 1 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004006: Server sent new topology view (id=14, age=0) containing 3 addresses: [172.22.1.77:11222, 172.22.2.65:11222, 172.22.0.78:11222]
2020-11-04 05:52:51.491  INFO 1 --- [-async-pool-1-1] org.infinispan.HOTROD                    : ISPN004014: New server added(172.22.2.65:11222), adding to the pool.
```

Thats all for now!


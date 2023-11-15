# RHSI Hackfest
## Use-case flexibility and security 
The use-case was picked to showcase the capabilities of RHSI extending secure communication as well as traceability using multiple Kubernetes clusters.
Even though, the LAB is not based upon Openshift it should be reproducible in a similar manner. Furthermore, it is important for me to provide these LAB's in a reproducible way that will run on anyone's hardware.

## Cluster setup 
Clone the repository with all necessary CRs from [github](https://github.com/michaelalang/rhsi-hackfest)
Please jump to the [kind Cluster setup](README.md#kind-Cluster-setup) section at the botton as the Clusters are considered not important.

## Installing RHSI in Gitops manner (but with a human controller)
**NOTE** it's expected that you have a working Cluster as configured in [kind Cluster setup](README.md#kind-Cluster-setup)

within the repository you will find a directory called `skupper`. It contains sub directories for various stages of the journey. Do not jump ahead as the behavior of the checks might differ if you do so.

* Deploy the RHSI operator:
    ```
    oc --context rhsi1 apply -f base/skupper-operator.yml
    oc --context rhsi2 apply -f base/skupper-operator.yml
    ```
* Wait until the operator has finished deploying.
* Verify that the deployment has been successful:
	```
	oc --context rhsi1 -n openshift-operators get pods -l rht.prod_name=Red_Hat_Service_Interconnect
	oc --context rhsi2 -n openshift-operators get pods -l rht.prod_name=Red_Hat_Service_Interconnect
	```
	* expected output
		```
		NAME                                       READY   STATUS    RESTARTS   AGE
		skupper-site-controller-7bd65fff79-7drcf   1/1     Running   0          23h
		```

### Initialize our RHSI sites
For the purpose of this lab the namespace is called `skupper`. If you do want to change that, please ensure to update all references to the name in each CR.

* Create the namespaces:
	```
	oc --context rhsi1 apply -f base/namespace.yml
	oc --context rhsi2 apply -f base/namespace.yml
	```
The CR already has the required label for Openshift ServiceMesh in mode `cluster` to be picked up. If you still want to utilize `tenant` based OSSM, ensure to add the `ServiceMeshMember` or `ServiceMeshMemberRole` accordingly.

* Deploy the RHSI `skupper-site` configmap:
	```
	oc --context rhsi1 -n skupper apply -f overlays/rhsi1/skupper-site.yml
	oc --context rhsi2 -n skupper apply -f overlays/rhsi2/skupper-site.yml
	```
**NOTE** The site definition is based upon NodePort due to the lack of Routes in kind Kubernetes. The adjustments for an Openshift Cluster with ServiceMesh enabled and exposed through OSSM virtualservices is not covered in that Lab.

#### adjustments to the Operator based deployments and service mappings to NodePorts
Since the kind Kubernetes cluster is based upon Kubernetes 1.26 the `runAsNonRoot: true` needs to be adjusted to get the deployments up-and-running.

* Update all deployments in the namespace:
	```
	oc --context rhsi1 -n skupper get deploy/skupper-router -o yaml | \
	  sed -e " s#runAsNonRoot: true#runAsNonRoot: false#; " | \
	     oc --context rhsi1 -n skupper apply -f - 

	oc --context rhsi2 -n skupper get deploy/skupper-router -o yaml | \
	  sed -e " s#runAsNonRoot: true#runAsNonRoot: false#; " | \
	     oc --context rhsi2 -n skupper apply -f - 
	```

* Adjust the `nodePorts` of skupper service :
	```
	oc --context rhsi1 -n skupper edit service skupper
	...
	spec:
	  ...
	  ports:
	  - name: metrics
	    nodePort: 32601
	    port: 8010
	    protocol: TCP
	    targetPort: 8010
	  - name: rest-api
	    nodePort: 31507
	    port: 8080
	    protocol: TCP
	    targetPort: 8080
	  - name: claims
	    nodePort: 32602
	    port: 8081
	    protocol: TCP
	    targetPort: 8081
	# save and quit your favorit editor
	```
* Adjust the `nodePorts` of skupper-router service :
	```
	oc --context rhsi1 -n skupper edit service skupper-router
	...
	spec:
	  ...
	  ports:
	  - name: inter-router
	    nodePort: 32603
	    port: 55671
	    protocol: TCP
	    targetPort: 55671
	  - name: edge
	    nodePort: 32604
	    port: 45671
	    protocol: TCP
	    targetPort: 45671
	# save and quit your favorit editor
	```
* **Repeat** that on the second cluster by changing the `--context` accordingly

* Verify the deployment
    ```
    oc --context rhsi1 -n skupper get pods
    NAME                                          READY   STATUS    RESTARTS     AGE
    skupper-prometheus-64b67b568-2dhnt            2/2     Running   0            9h
    skupper-router-f6bf8cd6b-w89td                3/3     Running   0            9h
    skupper-service-controller-76fb5cf855-xlfzk   3/3     Running   1 (9h ago)   9h
    ```
## Create RHSI tokens
Links require to create tokens. For the purpose of the Lab, those tokens are set to a long lifetime and multiple re-uses. 
**NOTE** long lived tokens and multiple re-uses are settings not recommended for production tokens.

* Create a token for each Site
    ```
    skupper -c rhsi1 -n skupper token create --name rhsi1 --expiry 86400m0s --uses 1000 step2/rhsi1.token
    skupper -c rhsi2 -n skupper token create --name rhsi2 --expiry 86400m0s --uses 1000 step2/rhsi2.token
    ```
## Create links to the Sites
**NOTE** RHSI Site links are bi-directional. It is not necessary to create links vice-versa.

* Apply the necessary ServiceMesh configuration to grant access to the RHSI services
	```
	oc --context rhsi1 -n skupper apply -f step4/rhsi1/se-rhsi2.yml
	oc --context rhsi2 -n skupper apply -f step4/rhsi2/se-rhsi1.yml
	```

* Create a link from Site1 to Site2
	```
	skupper -c rhsi1 -n skupper link create --name rhsi2 step2/rhsi2.token
	```
* Alternatively, create a link from Site2 to Site1
	```
	skupper -c rhsi2 -n skupper link create --name rhsi1 step2/rhsi1.token
	```
* Verify the link state
	```
	skupper -c rhsi2 -n skupper link status

	Links created from this site:
		 Link rhsi1 is connected
	```

For the Lab show-case purpose I have added an additional Link to consolidate Cluster spawning Trace collection. Alternatively you can expose the zipkin protocol port through default Openshift routes instead. 

**NOTE** Setup of a Distributed Tracing system is not included in the Lab

## Backup of RHSI sites
RHSI provides an easy way to backup and restore Sites by dumping the `configmaps` and `secrets` containing configuration and certificates. 
**NOTE** to be able to restore those configurations the `ownerReferences` need to be removed

The same process can be utilized if you for example want to enable the UI and did not in the initial deployment.

* Create backup of the current deployment Site1
    **NOTE** the command below expect the tool [yq](https://kislyuk.github.io/yq/#installation) to be installed in the system
	```
	cd step3
	mkdir backup/{rhsi1,rhsi2}/{configmap,secret} -p
	for e in $(oc --context rhsi1 -n skupper get cm,secret -o name  | grep -v istio | grep -v kube) ; do oc --context rhsi1 -n skupper get ${e} -o yaml | \
		yq -r 'del(.metadata.ownerReferences)' > backup/rhsi1/${e}.yml ; done
	for e in $(oc --context rhsi2 -n skupper get cm,secret -o name  | grep -v istio | grep -v kube) ; do oc --context rhsi2 -n skupper get ${e} -o yaml | \
		yq -r 'del(.metadata.ownerReferences)' > backup/rhsi2/${e}.yml ; done
	```
* Remove the `metadata.ownerReference` from the CR's if `yq` is not installed
* Delete existing Sites 
	```
	oc --context rhsi1 -n skupper delete cm skupper-site
	oc --context rhsi2 -n skupper delete cm skupper-site
	```
* Recreate the Sites 
	```
	oc --context rhsi1 -n skupper apply -f backup/rhsi1/configmap -f backup/rhsi1/secret
	oc --context rhsi2 -n skupper apply -f backup/rhsi2/configmap -f backup/rhsi2/secret
	```
* Verify the deployment
	``` 
	oc --context rhsi1 -n skupper get pods
	NAME                                          READY   STATUS    RESTARTS     AGE
	skupper-prometheus-64b67b568-2dhnt            2/2     Running   0            1h
	skupper-router-f6bf8cd6b-w89td                3/3     Running   0            1h
	skupper-service-controller-76fb5cf855-xlfzk   3/3     Running   1 (1h ago)   1h
	```
* Verify the link state
	```
	skupper -c rhsi2 -n skupper link status

	Links created from this site:
		 Link rhsi1 is connected
	```
## Deploy an example Application 
The application used to show-case consolidate Cluster spawning Trace collecting as a self-written mockbin tool which is python. For consolidated tracing it's important to be able to utilized forwarded `X-B3` headers which none of the tools available provided. Still feel free to deploy any application that fits your needs.

Since we are running this Lab in OSSM and want to ensure, no `plain-text` communication is passing or traveling through our Clusters, I have included some example.com Certificates. Those have been initialized in the [OSSM setup](README.md#Deploy-the-ServiceMesh-in-mode-cluster) section.
The deployment will be done in the namespace `mockbin` for each Cluster and automatically added to the ServiceMesh due to the namespace Label.

**NOTE** if you do not have a Distributed Tracing system remove the following Environment settings from the deployment
```
vi mockbin/base/deployment.yml
        env:
        - name: JAEGER_ALL_IN_ONE_INMEMORY_COLLECTOR_PORT_14268_TCP_ADDR # < remove
          value: jaeger-collector.skupper.svc.cluster.local              # < remove
        - name: JAEGER_ALL_IN_ONE_INMEMORY_COLLECTOR_PORT_14268_TCP_PORT # < remove 
          value: "14268"                                                 # < remove
```
* Deploy the mockbin application
	```
	oc --context rhsi1 apply -k mockbin/overlays/rhsi1/
	oc --context rhsi2 apply -k mockbin/overlays/rhsi2/
	```
* Verify the deployment
	```
	oc --context rhsi1 -n mockbin get pods
	NAME                       READY   STATUS    RESTARTS   AGE
	mockbin-67f898b8c9-d8gnd   2/2     Running   0          10h
	```
* Verify the ServiceMesh functionality for Cluster1
	```
	curl --cacert traefik/ca.crt https://rhsi1.example.com
	{"base_url":"http://rhsi1.example.com/",
	..,
	{"Accept":"*/*","Host":"rhsi1.example.com",
	"User-Agent":"curl/7.76.1",
	"X-B3-Parentspanid":"1204820a39fa055e",
	"X-B3-Sampled":"0",
	"X-B3-Spanid":"1d93981b0b1528ef",
	"X-B3-Traceid":"d8ec07401ba23a981204820a39fa055e",
	"X-Envoy-Attempt-Count":"1",
	...
	```
![trace](distributed-tracing-003.png)

**NOTE** the `ca.crt` file is the same for both deployments 

*  Verify the ServiceMesh functionality for Cluster2
	```
	curl --cacert traefik/ca.crt https://rhsi2.example.com
	{"base_url":"http://rhsi2.example.com/",
	..,
	{"Accept":"*/*","Host":"rhsi2.example.com",
	"User-Agent":"curl/7.76.1",
	"X-B3-Parentspanid":"f104bc3e7882c7e9",
	"X-B3-Sampled":"0",
	"X-B3-Spanid":"91dbf16989ee3b45",
	"X-B3-Traceid":"102a4401237f2745f104bc3e7882c7e9",
	"X-Envoy-Attempt-Count":"1",
	...
	```
![trace](distributed-tracing-004.png)

## Expose each Application through RHSI 
Even though we can utilize the `annotations` to expose Services, the Lab uses one RHSI Site to access multiple Services within one Cluster and we expose those manually.

* Expose service `mockbin` in Cluster1
	```
	skupper -c rhsi1 -n skupper service create rhsi1-mockbin 8080
	skupper -c rhsi1 -n skupper service bind rhsi1-mockbin service mockbin.mockbin.svc.cluster.local --target-port=8080
	```
**NOTE** since we have both namespace in ServiceMesh we can address the mockbin service directly
In a scenario where the RHSI namespace cannot join the ServiceMesh you can still configured it pointing to the ingress gateway of the ServiceMesh instead of the application.

* Alternatively, expose service `mockbin` through ServiceMesh ingress gateway 
	```
	skupper -c rhsi1 -n skupper service create rhsi1-mockbin 8080
	skupper -c rhsi1 -n skupper service bind rhsi1-mockbin service istio-ingressgateway.istio-system.svc.cluster.local --target-port=443
	```
**NOTE** depending on the `--target-port` you might introduce a `plain-text` path from RHSI to the ingress when choosing port `80`. Using `--target-port=443` requires the traffic passing through RHSI to be `https`.

* Expose service `mockbin` in Cluster2
	```
	skupper -c rhsi2 -n skupper service create rhsi2-mockbin 8080
	skupper -c rhsi2 -n skupper service bind rhsi2-mockbin service mockbin.mockbin.svc.cluster.local --target-port=8080
	```
* Apply the necessary changes to ServiceMesh reflecting the `hostname` in the RHSI service
	```
	oc --context rhsi1 apply -f mockbin/step2/rhsi1-virtualservice.yml
	oc --context rhsi2 apply -f mockbin/step2/rhsi2-virtualservice.yml
	```
* Apply the ServiceMesh ServiceEntry to grant access to the RHSI services
	```
	oc --context rhsi1 apply -f mockbin/step2/se-rhsi2.yml
	oc --context rhsi1 apply -f mockbin/step2/se-rhsi1.yml
	```
* Verify that Application in Cluster1 can access Application in Cluster2 through RHSI
	```
	curl --cacert traefik/ca.crt https://rhsi1.example.com/tracefwd -H 'XForward: http://rhsi2-mockbin.skupper.svc.cluster.local:8080'
	{"status":"OK","traceid":"f84961e9d798060f8ac321e770dd9127"}

	oc --context rhsi2 -n mockbin logs --tail 2 deploy/mockbin
	{"written_at": "2023-11-14T18:17:49.848Z", "written_ts": 1699985869848300000, "type": "request", "correlation_id": "2125b88b-a132-bbce-889f-87de833f7e47", "remote_user": "-", "request": "/", "referer": "-", "x_forwarded_for": "-", "protocol": "HTTP/1.1", "method": "GET", "remote_ip": "127.0.0.6", "request_size_b": -1, "remote_host": "127.0.0.6", "remote_port": "43365", "request_received_at": "2023-11-14T18:17:49.843Z", "response_time_ms": 4, "response_status": 200, "response_size_b": 3849, "response_content_type": "application/json", "response_sent_at": "2023-11-14T18:17:49.848Z"}
	{"written_at": "2023-11-14T18:17:49.849Z", "written_ts": 1699985869849441000, "msg": "127.0.0.6 - - [14/Nov/2023:18:17:49 +0000] \"GET / HTTP/1.1\" 200 3849 \"-\" \"mockbin-v1.0.0\"", "type": "log", "logger": "gunicorn.access", "thread": "MainThread", "level": "INFO", "module": "glogging", "line_no": 363, "correlation_id": "1e75e94a-831a-11ee-9459-6e64fcb2ea5a"}
	```

![trace](distributed-tracing-005.png)

Step 3 of the mockbin directory show-cases some distributed tracing outputs utilizing the `/tracefwd` endpoint of the mockbin application as well as using another deployment in the `production` namespace. Feel free to experiment with the deployments accordingly.

## more use-cases
### scraping metrics from the application and the ServiceMesh
to be able to scrape merged metrics in Prometheus format we need to expose the Envoy stats port through RHSI

* Expose the Envoy stats port
```
skupper -c rhsi1 -n skupper service create rhsi1-mockbin-metrics 15020
skupper -c rhsi1 -n skupper service bind rhsi1-mockbin-metrics service mockbin.mockbin --target-port 15020

skupper -c rhsi2 -n skupper service create rhsi2-mockbin-metrics 15020
skupper -c rhsi2 -n skupper service bind rhsi2-mockbin-metrics service mockbin.mockbin --target-port 15020
```
* Create the necessary ServiceEntries in ServiceMesh
	```
	oc --context rhsi1 -n mockbin apply -f mockbin/step5/se-rhsi2.yml
	oc --context rhsi2 -n mockbin apply -f mockbin/step5/se-rhsi1.yml
	```
* Update the Service resource to the Envoy stats port
	```
	oc --context rhsi1 -n mockbin apply -f rhsi1-svc-mockbin.yml
	oc --context rhsi2 -n mockbin apply -f rhsi2-svc-mockbin.yml
	```
* Create a DestinationRule to exclude the stats port from the ServiceMesh
	```
	oc --context rhsi1 -n mockbin apply -f rhsi1-dr-mockbin.yml
	oc --context rhsi2 -n mockbin apply -f rhsi2-dr-mockbin.yml
	```
**NOTE** even though metrics are scraped `plain-text` with that setting, there's a conflict of the openshift-monitoring namespace not being in the ServiceMesh which makes that setting mandatory.

* Verify that merged metrics are scrape able 
	```
	oc --context rhsi1 -n mockbin exec -ti deploy/mockbin -- curl http://rhsi2-mockbin-metrics.skupper.svc.cluster.local:15020/stats/prometheus

	# HELP istio_agent_cert_expiry_seconds The time remaining, in seconds, before the certificate chain will expire. A negative value indicates the cert is expired.
	# TYPE istio_agent_cert_expiry_seconds gauge
	istio_agent_cert_expiry_seconds{resource_name="default"} 44150.566006366
	...
	python_info{implementation="CPython",major="3",minor="9",patchlevel="18",version="3.9.18"} 1.0
    ...
	response_created{code="403",endpoint="/tracefwd"} 1.699953991900492e+09
	response_created{code="503",endpoint="/tracefwd"} 1.6999592038140461e+09
	response_created{code="500",endpoint="/tracefwd"} 1.6999609596167657e+09
	```
	the response should contain `istio` specific metrics as well as `application` specific ones

### SSO protection of the application through ServiceMesh

**NOTE** without a  proper SSO service you will not succeed  executing these commands in the Lab.

* Update `RequestAuthentication` and `AuthorizationPolicy` resources in the ServiceMesh for the application
	```
	vi skupper/step5/sso/ossm-protected-resource.yml
	...
	    issuer: https://sso.example.com/realms/Example
	    jwksUri: https://sso.example.com/realms/Example/protocol/openid-connect/certs
	...
	    when:
	    - key: request.auth.claims[iss]
	      values:
	      - https://sso.example.com/realms/Example
	    - key: request.auth.claims[groups]
	      values:
	      - rhsi-group
	    - key: request.auth.claims[email]
	      values:
	      - milang@redhat.com
	```
    Configure the values according to your SSO setup. If your JWT does not carry group and email drop those conditions from the `AuthorizationPolicy`

* Create the `RequestAuthentication` and `AuthorizationPolicy` resources
	```
	oc --context rhsi1 -n mockbin apply -f skupper/step5/sso/ossm-protected-resource.yml \
	 -f skupper/step5/sso/se-sso.yml
	oc --context rhsi2 -n mockbin apply -f skupper/step5/sso/ossm-protected-resource.yml \
	 -f skupper/step5/sso/se-sso.yml
	```
* Verify that the `/api/v2/sso` endpoint is protected
	```
	oc --context rhsi1 -n mockbin exec -ti deploy/mockbin -- curl http://rhsi2-mockbin.skupper.svc.cluster.local:8080/api/v2/sso
	{"error": "missing_authorization", "error_description": "Missing \"Authorization\" in headers."}
	```

![trace](distributed-tracing-006.png)

* Verify that providing a Bearer Token grants access to the endpoint
	```
	oc --context rhsi1 -n mockbin exec -ti deploy/mockbin -- curl -k http://rhsi2-mockbin.skupper.svc.cluster.local:8080/api/v2/sso -H "Authorization: Bearer ..." 
	{"data":{"base_url":"http://rhsi2-mockbin.skupper.svc.cluster.local:8080/api/v2/sso",
	..
	{"Accept":"*/*","Authorization":"Bearer ..",
	"Host":"rhsi2-mockbin.skupper.svc.cluster.local:8080",
	"User-Agent":"curl/7.76.1",
	"X-B3-Parentspanid":"e1d18ef8fc439b19",
	"X-B3-Sampled":"1",
	"X-B3-Spanid":"9779160ecd69b122",
	"X-B3-Traceid":"2090904aad5fc52de1d18ef8fc439b19",
	..,
	{"status":"OK","traceid":"2090904aad5fc52de1d18ef8fc439b19"}
	```
![trace](distributed-tracing-002.png)

* Authorization can even passed through multiple Clusters using RHSI
	```
	curl --cacert traefik/ca.crt -H "Authorization: Bearer $(oidc)" \
	   https://rhsi1.example.com/tracefwd \
	   -H "XForward: http://rhsi2-mockbin.skupper.svc.cluster.local:8080/api/v2/sso"
	   
	{"status":"OK","traceid":"d1d1ff2f9e213008705b24da97fae581"}
	```
![trace](distributed-tracing-001.png)

* Trace verification through API
	```
	trace-status d1d1ff2f9e213008705b24da97fae581
	
	STATUS_CODE_OK forward
	STATUS_CODE_OK database
	STATUS_CODE_OK users
	STATUS_CODE_OK results
	 mockbin.mockbin.svc.cluster.local:8080/*
	STATUS_CODE_OK oidc-auth
	STATUS_CODE_OK api
	```


![trace](distributed-tracing-005.png)

* having some fun with loops
![trace](distributed-tracing-007.png)

## kind Cluster setup
kind k8s is a small Kubernetes cluster that requires the kind cli binary and for the Lab purpose utilizes a yaml file for the Cluster definition.

* [Download](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries) the kind cli binary
* Configure the Cluster Listen addresses according to your Network configuration
	```
	vi rhsi1.yml
	...
	networking:
	  ...
	  apiServerAddress: 127.0.0.1 # < change to your Host IP address
	  ...
	nodes:
	...
	  - containerPort: 30080
	    hostPort: 80
	    listenAddress: 127.0.0.1 # < change to your Host IP address
	    # repeat the update of the IP address to all occurances
	  ...
	```
**NOTE** running both Cluster on one System requires to have multiple IP Addresses configured on the System
* Provide a `pull-secret` with privileges to fetch Red Hat Catalog sources and images
	```
	cat /root/.docker/config.json
	{"auths":{"cloud.openshift.com":{"username":"openshift-release..
	```
    * If necessary, the `pull-secret` can be extracted from an existing OCP Cluster or through console.redhat.com
* Create the kind clusters
	```
	kind create cluster --config rhsi1.yml
	kind create cluster --config rhsi2.yml
	```
* Update the file `~/.kube/config` on the `context` names to shorten the references
	```
	vi ~/.kube/config
	...
	contexts:
	- context:
	    cluster: kind-rhsi1
	    user: kind-rhsi1
	  name: rhsi1   # < remove the kind-
	- context:
	    cluster: kind-rhsi2
	    user: kind-rhsi2
	  name: rhsi2   # < remove the kind-
	...
	```
* Check that both Cluster are up and running
	```
	oc --context rhsi1 get nodes
	oc --context rhsi1 get pods -A
	oc --context rhsi2 get nodes
	oc --context rhsi1 get pods -A
	```
###  Deploy the metallb 
* Deploy the metallb resource
	```
	oc --context rhsi1 apply -f metallb/metallb-native.yaml
	oc --context rhsi2 apply -f metallb/metallb-native.yaml
	```
* Update the IPAddressPool for metallb to match the configured kind Cluster1 listenAddress
	```
	vi metallb/rhsi1-IPAddressPool.yml
	apiVersion: metallb.io/v1beta1
	...
	spec:
	  addresses:
	  - 127.0.0.1-127.0.0.1  # < replace with Cluster listenAddress 
	                         # example 192.168.0.1-192.168.0.1
	...
	```
* Update the IPAddressPool for metallb to match the configured kind Cluster2 listenAddress
	```
	vi metallb/rhsi2-IPAddressPool.yml
	apiVersion: metallb.io/v1beta1
	...
	spec:
	  addresses:
	  - 127.0.0.1-127.0.0.1  # < replace with Cluster listenAddress 
	                         # example 192.168.0.2-192.168.0.2
	...
	```
* Verify that the metallb-system is up and running prior progressing
	```
	oc --context rhsi1 -n metallb-system get pods -l app=metallb
	NAME                          READY   STATUS    RESTARTS   AGE
	controller-789c75c689-94zvl   1/1     Running   0          26h
	speaker-9brfq                 1/1     Running   0          26h
	speaker-v9hql                 1/1     Running   0          26h
	```
* Deploy the IPAddressPool CR
	```
	oc --context rhsi1 apply -f metallb/rhsi1-IPAddressPool.yml
	oc --context rhsi2 apply -f metallb/rhsi2-IPAddressPool.yml
	```
**NOTE** if receiving an Error your metallb-system hasn't finished the deployment and you missed the above step
### Deploy traefik as ingress controller
* Deploy the traefik resource
	```
	oc --context rhsi1 apply -k traefik
	oc --context rhsi2 apply -k traefik
	```
* Verify that the traefik deployment is up and running
	```
	oc --context rhsi1 -n kube-system get pod -l app.kubernetes.io/name=traefik
	NAME                       READY   STATUS    RESTARTS   AGE
	traefik-5498f765bc-c9vh9   1/1     Running   0          26h
	```
### Deploy the ServiceMesh in mode cluster
I did not choose to utilize the Operator Lifecycle Manager for the OSSM instance as we need some adjustments which conflict with the Operator deployed CRD.
Since we are using self-signed Certificates, we need to ensure SDS verification of Certificates is skipped. This change has already been added to the CRD by an additional Environment setting `VERIFY_SDS_CERTIFICATE=false` 

* Deploy the ServiceMesh 
	```
	oc --context rhsi1 apply -k istio/overlays/rhsi1
	oc --context rhsi2 apply -k istio/overlays/rhsi2
	```
* Verify the deployment
	```
	oc --context rhsi1 -n istio-system get pods
	NAME                                    READY   STATUS    RESTARTS   AGE
	istio-ingressgateway-5bb79c76b9-4c9lz   1/1     Running   0          26h
	istiod-5bd8cf6589-cmxwk                 1/1     Running   0          26h
	```
* Update the istio configuration to restrict the ServiceMesh as well to set some necessary parameters
	```
	oc --context rhsi1 -n istio-system apply -f istio/overlays/rhsi1/istio-cm.yml
	oc --context rhsi2 -n istio-system apply -f istio/overlays/rhsi2/istio-cm.yml
	```
* Restart the ServiceMesh to ensure all changes are picked up prior deploying our applications
	```
	oc --context rhsi1 -n istio-system rollout restart deploy
	oc --context rhsi2 -n istio-system rollout restart deploy
	```	
**NOTE** the tracing collector for the ServiceMesh is set to a RHSI endpoint. If you do not have a Distributed Tracing System this endpoint will not be utilized at all.

### Deploy the Operator Lifecycle Manager
The Operator Lifecycle Manager is used to show-case RHSI operator based deployment. For easy to move configurations between Openshift and kind k8s Clusters the CRD has been adjusted to utilize the same namespaces as in Openshift.
Furthermore an additional adjustment has been made to provide the Red Hat Catalogs accordingly.

* Deploy the Operator Lifecycle Manager
	```
	oc --context rhsi1 apply -k olm/
	oc --context rhsi2 apply -k olm/
	```
* Verify the Operator Lifecycle Manager has been deployed
	```
	oc --context rhsi1 -n openshift-operator-lifecycle-manager get pods
	NAME                               READY   STATUS    RESTARTS   AGE
	catalog-operator-59f767675-jxp5t   1/1     Running   0          26h
	olm-operator-7b9c8fd79-9xrjm       1/1     Running   0          26h
	packageserver-76bb94564b-hqk9k     1/1     Running   0          26h
	packageserver-76bb94564b-twlbn     1/1     Running   0          26h
	```
**NOTE** Do not continue until the OLM pods are up and running.

[continue](README.md#Cluster-setup) with the setup.

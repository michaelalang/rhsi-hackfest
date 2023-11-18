# Distributed Tracing setup

**NOTE** this deployment is not for Production use 
Furthermore, only one Cluster needs to have the deployment, in the Lab case it will be Cluster `rhsi1`

## Deploy S3 Storage 

* Deploy the S3 Storage subsystem
    ```
    oc --context rhsi1 apply -k minio/overlays/rhsi1
    ```

* Verify the deployment 
    ```
    oc --context rhsi1 -n openshift-distributed-tracing get pods
    minio-7d8b7778f-94dz7   2/2     Running     0          76s
    minio-init-5bdpg        0/1     Completed   0          76s
    ```

**NOTE** the Job `minio-init` configures a bucket for our Tracing instance

* Verify that the init job has completed successfully 
    ```
    oc --context rhsi1 -n openshift-distributed-tracing logs job/minio-init
    Added `s3` successfully.
    Bucket created successfully `s3/tempo`.
    ```

* In case of errors or if necessary remove the job and re-deploy it 
    ```
    oc --context rhsi1 -n openshift-distributed-tracing delete job/minio-init

    oc --context rhsi1 -n openshift-distributed-tracing apply -f minio/base/init-job.yml
    ```

## Deploy the Prometheus service

* Deploy the Prometheus service 
    ```
    oc --context rhsi1 apply -k monitoring/prometheus/overlays/rhsi1
    ```

* Verify the Prometheus service and target definition
    ```
    oc --context rhsi1 -n openshift-distributed-tracing exec -ti deploy/prometheus -- \
       wget -qO- http://localhost:9090/api/v1/targets | jq -r '.data.activeTargets|length'
    5
    ```

**NOTE** since we do not have ServiceMonitor and PodMonitor in kind k8s Clusters the configuration has been pre-populated with the values of the Lab

## Deploy the Tracing service

* Deploy Tempo as tracing service
    ```
    oc --context rhsi1 apply -k tempo/overlays/rhsi1/
    ```

* Verify the tracing service is working
    **NOTE** pod watch doesn't indicate that the tempo instance is `ready` that's why it might take a few more second until it returns `ready`

    ```
    oc --context rhsi1 -n openshift-distributed-tracing exec -ti deploy/tempo -- \
       wget -qO- http://localhost:3200/ready
    ready
    ```

## Expose the tracing service for the RHSI-hackfest Lab

* Create the namespaces:
        ```
        oc --context rhsi1 apply -f skupper/base/namespace.yml
        oc --context rhsi2 apply -f skupper/base/namespace.yml
        ```

* create the skupper services for jaeger-collector, jaeger-collector/zipkin
    ```
        skupper -c rhsi1 -n skupper service create jaeger-collector 4317 4318 9411 14267 14268 14250
        skupper -c rhsi1 -n skupper service bind jaeger-collector service tempo.openshift-distributed-tracing.svc.cluster.local
    ```

    **NOTE** following declarative config is Work in Progress
    ```
    oc --context rhsi1 apply -f skupper/step2/rhsi1-skupper-services.yml
    oc --context rhsi2 apply -f skupper/step2/rhsi2-skupper-services.yml
    ```

* verify that the services are configured as expected
    ```
    oc --context rhsi1 -n istio-system exec -ti deploy/istio-ingressgateway -- curl -X POST http://jaeger-collector.skupper.svc:9411/api/v1/traces -d '{}'

    json: cannot unmarshal object into Go value of type []*model.SpanModel
    ```
    ```
    oc --context rhsi2 -n istio-system exec -ti deploy/istio-ingressgateway -- curl -X POST http://jaeger-collector.skupper.svc:9411/api/v1/traces -d '{}'

    json: cannot unmarshal object into Go value of type []*model.SpanModel
    ```

## Deploy Grafana as metrics Dashboard

* Deploy Grafana as service

    ```
    oc --context rhsi1 apply -k monitoring/grafana/overlays/rhsi1/
    ```

* verify that the services starts as expected

    ```
    oc --context rhsi1 -n openshift-distributed-tracing logs deploy/grafana | grep "HTTP Server Listen"
    logger=http.server t=2023-11-17T19:48:18.785977432Z level=info msg="HTTP Server Listen" address=[::]:3000 protocol=http subUrl= socket=
    ```

* login to the Grafana dashboard 

    use the credentials `admin`:`admin` to login to the dashboard

    * configure Data sources
        * add `Prometheus` as your first and primary data source
            * Prometheus server URL `http://prometheus:9090` 
            * hit `Safe & test` 
        * add `Tempo` as your second data source for tracing
            * URL `http://tempo:80`
            * Expand `Additional settings`
            * Select `Prometheus` as Data source for `Service graph`
            * switch `Enable node graph` to on 
            * hit `Safe & test` 
    * import the [included](dashboard.json) Dashboard

[continue](README.md#Backup-of-RHSI-sites) with the next steps


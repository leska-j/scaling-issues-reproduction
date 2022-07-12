# Minimal reproduction of scaling issues

## Observed issues with `keda & http-add-on` scaling

inst- big delay between the time when service is scaled from 0 and time when request is received by service
  - after service is scaled and running, http-add-on passes trough the request to the service
  - based on service logs it takes several seconds until service receives http request
- when multiple services are scaled from zero in quick succession they are not scaled back to 0 after given time even though there are not new requests

We observed these issues in eks cluster in aws and we were able to reproduce it locally running cluster.
This repository exists to simplify investigation, it can be used to deploy minimal reproduction scenario
into local `kind` cluster. 

## Deployment

1) create kind cluster

   ```bash
   export CL_NAME=scalingtest
   export KEDA_NAMESPACE=keda  # name is used in helm files, if changed files may need adjustment
   export TMP_DIR=$(mktemp -d)
   kind create cluster --name ${CL_NAME}

2) prepare latest version of http-add-on(v0.3.0 has concurrency issue)
   ```bash
   # switch to dir
   cd ${TMP_DIR}
   # fetch repo
   git clone https://github.com/kedacore/charts.git
   git clone https://github.com/kedacore/http-add-on.git && cd http-add-on
   # build
   export KEDAHTTP_IMAGE_TAG=0.3.1  # non released version
   export KEDAHTTP_OPERATOR_IMAGE=http-add-on-operator:${KEDAHTTP_IMAGE_TAG}
   export KEDAHTTP_INTERCEPTOR_IMAGE=http-add-on-interceptor:${KEDAHTTP_IMAGE_TAG}
   export KEDAHTTP_SCALER_IMAGE=http-add-on-scaler:${KEDAHTTP_IMAGE_TAG}
   mage dockerBuild
   kind load docker-image ${KEDAHTTP_OPERATOR_IMAGE} --name ${CL_NAME}
   kind load docker-image ${KEDAHTTP_INTERCEPTOR_IMAGE} --name ${CL_NAME}
   kind load docker-image ${KEDAHTTP_SCALER_IMAGE} --name ${CL_NAME}
   # return to the repository (feel free to use any command this is just example)
   cd ~2
   ```
3) install keda
   ```bash
   # make keda available in helm
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo update

   # create namespace for keda and app
   kubectl --context kind-${CL_NAME} create namespace ${KEDA_NAMESPACE}
   kubectl --context kind-${CL_NAME} create namespace app

   # deploy keda to its namespace
   helm --kube-context kind-${CL_NAME} --namespace ${KEDA_NAMESPACE} install keda kedacore/keda 

   # install keda http addon
   helm install --kube-context kind-${CL_NAME} --namespace ${KEDA_NAMESPACE} \
        http-add-on -f keda-http-addon-config.yaml ${TMP_DIR}/charts/http-add-on
   ```
4) deploy scalable dummy apps
   ```bash
   N=10  # number of services to deploy
   for i in `seq 1 $N`
   do
       helm install --kube-context kind-${CL_NAME} --namespace app dummy$i --set deploymentNumber=$i dummy
   done
   ```
5) create test pod
   ```bash
   kubectl --context kind-${CL_NAME} --namespace app run "testpod" \
           --image=alpine/curl --restart=Never -- sh -c "sleep infinity"
   ```

## Cleanup
1) destroy kind cluster

   ```bash
   kind delete cluster --name ${CL_NAME}
   ```

## Test
these steps are intended to be runned between deployment and cleanup (kind cluster is persistent so deployment should be needed only once)

### Sanity check
1) run single service to see that it is scaled up
    ```bash
    kubectl --context=kind-${CL_NAME} exec -itn app testpod -c testpod -- \
        time curl keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local:8080 -H "Host: dummy-2"
    ```
2) real time result of time command should be close to pod startup time
3) wait till its scaled down, this should take 300s(in reality it usually takes ~7-9minutes)

### Sequential test
1) run all services sequentially with enough time between
   ```bash
   N=10  # should be less or equal as N in step 4 deployment
   DELAY=60
   for i in `seq 1 $N`
   do
        kubectl --context=kind-${CL_NAME} exec -itn app testpod -c testpod -- sh -c \
            "time curl -s keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local:8080 -H \"Host: dummy-$i\" > /dev/null"
        sleep $DELAY
   done
   ```
2) expected results:
    - real time should be close to sanity check values
    - scale down time should be close to sanity check values [failing]


### Parallel test
1) run all services sequentially with enough time between
   ```bash
   kubectl --context=kind-${CL_NAME} exec -itn app testpod -c testpod -- sh
   # run inside of the pod
   N=10  # should be less or equal as N in step 4 deployment
   for i in `seq 1 $N`
   do
       time curl -s keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local:8080 -H "Host: dummy-$i" > /dev/null &
   done
   ```

2) expected results:
    - real time should be close to sanity check values [failing]
    - scale down time should be close to sanity check values [failing]
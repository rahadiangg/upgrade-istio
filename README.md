# Upgrade Istio

This documentation based on Acasia production EKS, but actually you can use in general infrastructur of Kubernetes.

## Preparation

Before you upgrade the Istio, you should check current version with `istioctl version`, the result is like this:

```
client version: 1.20.0
control plane version: 1.16.1
data plane version: 1.16.1 (2 proxies)
```

The client is means your `istioctl` version and control plane or data plane is version that Istio running in your cluster. There a several method for upgrading Istio like Canary or In-plane Replacement. We user Canary upgrade for safely upgrade our Istio without (or minimum) Downtime. Istio's documentation on how to upgrade versions is lacking in detail, so what is in this documentation is based on the documentation and improvisations according to current conditions.

For canary upgrade, the recomendation max version that you can upgrade is 2x of minor version. The example is 1.16.x to 1.18.x, check this [link](https://istio.io/latest/news/releases/) to get list of Istio release version. Before upgrade your version, you also should check the Istio API version of [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) like Gateway or VirtualService that Istio support in that version. The example is like VirtualService in version [1.28.x only support v1alpha3 & v1beta1](https://istio.io/v1.20/docs/reference/config/networking/virtual-service/) but in version [1.22.x that version already stable using v1](https://istio.io/v1.22/docs/reference/config/networking/virtual-service/).

### Check Istio compability

You should check the compability matrix of Istio version with your kubernetes cluster in this [link](https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases).

### Download specified Istioctl

You can download specified version of `istioctl` with this command:

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.7 sh -
```

After that you can try use `istioctl` binary file

```
cd istio-1.18.7/bin
./istioctl version
```

> <b>Note:</b> When download `istioctl` in specified version that also work for install/upgrade to that version. 

### Check running sidecar proxy & ingress version

Before, after, or anytime when process upgrading Istio, you should check running sidecar & ingress version using this command `./istioctl x ps` or `./istioctl ps`

```
NAME                                                   CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                     VERSION
istio-ingressgateway-74445859fd-bp2jr.istio-system     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-9c99ccc45-htmlk     1.16.1
nginx-test-69cc5bd84b-msxrb.default                    Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-9c99ccc45-htmlk     1.16.1
```

In above example you can see exiting sidecar & istio ingress using version 1.16.1 and connect to IstioD (control plane) `istiod-9c99ccc45-htmlk` in namespace istio-system. Later this will help us to identify version when upgrading version.

### Check existing Istio revision & ingress service type

In general tutorial when installing Istio people using command like `istioctl install` or `istioctl install --set profile=default`. That will use Istio default profile and will install CRD, IstioD, and ingress-gateway. The ingress-gateway will use LoadBalancer as service type, however in Acasia this use NodePort because a reason that will explain later.

```
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
istio-egressgateway    ClusterIP   10.100.1.154   <none>        80/TCP,443/TCP                               2y1d
istio-ingressgateway   NodePort    10.100.0.221   <none>        15021:30249/TCP,80:30893/TCP,443:32471/TCP   2y1d
```

> <b>Note:</b> Check Istio profile with the component that will installed in this [link](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)

In one kubernetes cluster can have multiple installed Istio version, this method allow us to Canary upgrade. Each installed version called as <b>revision</b>. To check list of revision in current cluster, you can use command `./istioctl x revision list`

```
REVISION TAG     ISTIO-OPERATOR-CR            PROFILE REQD-COMPONENTS
default  default istio-system/installed-state default Istio core
                                                      Istiod
                                                      Ingress gateways:istio-ingressgateway
```

## Canary Upgrade Istio

### Install new revision

This part we will install Istio as another revision to get benefit of Canary upgrade. In this example we will try upgrade from 1.16.x to 1.18.x. To installed, please follow this command format:

```
cd istio-1.18.7/bin
./istioctl install --set profile=minimal --set revision=1-18-7
```

> <b>Note:</b> Why using minimal as Istio profile ?. This to make sure we not replace exising ingress-gateway with the new version. 

After another revision already installed, we can check the revision and the result will like this:

```
REVISION TAG      ISTIO-OPERATOR-CR                   PROFILE REQD-COMPONENTS
default  default  istio-system/installed-state        default Istio core
                                                              Istiod
                                                              Ingress gateways:istio-ingressgateway
1-18-7   <no-tag> istio-system/installed-state-1-18-7 minimal Istio core
                                                              Istiod
```

> <b>Note:</b> the `default` revision in above example is Istio with version 1.16.1

### Upgrade sidecar proxy to new Istio revision

There a several way to doing Istio Injection to make Istio sidecar proxy run in specified namespace or pod. Usually we enable Istio injection at namespace level with this command `kubectl label namespace <namespace> istio-injection=enabled`. You can it first using `kubectl describe namespace <namespace>`.

```
Name:         default
Labels:       istio-injection=enabled
              kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active
```

To upgrade/ use another revision, we should remove istio-injection label and add new label istio.io/rev using this command `kubectl namespaces default istio-injection- istio.io/rev=1-18-7` and if we check again, the result will like this:

```
Name:         default
Labels:       istio.io/rev=1-18-7
              kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active
```

> <b>Note:</b> we should not using label `istio-injection=` with `istio.io/rev=` for same namespace!, because that labels is mutually exclusive.

After add that label, we should rollout restart all pod (deployment or statefulsets) inside that namespace to get new sidecar proxy version. After that we can [check running proxy](#check-running-sidecar-proxy--ingress-version) again and the result will like this

```
$ ./istioctl x ps
NAME                                                   CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                             VERSION
istio-ingressgateway-74445859fd-bp2jr.istio-system     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-9c99ccc45-htmlk             1.16.1
nginx-test-8cc8b456d-qgjc4.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-1-18-7-8476f6fcd6-klfzg     1.18.7
```

If you can see our application already using new sidecar proxy version, at this step we already successfully upgrade the sidecar without downtime. You can try check service communication between pod & exposed domain. But the ingress-gateway still using old version, we will learn how to upgrade this in [another section](#upgrade-istio-ingress-gateway).

### Upgrade Istio ingress gateway

In general when people using Istio Gateway, they will use this selector.

```
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  selector:
    # sometims people use this labels
    app: istio-ingressgateway
    istio: ingressgateway
```

To upragde ingress-gateway without downtime we can deploy another kubernetes Deployments of gateway with same label. You can use example file `istio-ingress-auto-inject-from-revision.yaml`. In that file you should change this part.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-ingressgateway-1-18-7 # you can name it same with the revision
  namespace: istio-system
spec:
  ...
  template:
    metadata:
      labels:
        ...
        istio.io/rev: 1-18-7 # Set to the istio revision that you want to use
    spec:
        ...
```

After that you can apply that file. At this moment we have 2 kubernetes Deployments of ingress-gateway.

```
$ kubectl get deployments -n istio-system
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
istio-ingressgateway          1/1     1            1           2d4h
istio-ingressgateway-1-18-7   1/1     1            1           27s

$ kubectl get pod
NAME                                          READY   STATUS    RESTARTS   AGE
istio-ingressgateway-1-18-7-cddb98855-qjjmp   1/1     Running   0          52s
istio-ingressgateway-74445859fd-bp2jr         1/1     Running   0          2d4h
```

Make sure new Deployment ingress-gateway using Istio proxy version & connect to new IstioD, the result should like below:

```
NAME                                                         CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                             VERSION
istio-ingressgateway-1-18-7-cddb98855-qjjmp.istio-system     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-1-18-7-8476f6fcd6-klfzg     1.18.7
istio-ingressgateway-74445859fd-bp2jr.istio-system           Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-9c99ccc45-htmlk             1.16.1
nginx-test-8cc8b456d-qgjc4.default                           Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-1-18-7-8476f6fcd6-klfzg     1.18.7
```

At this moment kubernetes service `istio-ingressgateway` will load balance to both different ingress-gateway (1.16.x and 1.18.x).

```
$ kubectl get pod -n istio-system -o wide
NAME                                          READY   STATUS    RESTARTS   AGE     IP              NODE                                              NOMINATED NODE   READINESS GATES
istio-ingressgateway-1-18-7-cddb98855-qjjmp   1/1     Running   0          8m32s   172.31.27.104   ip-172-31-30-81.ap-southeast-3.compute.internal   <none>           <none>
istio-ingressgateway-74445859fd-bp2jr         1/1     Running   0          2d4h    172.31.5.155    ip-172-31-7-10.ap-southeast-3.compute.internal    <none>           <none>

$ kubectl describe endpoints istio-ingressgateway
Name:         istio-ingressgateway
...
Subsets:
  Addresses:          172.31.27.104,172.31.5.155
  NotReadyAddresses:  <none>
  Ports:
    Name         Port   Protocol
    ----         ----   --------
    status-port  15021  TCP
    http2        8080   TCP
    https        8443   TCP
```

You can scale to zero the old Deployment and test access exposed service. you should still be able to access the exposed applications.

```
$ kubectl scale deployment istio-ingressgateway --replicas 0
```

### Remove old ingress gateway

There a several steps to safely remove old version of istio-ingress. "Remove" this step that means replace old version wih new version. At the begining when install new revision we use `minimal` as Istio profile. In this step we will install with same command but with `default` as Istio profile.

> <b>Note:</b> Remember, at Acasia ingress-gateway use NodePort as service type. This because Acasia use Kubernetes ingress combined with istio-ingress to expose the service.

This example command based on Acasia use case, in general case you can use LoadBalancer as service type as usual.

```
cd istio-1.18.7/bin
./istioctl install --set profile=default --set revision=1-18-7 --set values.gateways.istio-ingressgateway.type=NodePort
```

After that you can scale up the `istio-ingressgateway` and check again the proxy version, the version should same with new revision.

```
$ kubectl scale deployment istio-ingressgateway --replicas 1
$ ./istioctl x ps
NAME                                                         CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                             VERSION
istio-ingressgateway-1-18-7-cddb98855-qjjmp.istio-system     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-1-18-7-8476f6fcd6-klfzg     1.18.7
istio-ingressgateway-59d5cc4954-zxgsl.istio-system           Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-1-18-7-8476f6fcd6-klfzg     1.18.7
nginx-test-8cc8b456d-qgjc4.default                           Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT_SENT     istiod-1-18-7-8476f6fcd6-klfzg     1.18.7
```

at this moment can safely remove the custom deployment that we apply before

```
$ kubectl delete -f istio-ingress-auto-inject-from-revision.yaml
```

You can check that the application should still be accessible at this time. We can continue to remove old revision.

```
$ ./istioctl x revision list
REVISION TAG      ISTIO-OPERATOR-CR                   PROFILE REQD-COMPONENTS
default  default  istio-system/installed-state        default Istio core
                                                              Istiod
                                                              Ingress gateways:istio-ingressgateway
1-18-7   <no-tag> istio-system/installed-state-1-18-7 default Istio core
                                                              Istiod
                                                              Ingress gateways:istio-ingressgateway

$ ./istioctl uninstall --revision default
...
✔ Uninstall complete
```

# References

- https://istio.io/latest/docs/setup/upgrade/canary/
- https://istio.io/latest/docs/setup/additional-setup/gateway/#canary-upgrade-advanced

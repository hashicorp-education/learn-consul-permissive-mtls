
## Create environment 

- `terraform init`
- `terraform apply`

```
aws eks update-kubeconfig \
  --region $(terraform output -raw region) \
  --name $(terraform output -raw cluster_name)
```

### Deploy Consul

- `./consul-k8s install -config-file k8s-yamls/consul-helm.yaml`

Confirm Consul is up.

```
$ kubectl get pods --namespace consul
NAME                                           READY   STATUS    RESTARTS   AGE
consul-connect-injector-59b5b4fccd-mqmhv       1/1     Running   0          90s
consul-mesh-gateway-7b86b77d99-rhfgd           1/1     Running   0          90s
consul-server-0                                1/1     Running   0          90s
consul-webhook-cert-manager-57c5bb695c-qxc5t   1/1     Running   0          90s
```

Confirm that helm chart version is `1.2.0-dev`.

```
$ helm list --namespace consul
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS     CHART            APP VERSION
consul  consul          1               2023-05-14 06:12:25.898867 -0700 PDT    deployed   consul-1.2.0-dev 1.15.1     
```

### Deploy HashiCups services

```
$ for service in {products-api,postgres,intentions-api-db}; do kubectl apply -f hashicups-v1.0.2/$service.yaml; done
service/products-api created
serviceaccount/products-api created
servicedefaults.consul.hashicorp.com/products-api created
configmap/db-configmap created
deployment.apps/products-api created
service/postgres created
serviceaccount/postgres created
servicedefaults.consul.hashicorp.com/postgres created
deployment.apps/postgres created
serviceintentions.consul.hashicorp.com/postgres created
serviceintentions.consul.hashicorp.com/deny-all created
```

Verify services in k8s.

```
$ kubectl get service
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes     ClusterIP   172.20.0.1      <none>        443/TCP    12m
postgres       ClusterIP   172.20.196.70   <none>        5432/TCP   4s
products-api   ClusterIP   172.20.182.94   <none>        9090/TCP   7s
```

Verify services in Consul.

```
$ kubectl exec --namespace consul -it consul-server-0 -- consul catalog services
Defaulted container "consul" out of: consul, locality-init (init)
consul
mesh-gateway
postgres
postgres-sidecar-proxy
products-api
products-api-sidecar-proxy
```

## Enable permissive mTLS (mesh)

```
$ kubectl apply -f k8s-yamls/mesh-config-entry.yaml --namespace consul
mesh.consul.hashicorp.com/mesh created
```

## Connect services to Consul services

First, you need to dpeloy services to your Kubernetes clusters. Permissive mTLS requires TProxy (so it only works with Consul on Kuberentes for now). The following HashiCups service deployments have `consul.hashicorp.com/connect-inject` explicitly set to `false` so Consul does not register them.

(might need to do a deeper dive on tproxy and how routing works)

```
$ for service in {frontend,nginx,public-api,payments}; do kubectl apply -f hashicups-v1.0.2/$service.yaml; done
service/frontend created
serviceaccount/frontend created
servicedefaults.consul.hashicorp.com/frontend created
deployment.apps/frontend created
service/nginx created
serviceaccount/nginx created
servicedefaults.consul.hashicorp.com/nginx created
configmap/nginx-configmap created
deployment.apps/nginx created
service/public-api created
serviceaccount/public-api created
servicedefaults.consul.hashicorp.com/public-api created
deployment.apps/public-api created
service/payments created
serviceaccount/payments created
servicedefaults.consul.hashicorp.com/payments created
deployment.apps/payments created
```

Verify services in k8s.

```
$ kubectl get service
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
frontend       ClusterIP   172.20.55.80     <none>        3000/TCP   20s
kubernetes     ClusterIP   172.20.0.1       <none>        443/TCP    14m
nginx          ClusterIP   172.20.11.54     <none>        80/TCP     18s
payments       ClusterIP   172.20.172.174   <none>        1800/TCP   14s
postgres       ClusterIP   172.20.196.70    <none>        5432/TCP   117s
products-api   ClusterIP   172.20.182.94    <none>        9090/TCP   2m
public-api     ClusterIP   172.20.255.151   <none>        8080/TCP   16s
```

Verify services do not appear in Consul.

```
$ kubectl exec --namespace consul -it consul-server-0 -- consul catalog services
Defaulted container "consul" out of: consul, locality-init (init)
consul
mesh-gateway
postgres
postgres-sidecar-proxy
products-api
products-api-sidecar-proxy
```

Open HashiCups in your browser. In a new terminal, port-forward the `nginx` service to port `8080`. 

```
$ kubectl port-forward deploy/nginx 8080:80
```

Open [localhost:8080]() in your browser to view the HashiCups UI. Notice that it displays no products, since the `public-api` cannot connect to the `products-api`.

### Set permissive mTLS at service level

Enable `products-api` to allow non-mTLS traffic.

```
$ kubectl apply -f k8s-yamls/service-defaults-products-api.yaml --namespace consul
```

## Migrate services to Consul

Even though `public-api` was able to connect to `products-api` and there is a deny-all intention on the Consul datacenter, **intentions only take effect for mTLS connections**. For non-mTLS connections (permissive), intentions are effectively ignored.
As a result, use permissive mTLS with great caution and be mindful of [security best practices]().

You will migrate the remaining HashiCups services to Consul service mesh. First, in each service deployment definition, update the `consul.hashicorp.com/connect-inject` annotation from `false` to `true`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        consul.hashicorp.com/connect-inject: "true"
```

Once you have done this to all four files, apply the changes. In addition, you will apply a file that creates intentions between these services. 

```
$ for service in {frontend,nginx,public-api,payments,intentions-new-services}; do kubectl apply -f hashicups-v1.0.2/$service.yaml; done
service/frontend unchanged
serviceaccount/frontend unchanged
servicedefaults.consul.hashicorp.com/frontend unchanged
deployment.apps/frontend configured
service/nginx unchanged
serviceaccount/nginx unchanged
servicedefaults.consul.hashicorp.com/nginx unchanged
configmap/nginx-configmap unchanged
deployment.apps/nginx configured
service/public-api unchanged
serviceaccount/public-api unchanged
servicedefaults.consul.hashicorp.com/public-api unchanged
deployment.apps/public-api configured
service/payments unchanged
serviceaccount/payments unchanged
servicedefaults.consul.hashicorp.com/payments unchanged
deployment.apps/payments configured
serviceintentions.consul.hashicorp.com/public-api created
serviceintentions.consul.hashicorp.com/payments created
serviceintentions.consul.hashicorp.com/frontend created
```

Verify services appear in Consul.

```
$ kubectl exec --namespace consul -it consul-server-0 -- consul catalog services
Defaulted container "consul" out of: consul, locality-init (init)
consul
frontend
frontend-sidecar-proxy
mesh-gateway
nginx
nginx-sidecar-proxy
payments
payments-sidecar-proxy
postgres
postgres-sidecar-proxy
products-api
products-api-sidecar-proxy
public-api
public-api-sidecar-proxy
```

### Set up intentions

```
$ kubectl apply -f hashicups-v1.0.2/intentions-public-products-api.yaml --namespace consul
serviceintentions.consul.hashicorp.com/public-api created
```

### Restrict permissive mTLS at service level

Restrict `products-api` to only accept mTLS traffic.

In `k8s-yamls/service-defaults-products-api.yaml`, update `mutualTLSMode` to `"strict"`.

```yaml
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: products-api
spec:
  protocol: http
  mutualTLSMode: "permissive"
```

Then, apply the changes.

```
$ kubectl apply -f k8s-yamls/service-defaults-products-api.yaml --namespace consul
servicedefaults.consul.hashicorp.com/products-api configured
```

## Restrict permissive mTLS (mesh)

In `k8s-yamls/mesh-config-entry.yaml`, update `allowEnablingPermissiveMutualTLS` to `false`.

```yaml
apiVersion: consul.hashicorp.com/v1alpha1
kind: Mesh
metadata:
  name: mesh
spec:
  allowEnablingPermissiveMutualTLS: true
```

Then, apply the changes.

```
$ kubectl apply -f k8s-yamls/mesh-config-entry.yaml --namespace consul
mesh.consul.hashicorp.com/mesh configured
```

## Clean up environments

```
$ ./consul-k8s uninstall
```

```
$ terraform destroy
```
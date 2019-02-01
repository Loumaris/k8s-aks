# k8s

Example repository for setting up cert-manager on aks. Demo domain is `brainops.io`.

## pre requirements

You need to setup an aks (_azure kubernetes service_) with RBAC, the kubeconfig should
point to the cluster.

## enable dashboard
```
# enable dashbaord
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```

## deploy demo container

```
cd 01-brainops

kubectl apply -f 01-brainops.deployment.yaml
kubectl apply -f 02-brainops.service.yaml
```

## deploy ingres

! you need to change the hostname in `09-ingress.yml`

! the tls section in `09-ingress.yml` needs to be disabled on the first run

```
cd 02-ingress

kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-rbac.yaml
kubectl apply -f 03-configmap.yaml
kubectl apply -f 04-tcp-services-configmap.yaml
kubectl apply -f 05-udp-services-configmap.yaml
kubectl apply -f 06-default-backend.yaml
kubectl apply -f 07-ingress.deployment.yaml
kubectl apply -f 08-ingress.service.yaml
kubectl apply -f 09-ingress.yml
```

wait for the ip... need some time. Pick the ip and set it to your domain.

## cert-manager

See https://cert-manager.readthedocs.io/en/latest/getting-started/install.html

```
# create namespace
kubectl create namespace cert-manager

# disable validation (important)
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# create resources
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/00-crds.yaml

# deploy cert-manager, don't forget the flag at the end
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/cert-manager.yaml --validate=false

# check if everything is working
kubectl get pods --namespace cert-manager

# should be like this:
kubectl get pods --namespace cert-manager
NAME                                    READY     STATUS      RESTARTS   AGE
cert-manager-7b6d7f868d-465kf           1/1       Running     0          1m
cert-manager-webhook-5c745c69f7-ppwg5   1/1       Running     0          1m
cert-manager-webhook-ca-sync-g2vg4      0/1       Completed   0          1m
```

! one _failed_ or just _completed_ pod is **correct** don't worry about it!

## issuer deployen

create two issuer, one for staging (testing), one for production

```
cd 03-cert-manager

kubectl apply -f 01-issuer.yml
```

## create certs


You need to update the domain names inside the cert files. For testing,
use the staging issuer first, the api of let's encrypt staging env is much more
fault-tolerant than the production.

### staging zum testen

```
cd 03-cert-manager
kubectl apply -f 02-cert-staging.yml

# check state
kubectl describe certificate  <cert-name>

# after some minutes it should be like this
...

  Type    Reason         Age   From          Message
  ----    ------         ----  ----          -------
  Normal  Generated      37s   cert-manager  Generated new private key
  Normal  OrderCreated   37s   cert-manager  Created Order resource "kube-heimke-net-staging-3300275463"
  Normal  OrderComplete  8s    cert-manager  Order "brainops-io-staging-3300275463" completed successfully
  Normal  CertIssued     8s    cert-manager  Certificate issued successfully
```

### production

Same as staging, just another issuer.

## ingress anpassen

You can now enable TLS in ingress!

```
cd 02-ingress

cat 09-ingress.yml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  # dieser abschnitt darf erst aktiviert werden, wenn der
  cert-manager deployed ist
  tls:
  - hosts:
    - brainops.io
    secretName: brainops-io-tls
  rules:
  # hostnamen anpassen
  - host: brainops.io
    http:
      paths:
      - path: /
        backend:
          serviceName: brainops
          servicePort: 80

kubectl apply -f 09-ingress.yml
```

After some seconds, your domain should have a valid SSL cert by let's encrypt.
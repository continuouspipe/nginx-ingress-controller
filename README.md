# Deployment Instructions

## Annotate Any Existing Ingresses

Existing ingresses that don't have an "ingress.class" will be fought over by a second ingress controller, with a constant fight between controllers to own the ingress.

Add an annotation with the right annotation - "gce" if hosted in Google Cloud. The following will generate commands that you can run:
```
INGRESS="$(kubectl get ingress --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations}{"\n"}{end}' | grep -v kubernetes.io/ingress.class)"
if [ -n "$INGRESS" ]; then echo $INGRESS | cut -f1,2 | awk '{print "kubectl --namespace="$1" annotate ingress "$2" kubernetes.io/ingress.class=gce"}'; fi
```

## Get a Wildcard SSL Certificate for the Cluster

### Self Signed Certificate
To generate a self signed certificate for the domain name you wish to assign to the cluster where the nginx ingress controller is deployed:

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout star.foo.continuouspipe.net.key -out star.foo.continuouspipe.net.crt -subj "/CN=*.foo.continuouspipe.net/O=CP"`
```

### Where to Place It:

Base64 the contents of both the certificate and key, e.g.:
```
base64 star.foo.continuouspipe.net.key
base64 star.foo.continuouspipe.net.crt
```

Update the values in `ingress-controller/deployment.yml` with the base64 key and certificate values.

## Deploy the Ingress Controller

Deploy a default backend to serve if no server is found for a given hostname:

```
kubectl create ns continuouspipe-system
kubectl --namespace=continuouspipe-system create -f default-backend/service.yml
kubectl --namespace=continuouspipe-system create -f default-backend/deployment.yml
```

Deploy the ingress controller:
```
kubectl --namespace=continuouspipe-system create -f ingress-controller/nginx-load-balancer-conf.yml
kubectl --namespace=continuouspipe-system create -f ingress-controller/service.yml
kubectl --namespace=continuouspipe-system create -f ingress-controller/deployment.yml
```

## Set up a Wildcard DNS Entry

The wildcard DNS should point to the external IP exposed for the service of the ingress controller:

```
$ kubectl --namespace=continuouspipe-system get service nginx-ingress-controller -o yaml
[...]
  status:
    loadBalancer:
      ingress:
      - ip: 123.123.123.123
````

```
123.123.123.123 *.foo.continuouspipe.net
```

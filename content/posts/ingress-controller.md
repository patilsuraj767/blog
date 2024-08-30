+++
title = 'Exposing application outside kubernetes without external load balancer'
date = 2024-08-30T11:48:12+05:30
draft = false
+++

This post describe how to expose http service to outside world on port 80 without external load balancer.

For an example here will be using dummy bookinfo application by istio. This application contains a product page service which we will be exposing on the port 80 of kubernetes node.

To deploy the istio sample application, run the below command.

```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml
```

For more details on sample bookinfo application check istio doc - https://istio.io/latest/docs/examples/bookinfo/

Now we will deploy ingress-nginx-controller and will route traffic to product page service of bookinfo application.

![Scenario 1: ingress-controller](/ingress-controller.png)

Run below command to install ingress-nginx controller.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
```

For more details check official quick-start guide -
https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

Confirm that ingress-nginx-controller pod is running.

```
kubectl get pods -n ingress-nginx
```

To bind the nginx-controller to the host network we need to edit ingress-nginx-controller deployment.

```
kubectl edit deployments ingress-nginx-controller -n ingress-nginx
```

Add `hostNetwork: true` under `template.spec`

```
template:
  spec:
    hostNetwork: true
```

Once above changes are saved. ingress-nginx-controller pod will be restarted.

```
kubectl get pods -n ingress-nginx
```

Finally we need to create the Ingress custom resource to route traffic to the product service.

Use below command to create the Ingress custom resource.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-myservicea
spec:
  rules:
  - host: bookinfo.example.com       # change this to your hostname
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: productpage
            port:
              number: 9080
  ingressClassName: nginx
EOF
```

Now access the product page at bookinfo.example.com (obviously change this hostname to yours).

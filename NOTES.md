# Notes


Get Ingres information:

```
kubectl get ingresses.v1.networking.k8s.io -n smt-local
```

## Ingress Issues


To get Ingress Nginx logs on Minikube use:

```
kubectl logs -f ingress-nginx-controller-558664778f-r9hrf -n kube-system
```

where the random number string differs per launch of course.
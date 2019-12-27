1. Setup Nginx Ingress controller

```helm upgrade --install nginx-ingress stable/nginx-ingress```

2. Setup Certificate Manger

```helm upgrade --install cert-mgr stable/cert-manager```

3. Setup Certificate Cluster Issuer

```kubectl apply -f cluster-issuer-staging.yaml```

4. Test configuration - Demo!

```helm upgrade jenkins-ingress --install --namespace livedemo -f ./jenkins-values-demo.yaml stable/jenkins```

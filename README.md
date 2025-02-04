# Deploying ELK on Hetzner with Kubernetes Operator

This guide will walk you through deploying the **ELK Stack** on a **Hetzner machine** using the **official Kubernetes operator** for ELK and exposing it via **NGINX Ingress Controller**.

## Prerequisites
- A **Hetzner** machine with **Kubernetes** installed
- **Helm** installed on your system
- **kubectl** configured for your cluster

---

## Step 1: Deploy ELK Operator on Kubernetes
Follow the official **Elastic Cloud on Kubernetes (ECK)** documentation for deploying ELK:

[Elastic ECK Quickstart Guide](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)

Once the ELK operator is deployed successfully, proceed to the next step.

---

## Step 2: Install Cert-Manager for SSL
To enable **HTTPS**, install **Cert-Manager**:

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Next, create a **ClusterIssuer** for Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    email: your-email@example.com   # Replace with your email
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply this YAML file using:
```sh
kubectl apply -f cluster-issuer.yaml
```

---

## Step 3: Install the NGINX Ingress Controller
To expose the services, install **NGINX Ingress Controller** using Helm:

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx  
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.watchNamespace=""
```

---

## Step 4: Create an Ingress Resource for Kibana
Now, create an **Ingress Resource** to expose Kibana. Apply the following YAML file in the **default** namespace (or the namespace where ELK is deployed):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.domain.com
    secretName: kibana-tls
  rules:
  - host: example.domain.com  # Replace with your domain or external IP
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: quickstart-kb-http
            port:
              number: 5601
```

Apply this YAML file using:
```sh
kubectl apply -f kibana-ingress.yaml
```

---

## Step 5: Accessing Kibana
Once everything is set up, access **Kibana** using your domain (`https://example.domain.com`).

To retrieve the **Kibana username and password**, use:

```sh
Username: elastic
Password: kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

---

## Conclusion
Your **ELK Stack** is now successfully deployed on a **Hetzner machine** with Kubernetes, secured with **NGINX Ingress Controller** and **Let's Encrypt SSL**.

If you face any issues, feel free to reach out to me at **abijithka02@gmail.com**.

Happy Deploying! 🚀


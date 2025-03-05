# Kubernetes Contour Ingress with Auto-Renewing TLS Certificates

### Overview
This repository provides a step-by-step guide to setting up a Kubernetes cluster with **Contour** as the Ingress controller and **cert-manager** for automatically provisioning and renewing TLS certificates from **Let's Encrypt** using the **HTTP-01 challenge**. This setup ensures that your services are securely exposed with HTTPS without manual certificate management.

This guide will help you:
1. Install and configure **Contour** as the Ingress controller.
2. Install **cert-manager** to manage TLS certificates.
3. Use the **HTTP-01 challenge** to automatically provision and renew Let's Encrypt certificates.
4. Expose a sample service (e.g., Airflow dashboard) securely using HTTPS.

---

### Prerequisites

Before proceeding, ensure you have the following:
1. A **Kubernetes cluster** (v1.25 or later).
2. **kubectl** configured to access your cluster.
3. A **domain name** (e.g., `rnd.test`) with DNS records pointing to your cluster's public IP.
4. Helm (v3 or later) installed.



## Steps

### 1. Install Contour Ingress Controller

Contour is a high-performance Ingress controller for Kubernetes. Install it using Kubectl:

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

Verify the installation:

```bash
kubectl get pods,svc -n projectcontour
```

You should see the following:
- 2 Contour pods each with status **Running** and 1/1 **Ready**
- 1+ Envoy pod(s), each with the status **Running** and 2/2 **Ready**

---

### 2. Install cert-manager

cert-manager automates the management and issuance of TLS certificates. Install it using Helm:

```bash
# Add the cert-manager Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true
```

Verify the installation:

```bash
kubectl get pods -n cert-manager
```

---

### 3. Configure a ClusterIssuer for Let's Encrypt

Create a `ClusterIssuer` to issue TLS certificates using Let's Encrypt's HTTP-01 challenge:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: contour  # Use Contour as the Ingress controller
```

Apply the `ClusterIssuer`:

```bash
kubectl apply -f cluster-issuer.yaml
```

> The **letsencrypt-prod-account-key** is a Kubernetes secret that cert-manager creates automatically when you configure a ClusterIssuer or Issuer. It stores the private key for your Let's Encrypt account. You don't need to create this manually—it will be generated and managed by cert-manager.

---

### 4. Expose a Service Using Ingress

Create an Ingress resource to expose a service (e.g., Airflow dashboard) with HTTPS:

#### Example: Airflow Dashboard Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airflow-ingress
  namespace: airflow
  annotations:
    kubernetes.io/ingress.class: contour
    cert-manager.io/cluster-issuer: letsencrypt-prod  # Use the ClusterIssuer
spec:
  tls:
  - hosts:
    - airflow.contour.rnd.test
    secretName: airflow-tls  # Secret to store the TLS certificate
  rules:
  - host: airflow.contour.rnd.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: airflow-webserver  # Replace with your service name
            port:
              number: 8080  # Replace with your service port
```

Apply the Ingress resource:

```bash
kubectl apply -f airflow-ingress.yaml
```

---

### 5. Verify the Setup

1. Check the status of the certificate:

   ```bash
   kubectl get certificates -n airflow
   ```

2. Verify that the Ingress resource is working:

   ```bash
   kubectl get ingress -n airflow
   ```

3. Access your service securely at `https://airflow.contour.rnd.test`.

---

### 6. Automatic Certificate Renewal

cert-manager automatically renews certificates before they expire. You can verify the renewal process by checking the certificate's status:

```bash
kubectl describe certificate airflow-tls -n airflow
```

---

## Troubleshooting

1. **Ingress Not Working**:
   - Ensure the `kubernetes.io/ingress.class: contour` annotation is set.
   - Check the Contour and Envoy logs for errors:
     ```bash
     kubectl logs -n projectcontour -l app=contour
     kubectl logs -n projectcontour -l app=envoy
     ```

2. **Certificate Not Issued**:
   - Verify the `ClusterIssuer` is correctly configured.
   - Check the cert-manager logs:
     ```bash
     kubectl logs -n cert-manager -l app=cert-manager
     ```

3. **DNS Issues**:
   - Ensure your DNS records (e.g., `*.contour.rnd.test`) point to the public IP of the Envoy service.

---

## References
- [Renewing certificate automatically using cert-manager and Let’s Encrypt in a k8s cluster](https://blog.searce.com/renewing-certificate-automatically-using-cert-manager-and-lets-encrypt-prod-in-a-k8s-cluster-858910a45ac6)
- [Installing Cert-Manager and NGINX Ingress with Let’s Encrypt on Kubernetes](https://hbayraktar.medium.com/installing-cert-manager-and-nginx-ingress-with-lets-encrypt-on-kubernetes-fe0dff4b1924)

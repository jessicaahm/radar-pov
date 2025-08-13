# radar-pov

```sh
#minikube start
minikube start --memory=7837 --cpus=4
```

```sh
kubectl get all --all-namespaces
```

```sh
minikube status
```

## Deploy Confluence

```sh
# generate certificates
cd ./vault-radar-demo/confluence/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout selfsigned.key \
  -out selfsigned.crt \
  -config sans.cnf
```

```sh
# Create namespace if it doesn't exist
kubectl create namespace confluence 
```

```sh
cd Confluence/ssl
kubectl create secret tls nginx-tls --cert=selfsigned.crt --key=selfsigned.key -n confluence
```

```sh

```

## Deploy Jira (Simplified Setup)

```sh
# Create namespace if it doesn't exist
kubectl create namespace jira 
# Deploy configmap
kubectl create configmap jira-server-xml --from-file=server.xml=server.xml -n jira
```

```sh
# Generate a private key
openssl genrsa -out tls.key 2048

# Generate a certificate signing request and self-signed certificate
#openssl req -new -x509 -key tls.key -out tls.crt -days 365 -subj "/CN=jira.local"

openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=nginx-service.jira.svc.cluster.local" \
  -addext "subjectAltName=DNS:nginx-service.jira.svc.cluster.local,DNS:jira.local"
```

```sh
# Delete the existing secret if it exists
#kubectl delete secret nginx-tls -n jira --ignore-not-found

# Create the secret from the certificate files
kubectl create secret tls nginx-tls --cert=tls.crt --key=tls.key -n jira
```

```sh {"terminalRows":"14"}
# Apply the manifest (this will update NGINX config)
kubectl apply -f jira-manifest.yaml


```

```sh
kubectl get svc -n jira
kubectl get po -n jira
```

```sh
# Forward HTTPS port from NGINX service to access https://localhost:8443
sudo kubectl port-forward -n jira svc/nginx-service 443:443
```

```sh
# Set up Jira with License key
# Start minikube tunnel (run in background)
minikube tunnel &

# Check if external IP is assigned to LoadBalancer services
kubectl get svc -n jira

# Access via the external IP shown for jira-loadbalancer
# http://EXTERNAL_IP:8080/jira
```

To test scanning, you will have to create issue in Jira board manually [To provide script for this in the future]

## Troubleshooting - Jira

```sh {"terminalRows":"31"}
# Get logs for the Jira pod
kubectl logs -n jira jira-c95667c4c-kphrb  -f
```

```sh
# Access Jira directly via the pod service 
minikube service jira-loadbalancer -n jira --url
```

```sh
# Check probes
kubectl exec -n jira deployment/jira -- curl -I http://localhost:8080/jira/status
```

```sh
# Scale down to delete the deployment
kubectl scale deployment jira -n jira --replicas=0

# Scale up to redeploy
kubectl scale deployment jira -n jira --replicas=1

# Delete Database
kubectl delete pvc postgres-pvc -n jira
kubectl patch pvc postgres-pvc -n jira -p '{"metadata":{"finalizers":null}}' --type=merge

```

```sh
# To redeploy Jira
# Restart NGINX to pick up the new wildcard config
kubectl rollout restart deployment/jira -n jira

# Delete config map
kubectl delete configmap jira-server-xml -n jira --ignore-not-found
```

## Radar deployment

```sh
# Apply agent manifest
kubectl apply -f ./Radar/agent-manifest.yaml

# Delete the existing secret if it exists
kubectl delete secret nginx-tls -n vault-radar --ignore-not-found

# Create the secret from the certificate files
kubectl create secret tls nginx-tls --cert=./Jira/tls.crt --key=./Jira/tls.key -n vault-radar
```

```sh
# Secrets
export HCP_CLIENT_SECRET=$(cat .env | grep HCP_CLIENT_SECRET | cut -d '=' -f2)
export VAULT_RADAR_GIT_TOKEN=$(cat .env | grep VAULT_RADAR_GIT_TOKEN | cut -d '=' -f2)
export JIRA_TOKEN=$(cat .env | grep JIRA_TOKEN | cut -d '=' -f2)
export CONFLUENCE_TOKEN=$(cat .env | grep CONFLUENCE_TOKEN | cut -d '=' -f2)

# Base64 encode the client ID and secret    
export HCP_CLIENT_ID_BASE64=$(echo -n "$HCP_CLIENT_ID" | base64)
export HCP_CLIENT_SECRET_BASE64=$(echo -n "$HCP_CLIENT_SECRET" | base64)
export VAULT_RADAR_GIT_TOKEN_B64=$(echo -n "$VAULT_RADAR_GIT_TOKEN" | base64)
export CONFLUENCE_TOKEN_B64=$(echo -n "$CONFLUENCE_TOKEN" | base64)
export JIRA_TOKEN_B64=$(echo -n "$JIRA_TOKEN" | base64)

# Create a Kubernetes secret with the base64 encoded values
cat <<EOF > ./Radar/agent-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-radar-secrets
  namespace: vault-radar
  labels:
    app: vault-radar-agent
type: Opaque
data:
  HCP_CLIENT_SECRET: $HCP_CLIENT_SECRET_BASE64
  VAULT_RADAR_GIT_TOKEN: $VAULT_RADAR_GIT_TOKEN_B64
  CONFLUENCE_TOKEN: $CONFLUENCE_TOKEN_B64
  JIRA_TOKEN: $JIRA_TOKEN_B64
EOF

```

```sh
HCP_CLIENT_ID=$(cat .env | grep HCP_CLIENT_ID | cut -d '=' -f2)
HCP_PROJECT_ID=$(cat .env | grep HCP_PROJECT_ID | cut -d '=' -f2)
HCP_RADAR_AGENT_POOL_ID=$(cat .env | grep HCP_RADAR_AGENT_POOL_ID | cut -d '=' -f2)

cat <<EOF > ./Radar/agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-radar-agent
  namespace: vault-radar
  labels:
    app: vault-radar-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault-radar-agent
  template:
    metadata:
      labels:
        app: vault-radar-agent
    spec:
      serviceAccountName: vault-radar-agent
      automountServiceAccountToken: true
      containers:
        - name: vault-radar-agent
          image: vault-radar-custom:latest
          command: ["vault-radar"]
          args: ["agent", "exec"]
          imagePullPolicy: Never
          tty: true
          resources:
            limits:
              cpu: 1000m
              memory: 1024Mi
            requests:
              cpu: 100m
              memory: 512Mi
          env:
            - name: HCP_PROJECT_ID
              value: ${HCP_PROJECT_ID}
            - name: HCP_RADAR_AGENT_POOL_ID
              value: ${HCP_RADAR_AGENT_POOL_ID}
            - name: HCP_CLIENT_ID
              value: ${HCP_CLIENT_ID}
            - name: HCP_CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: vault-radar-secrets
                  key: HCP_CLIENT_SECRET
            - name: VAULT_RADAR_GIT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: vault-radar-secrets
                  key: VAULT_RADAR_GIT_TOKEN
            - name: JIRA_TOKEN
              valueFrom:
                secretKeyRef:
                  name: vault-radar-secrets
                  key: JIRA_TOKEN
         # SSL Certificate configuration
            - name: SSL_CERT_FILE
              value: "/etc/ssl/certs/jira-ca.crt"
            - name: REQUESTS_CA_BUNDLE
              value: "/etc/ssl/certs/jira-ca.crt"
            - name: CURL_CA_BUNDLE
              value: "/etc/ssl/certs/jira-ca.crt"
          volumeMounts:
            - name: jira-ssl-cert
              mountPath: /etc/ssl/certs/jira-ca.crt
              subPath: tls.crt
              readOnly: true
      volumes:
        - name: jira-ssl-cert
          secret:
            secretName: nginx-tls
            items:
            - key: tls.crt
              path: tls.crt
EOF

```

```sh
# Apply all YAML files in the Radar directory
cd ./Radar
kubectl apply -f .
```

```sh
# Or apply specific files one by one:
# kubectl apply -f agent-manifest.yaml
echo "HCP_CLIENT_SECRET_B64=$(echo -n "$HCP_CLIENT_SECRET" | base64)"
echo "HCP_CLIENT_ID_B64=$(echo -n "$HCP_CLIENT_ID" | base64)"
echo "VAULT_RADAR_GIT_TOKEN_B64=$(echo -n "$VAULT_RADAR_GIT_TOKEN" | base64)"

# Store base64 values in variables
export HCP_CLIENT_SECRET_B64=$(echo -n "$HCP_CLIENT_SECRET" | base64)
export HCP_CLIENT_ID_B64=$(echo -n "$HCP_CLIENT_ID" | base64)
```

Build Dockerfile for self-signed cert

```sh
# Build Custom Docker Image
cd Radar
docker build -t vault-radar-custom:latest .

# Load image into Minikube
minikube image load vault-radar-custom:latest

# Verify the image is loaded
minikube image ls | grep vault-radar-custom

```

# Troubleshooting

https://nginx-service.jira.svc.cluster.local

http://jira-service.jira.svc.cluster.local:8080/jira

```sh {"terminalRows":"38"}
kubectl logs -n vault-radar deployment/vault-radar-agent
```

```sh
# Check if secrets/env variables are set correctly
kubectl exec -it -n vault-radar deployment/vault-radar-agent -- /bin/sh
```

```sh
# wget webhook test

wget --header="Authorization: Bearer $JIRA_TOKEN" https://nginx-service.jira.svc.cluster.local/rest/jira-webhook/1.0/webhooks
```

### Error Message

time="2025-08-13T05:34:06Z" level=info msg="received response" job_id=cecd5af4-9536-42df-ac6d-e23e4e086ffb method=GET status_code=404 url="https://nginx-service.jira.svc.cluster.local/rest/jira-webhook/1.0/webhooks" worker="HTTP job"
/rest/webhooks/1.0/webhook
# radar-pov

> This installation is done in minikube. May have memory issue if both confluence and jira is up at once.

Other possible way to use helm chart: https://atlassian.github.io/data-center-helm-charts/userguide/INSTALLATION/

Configure Jira to run behind a nginx reverse proxy: https://support.atlassian.com/jira/kb/configure-jira-to-run-behind-a-nginx-reverse-proxy/

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

## Confluence [Not working - This should be moved to use helm/eks]

Reference: https://hub.docker.com/r/atlassian/confluence-server

```sh
# Generate a private key
openssl genrsa -out tls.key 2048

# Generate a certificate signing request and self-signed certificate
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=nginx-service.confluence.svc.cluster.local" \
  -addext "subjectAltName=DNS:nginx-service.confluence.svc.cluster.local,DNS:confluence.local"
```

```sh
#Create namespace
kubectl create namespace confluence

# Deploy configmap
cd ./Confluence
kubectl create configmap confluence-server-xml-file --from-file=server.xml=server.xml -n confluence
```

```sh
# Verify ConfigMap was created
kubectl get configmap confluence-server-xml-file -n confluence

# View the full ConfigMap YAML (including the server.xml content)
kubectl get configmap confluence-server-xml-file -n confluence -o yaml

# Delete the ConfigMap
kubectl delete configmap confluence-server-xml-file -n confluence
```

```sh {"terminalRows":"11"}
# Apply the Confluence manifest
cd ./Confluence
kubectl apply -f confluence-manifest.yaml -n confluence
```

```sh
# Start minikube tunnel (run in background)
minikube tunnel&

sudo kubectl port-forward -n confluence svc/confluence-loadbalancer 8090:8090
sudo kubectl port-forward -n confluence svc/nginx-service 443:443
```

```sh
kubectl get svc -n confluence
kubectl get pods -n confluence
kubectl get endpoints -n confluence
```

```sh
kubectl exec -it -n confluence deployment/confluence-server -- /bin/sh

# Verify it is running
curl http://localhost:8090/status

```

## Troubleshooting to reduce size (memory)

```sh
# ...existing code...
# Apply the Confluence manifest
cd ./Confluence

# Scale down deployment to 0 replicas
kubectl scale deployment confluence-server -n confluence --replicas=0 
kubectl scale deployment postgresql -n confluence --replicas=0

# Delete existing PVCs if they exist (to allow size reduction)
kubectl delete pvc confluence-home-pvc -n confluence --ignore-not-found
kubectl delete pvc postgres-pvc -n confluence --ignore-not-found

# Force remove PVC finalizers if stuck
kubectl patch pvc confluence-home-pvc -n confluence -p '{"metadata":{"finalizers":null}}' --type=merge 2>/dev/null || true
kubectl patch pvc postgres-pvc -n confluence -p '{"metadata":{"finalizers":null}}' --type=merge 2>/dev/null || true

# Apply the manifest with new PVC sizes
kubectl apply -f confluence-manifest.yaml -n confluence
kubectl scale deployment postgresql -n confluence --replicas=1
kubectl scale deployment confluence-server -n confluence --replicas=1 2>/dev/null || true

```

```sh {"terminalRows":"15"}
# check svc and pods    
kubectl get svc -n confluence
kubectl get pods -n confluence

# https://nginx-service.confluence.svc.cluster.local
# https://confluence-service.confluence.svc.cluster.local 
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

## Radar deployment

```sh
# Build docker container to add the self-signed certificate to agent
cd ./Radar
docker build -t vault-radar-custom:latest .

# Add to minikube the new docker container
minikube image load vault-radar-custom:latest   
```

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
export HCP_CLIENT_ID=$(cat .env | grep HCP_CLIENT_ID | cut -d '=' -f2)

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
# Or apply specific files one by one:
# kubectl apply -f agent-manifest.yaml
echo "HCP_CLIENT_SECRET_B64=$(echo -n "$HCP_CLIENT_SECRET" | base64)"
echo "HCP_CLIENT_ID_B64=$(echo -n "$HCP_CLIENT_ID" | base64)"
echo "VAULT_RADAR_GIT_TOKEN_B64=$(echo -n "$VAULT_RADAR_GIT_TOKEN" | base64)"

# Store base64 values in variables
export HCP_CLIENT_SECRET_B64=$(echo -n "$HCP_CLIENT_SECRET" | base64)
export HCP_CLIENT_ID_B64=$(echo -n "$HCP_CLIENT_ID" | base64)
```

# Troubleshooting

Jira

https://nginx-service.jira.svc.cluster.local

http://jira-service.jira.svc.cluster.local:8080/jira

Confluence
https://nginx-service.confluence.svc.cluster.local

```sh {"terminalRows":"38"}
kubectl logs -n vault-radar deployment/vault-radar-agent
```

```sh
# Check if secrets/env variables are set correctly
kubectl exec -it -n vault-radar deployment/vault-radar-agent -- /bin/sh
```

```sh
# URL

```

```sh
# Create a Policy
vault policy write -namespace=admin radar radar.hcl 

# Test Policy[To convert to script]
vault secrets enable -namespace=admin -path=confluence kv-v2
vault secrets enable -namespace=admin -path=jira kv-v2

vault kv put jira/exampleapp/1 \
   password='Password123'

vault kv put static/exampleapp/1 \
   username='jalbertson' \
   password='b3stp@stw00rd3vA!'

vault kv list jira/exampleapp
vault kv list static/exampleapp

vault kv get jira/exampleapp/1
vault kv get jira/exampleapp/1

vault kv get static/exampleapp/2
vault kv get static/exampleapp/1

```

```sh
# Create a Kubernetes Secret for the vault-radar-agent Kubernetes service account.
kubectl create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: vault-radar-agent
  namespace: vault-radar
  annotations:
    kubernetes.io/service-account.name: vault-radar-agent
type: kubernetes.io/service-account-token
---
EOF
```

```sh
kubectl proxy --disable-filter=true # ENSURE THIS IS RUNNING IN BACKGROUND
```

```sh
ngrok http --url http:// 127.0.0.1:8001  # ENSURE THIS IS RUNNING IN BACKGROUND
```

```sh
# Decode env variable 
export K8S_URL="http://93599f08af82.ngrok-free.app" # Replace with your actual ngrok URL
export VAULTSA_SECRET=$(kubectl get secret --namespace vault-radar vault-radar-agent --output json | jq -r '.data') \
    && echo $VAULTSA_SECRET
export K8S_CA_CRT=$(echo $VAULTSA_SECRET | jq -r '."ca.crt"' | base64 -d) && echo $K8S_CA_CRT
export VAULTSA_TOKEN=$(echo $VAULTSA_SECRET | jq -r '.token' | base64 -d) && echo $VAULTSA_TOKEN
```

```sh
vault auth enable kubernetes
```

```sh
vault write auth/kubernetes/config \
 token_reviewer_jwt=$VAULTSA_TOKEN \
 kubernetes_host=$K8S_URL \
 kubernetes_ca_cert=$K8S_CA_CRT
```

```sh
vault write auth/kubernetes/role/vault-radar-agent-role \
    bound_service_account_names=vault-radar-agent \
    bound_service_account_namespaces=vault-radar \
    policies=radar 

# Delete Role
vault delete -namespace=admin auth/kubernetes/role/vault-radar-agent-role

# Read role
vault read -namespace=admin -format=json auth/kubernetes/role/vault-radar-agent-role

```

```sh
vault write -namespace="admin" -format=json auth/kubernetes/login \
        role=vault-radar-agent \
        jwt="$VAULTSA_TOKEN"


TOKEN_PAYLOAD=$(echo "$TOKEN" | cut -d '.' -f2 | tr '_-' '/+' | awk '{l=length($0)%4; if (l==2) print $0 "=="; else if (l==3) print $0 "="; else print $0; }')
echo "$TOKEN_PAYLOAD" | base64 --decode | jq .


```

```sh
export VAULT_SA_NAME=$(kubectl -n vault-radar get sa vault-radar-agent -o jsonpath="{.secrets[0].name}")

```

## GitHub Configuration 

Go to and configure : https://github.com/apps/hashicorp-vault-radar-checks-app

Example Installation Page: https://github.com/organizations/jessicahm-org/settings/installations/74288876

Create a PAT in GITHUB: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token

Branch Protection: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

Ensure PR Check are enabled
<img src="PRCheck.png"></img>

```sh
kubectl exec -it -n vault-radar deployment/vault-radar-agent -- /bin/sh
```

## Clean-up

```sh
kubectl delete all --all -n confluence
kubectl delete all --all -n jira

# Clean up all resources in minikube
minikube delete

```

```sh
# Method 3: Create secret from files (if you have secret files)
# kubectl create secret generic vault-radar-secrets \
#   --from-file=HCP_CLIENT_SECRET=./hcp-secret.txt \
#   --from-file=VAULT_RADAR_GIT_TOKEN=./git-token.txt \
#   --namespace=vault-radar

# Verify the secret was created
kubectl get secrets -n vault-radar
kubectl describe secret vault-radar-secrets -n vault-radar
```

## 
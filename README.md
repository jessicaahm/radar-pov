# radar-pov

```sh
minikube start
```

```sh
kubectl get all --all-namespaces
```

```sh
minikube status
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
openssl req -new -x509 -key tls.key -out tls.crt -days 365 -subj "/CN=jira.local"
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

# To redeploy Jira
# Restart NGINX to pick up the new wildcard config
kubectl rollout restart deployment/jira -n jira

# Delete config map
kubectl delete configmap jira-server-xml -n jira --ignore-not-found
```

```sh
kubectl get svc -n jira
```

```sh
# Forward HTTPS port from NGINX service to access https://localhost:8443
kubectl port-forward -n jira svc/nginx-service 8443:443
```

To test scanning, you will have to create issue in Jira board manually [To provide script for this in the future]

## Troubleshooting - Jira

```sh {"terminalRows":"31"}
# Get logs for the Jira pod
kubectl logs -n jira jira-6cc485495c-q5hcn  -f
```

```sh
# Access Jira directly via the pod service 
minikube service jira-loadbalancer -n jira --url
```

## Radar deployment

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
  replicas: 2
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
          image: docker.io/hashicorp/vault-radar:latest
          command: ["vault-radar"]
          args: ["agent", "exec"]
          imagePullPolicy: Always
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
EOF

```

```sh
cd ./Radar
kubectl apply -f .
```

```sh
# Check if secrets were created successfully
kubectl get secret vault-radar-secrets -n vault-radar -o yaml
```
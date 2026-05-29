# Secrets Management with External Secrets Operator

## What Is ESO

External Secrets Operator (ESO) is a Kubernetes controller that syncs secrets
from external providers (AWS Secrets Manager, Vault, etc.) into native
Kubernetes Secrets automatically.

```
AWS Secrets Manager
  dev/my-app → {"db.password": "secret123", "db.user": "user123"}
        │
        │  ESO fetches (default: every 1h)
        ▼
  Kubernetes Secret
  my-app → application-secret.properties: "db.password=secret123"
        │
        │  mounted as volume
        ▼
  Pod file: /config/application-secret.properties
```

---

## Why ESO Works Without Extra AWS Config on EC2

ESO runs as a **Pod**. Any pod on an EC2 node automatically inherits the
EC2 instance IAM role via the metadata service at `169.254.169.254`.

ESO uses this to call AWS Secrets Manager — no AWS access keys, no IRSA,
no extra credential config needed. The EC2 IAM role is enough.

```
EC2 Node (IAM role: SecretsManagerReadOnly)
└── ESO Pod
    └── GET http://169.254.169.254/...  ← gets temp AWS credentials
    └── Calls Secrets Manager API      ← fetches secret values
    └── Creates K8s Secret             ← application reads this
```

---

## IAM Role Requirements

The EC2 nodes IAM role needs:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "secretsmanager:DescribeSecret"
  ],
  "Resource": "arn:aws:secretsmanager:us-east-1:YOUR_ACCOUNT:secret:*"
}
```

---

## AWS Secrets Manager Structure

Store all secrets for one service in a **single JSON secret**:

```bash
# Create secret
aws secretsmanager create-secret \
  --name dev/my-app \
  --region us-east-1 \
  --secret-string '{
    "db.password": "secret",
    "db.username": "user",
    "api.key": "key123"
  }'

# Update secret (add new key)
EXISTING=$(aws secretsmanager get-secret-value \
  --secret-id dev/my-app --region us-east-1 \
  --query 'SecretString' --output text)

UPDATED=$(echo $EXISTING | jq '. + {"new.key": "value"}')

aws secretsmanager update-secret \
  --secret-id dev/my-app \
  --region us-east-1 \
  --secret-string "$UPDATED"
```

Naming convention: `{env}/{service-name}` → `dev/my-app`

---

## Installation

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace \
  --set installCRDs=true

# Verify
kubectl get pods -n external-secrets-system
```

See [examples/install-eso.yml](examples/install-eso.yml) for Ansible.

---

## Helm Chart Patterns

Two resources per service: **SecretStore** + **ExternalSecret**.

### SecretStore

```yaml
# templates/secret-store.yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: {{ .Release.Name }}-secret-store
  namespace: {{ .Values.namespace }}
spec:
  provider:
    aws:
      service: SecretsManager
      region: {{ .Values.externalSecrets.region }}
      # No auth needed — External Secret Operator (ESO) detects secret store and uses EC2 IAM role automatically
```

### ExternalSecret

See [examples/external-secret.yaml](examples/external-secret.yaml)

### values.yaml

```yaml
externalSecrets:
  env: dev                # AWS path prefix → dev/my-app
  region: us-east-1
  store: my-app-secret-store
  targetSecretName: my-app
  keys:
    # k8s_key: aws_json_property
    # k8s_key: underscores only (dots invalid in K8s secret keys)
    db_password: db.password
    db_username: db.username
    api_key: api.key
```

Adding a new secret = one line in `values.yaml`. No template changes needed.

---

## Spring Boot Integration

Mount the K8s Secret as a volume alongside the ConfigMap:

```yaml
# deployment.yaml
spec:
  containers:
    - name: app
      env:
        - name: SPRING_CONFIG_LOCATION
          value: "file:/config/application.properties"
      volumeMounts:
        - name: config
          mountPath: /config
  volumes:
    - name: config
      projected:
        sources:
          - configMap:
              name: my-app      # application.properties
          - secret:
              name: my-app      # application-secret.properties (from ESO)
```

`application.properties` imports the secret file:

```properties
spring.config.import=optional:file:/config/application-secret.properties
```

---

## Key Lessons Learned

**1. Helm eats ESO template variables**

Both Helm and ESO use `{{ }}` syntax. Without escaping, Helm consumes
ESO variables before ESO sees them — leaving empty values.

```yaml
# WRONG — Helm renders this as empty:
db.password={{ .db_password }}

# CORRECT — Helm outputs literal {{ }} for ESO to process:
db.password={{ "{{" }} .db_password {{ "}}" }}
```

Verify the rendered ExternalSecret has literal `{{ }}`:
```bash
kubectl get externalsecret my-app-external-secret -n my-namespace \
  -o jsonpath='{.spec.target.template.data.application-secret\.properties}'
# Must show: db.password={{ .db_password }}
# Not:       db.password=
```

**2. Use `range $key, $val` consistently**

```yaml
# WRONG — $val undefined in second range block:
{{- range $key, $val := .Values.externalSecrets.keys }}   # defines $val
  ...
{{- range .Values.externalSecrets.keys }}                  # $val not defined!
  property: {{ $val }}                                     # ERROR

# CORRECT — same syntax in both blocks:
{{- range $key, $val := .Values.externalSecrets.keys }}
  property: {{ $val }}
{{- end }}
```

**3. Single JSON secret with `property` field**

When your AWS secret stores values as JSON:
```json
{"db.password": "secret123", "db.user": "user123"}
```

ESO returns the whole JSON string unless you specify `property`:
```yaml
remoteRef:
  key: dev/my-app
  property: db.password    # extracts just "secret123"
```

**4. Secret namespace must match pod namespace**

K8s secrets are namespace-scoped. Create the ExternalSecret in the same
namespace as the pod that consumes it.

---

## Troubleshooting

```bash
# Check sync status
kubectl get externalsecret -n your-namespace

# Check SecretStore is valid
kubectl get secretstore -n your-namespace

# Check actual secret values
kubectl get secret my-app -n your-namespace \
  -o jsonpath='{.data.application-secret\.properties}' | base64 -d

# Force resync
kubectl annotate externalsecret my-app-external-secret \
  -n your-namespace \
  force-sync=$(date +%s) --overwrite

# ESO controller logs
kubectl logs -n external-secrets-system \
  -l app.kubernetes.io/name=external-secrets --tail=50
```
## Examples
- [`examples/secret-store.yaml`](examples/secret-store.yaml) — Helm template for SecretStore
- [`examples/external-secret.yaml`](examples/external-secret.yaml) — Helm template for ExternalSecret
- [`examples/install-eso.yml`](examples/install-eso.yml) — Ansible playbook to install ESO

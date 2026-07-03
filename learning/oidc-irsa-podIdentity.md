# OIDC, IRSA, and Pod Identity — Complete Reference

> Written from the perspective of a 20+ year DevSecOps engineer.
> Designed to be read top to bottom, once. Every concept builds on the previous one.

---

## How to Read This Document

This document answers one question end to end:

```
How does a pod running in Kubernetes
prove its identity to AWS
and get the right permissions
without hardcoding any credentials?
```

It builds from the problem → the concepts → the two solutions → how they compare → secrets management → practical guidance for this project. Do not skip sections. Each one is a foundation for the next.

---

## Part 1 — The Problem

---

### 1.1 Every Pod That Talks to AWS Needs Credentials

Suppose you have a payments pod that needs to read from DynamoDB. To make an AWS API call, the pod needs credentials. You have three options:

```
Option 1 — Hardcode credentials in the pod
AWS_ACCESS_KEY = "AKIA..."    ← leaks in git history, logs, env dumps
AWS_SECRET_KEY = "xxxx"       ← rotates manually, one leak = account exposed

Option 2 — Use the node's IAM role
Pod inherits the EC2 node's permissions
Works, but dangerously broad — explained below

Option 3 — Give each pod its own IAM role
Pod gets exactly the permissions it needs, nothing more
This is what IRSA and Pod Identity both achieve
```

Options 1 and 2 both violate **Least Privilege** — the foundational security principle that every identity gets the minimum permissions needed to do its job, and nothing more.

---

### 1.2 Why the Node Role is Dangerous — The Blast Radius Problem

Option 2 feels convenient. The node already has an IAM role. Why not just use it?

Here is what the node role actually looks like in practice:

```
Node IAM Role has:
- S3 full access       (needed by the backup pod)
- DynamoDB full access (needed by the payments pod)
- SES full access      (needed by the email pod)
- CloudWatch write     (needed by the logging pod)

Every pod on this node inherits ALL of these permissions:

Pod A (payments)  → gets S3, DynamoDB, SES, CloudWatch
Pod B (frontend)  → gets S3, DynamoDB, SES, CloudWatch
Pod C (logging)   → gets S3, DynamoDB, SES, CloudWatch
```

Now suppose Pod B (frontend) gets compromised by an attacker:

```
Attacker now has:
- S3 full access      → read, write, delete any bucket
- DynamoDB full access → read, modify, delete any table
- SES full access     → send emails from your domain
- CloudWatch write    → tamper with your monitoring

One compromised frontend pod = entire AWS account potentially exposed
```

This is called **blast radius** — how much damage one compromise can cause. With a shared node role, the blast radius is your entire AWS account.

With per-pod roles:

```
Pod A (payments)  → DynamoDB access ONLY on the payments table
Pod B (frontend)  → S3 read on ONE specific bucket ONLY
Pod C (logging)   → CloudWatch write ONLY

Pod B gets compromised:
Attacker has S3 read on one bucket
Cannot touch DynamoDB, cannot touch other buckets, cannot send emails
Blast radius contained to exactly what that pod was supposed to do
```

This is the problem IRSA and Pod Identity both solve. Each pod gets its own scoped identity. The node role stays minimal — just enough to run Kubernetes, nothing more.

---

## Part 2 — Understanding the Technology Stack

Before diving into IRSA and Pod Identity, you need to understand three concepts they are built on: HTTP, OIDC, and JWT tokens. Each one solves a specific gap the previous one left open.

---

### 2.1 HTTP — Just a Delivery System

You already know HTTP. When a pod calls an AWS API, it sends something like:

```
GET /dynamodb/payments-table
Host: dynamodb.ap-south-1.amazonaws.com
```

But stop and ask: **who sent this request?**

HTTP has no answer. It only carries data from A to B. It does not know who you are, whether you are authorised, or whether your request is legitimate.

Think of HTTP like a postal service:

```
Someone
   ↓
Letter (the HTTP request)
   ↓
Post Office (the network)
   ↓
Destination (AWS API)
```

The post office delivers the letter. It does not verify who wrote it. Anyone can drop a letter in the box claiming to be anyone.

**Gap:** AWS needs to know who is making a request. HTTP gives it no way to find out.

---

### 2.2 Authentication — Proving Who You Are

Suppose you visit Gmail. You type your email and password. Google verifies them. Now Google knows:

```
You are Alice.
```

Notice what Google learned — WHO you are. It has not yet decided what you can do. That is a completely separate question.

This is **authentication** — proving identity.

**New gap:** Alice uses many applications. Every application would need its own separate password. And every application would need to verify those passwords itself. That is a maintenance nightmare and a security disaster. There must be a better way.

---

### 2.3 OpenID Connect (OIDC) — Delegated Identity

Suppose Alice visits Canva. Instead of creating yet another password, Canva offers:

```
Login with Google
```

Here is what happens:

```
Alice visits Canva
    │
    │ clicks "Login with Google"
    ▼
Canva sends Alice to Google
    │
    │ Alice enters her Google password
    ▼
Google verifies Alice's identity
    │
    │ Google sends Canva a signed statement:
    │ "I have verified this person.
    │  Name: Alice
    │  Email: alice@gmail.com
    │  I guarantee this identity."
    ▼
Canva trusts Google's word
Canva logs Alice in
```

Canva never saw Alice's password. Canva simply trusted Google's signed statement about who Alice is. That signed statement is called an **ID Token**.

This is **OpenID Connect** — a standard that lets one trusted authority vouch for someone's identity to a third party. The third party trusts the authority, not the password.

---

### 2.4 OAuth 2.0 — Delegated Authorization (Different Problem)

Now Alice wants Canva to access her Google Drive. That is a completely different question — not "who are you" but "what are you allowed to do."

Google asks Alice:

```
Canva wants permission to:
✓ Read your Google Drive files
✓ Create new files in Drive

Allow or Deny?
```

Alice allows it. Google gives Canva an **Access Token** that says "Canva can read Drive on Alice's behalf." Canva presents this token to Drive with every request.

This is **OAuth 2.0** — delegated authorization.

The critical distinction:

```
OIDC       → WHO is this person?       "This is Alice, verified by Google"
OAuth 2.0  → WHAT can they do?         "Canva can read Drive"
```

Completely separate questions. OIDC is about identity. OAuth is about permissions.

**Why OIDC is built on OAuth 2.0:** OAuth already solved the secure communication channel between parties. Rather than inventing a new protocol, OIDC reuses OAuth's plumbing and adds one new thing — an ID Token:

```
OAuth 2.0 gives:   Access Token   (what you can do)
OIDC adds:         ID Token       (who you are)

OIDC = OAuth 2.0 (the plumbing) + ID Token (the identity claim)
```

---

### 2.5 JWT Token — What Is Actually Inside

The ID Token that OIDC issues is a **JWT** — JSON Web Token. This is the actual object that gets passed around to prove identity. Understanding its structure is critical because it is what Kubernetes generates for your pods.

A JWT has three parts separated by dots:

```
header.payload.signature
```

**Header** — metadata about the token:
```json
{
  "alg": "RS256",
  "kid": "abc123"
}
```
- `alg` — the algorithm used to create the signature
- `kid` — which specific key was used (clusters rotate keys periodically)

**Payload** — the actual identity claims:
```json
{
  "iss": "https://oidc.eks.ap-south-1.amazonaws.com/id/ABC123",
  "sub": "system:serviceaccount:payments-namespace:payments-sa",
  "aud": ["sts.amazonaws.com"],
  "exp": 1735689600,
  "iat": 1735686000,
  "kubernetes.io/serviceaccount/name": "payments-sa",
  "kubernetes.io/serviceaccount/namespace": "payments-namespace"
}
```

| Field | Meaning | Why It Matters |
|-------|---------|----------------|
| `iss` | Who issued this token | AWS uses this to find the right OIDC provider to verify against |
| `sub` | Who this token is for | AWS checks this against trust policy conditions |
| `aud` | Who this token is addressed to | AWS rejects tokens not addressed to it |
| `exp` | When token expires | Expired tokens rejected — replay attacks blocked |
| `iat` | When token was created | Freshness check |

**Signature** — cryptographic proof:
```
Created using the issuer's PRIVATE key
Proves header + payload were not tampered with
Cannot be forged without the private key
```

**Critical distinction — base64 is NOT encryption:**
```
The JWT payload is base64 encoded, not encrypted
Anyone can decode it instantly:

echo "c3lzdGVtOnNlcnZpY2VhY2NvdW50..." | base64 -d
→ system:serviceaccount:payments-namespace:payments-sa

The payload is meant to be readable — it is a public claim
The SIGNATURE is what makes it trustworthy, not secrecy
```

---

### 2.6 Now Map Everything to EKS

Every concept above has a direct EKS equivalent:

```
Alice                   → your pod
                          the thing that needs to prove its identity

Google                  → your EKS cluster's OIDC provider
                          the trusted authority that issues and signs tokens

Canva                   → AWS IAM / STS
                          the party that needs to verify identity

"Login with Google"     → pod presents JWT token to AWS STS

Google's ID Token       → the JWT token Kubernetes generates for the pod
                          signed by the cluster's private key

"Canva trusts Google"   → AWS trusts your cluster's OIDC provider
                          because YOU registered it in IAM

Access to Drive         → temporary AWS credentials
                          what the pod is allowed to do after identity verified
```

The "Login with Google" flow you already understand intuitively is exactly what happens between your pod and AWS STS. EKS is Google. Your pod is Alice. AWS IAM is Canva.

Every EKS cluster gets an OIDC issuer URL automatically — a unique address that identifies it as an identity authority:

```
https://oidc.eks.ap-south-1.amazonaws.com/id/ABC123UNIQUEID
                                               ^^^^^^^^^^^^^^^
                                               unique per cluster
                                               generated automatically by EKS
```

This URL is the cluster's "Google" — the trusted party that issues and signs pod identity tokens.

---

## Part 3 — Solution 1: IRSA

---

### 3.1 What IRSA Is

IRSA = IAM Roles for Service Accounts. Introduced by AWS in 2019.

The core idea: **the pod proves its own identity** by presenting a JWT token that Kubernetes signed. AWS verifies the signature and issues temporary credentials scoped to that pod's role.

---

### 3.2 One-Time Setup

Three things to configure before any pod can use IRSA:

**Step 1 — Register the OIDC provider in IAM**

EKS gives you the issuer URL automatically. But AWS IAM does not trust it until you explicitly register it:

```python
# In your cluster.py — what you built in Milestone 2
aws.iam.OpenIdConnectProvider(
    url=cluster.identities[0].oidcs[0].issuer,
    client_id_lists=["sts.amazonaws.com"],
    thumbprint_lists=["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"],
)
```

Without this step, AWS has never heard of your cluster's OIDC provider. All tokens will be rejected. This is equivalent to Canva never configuring "trust Google as an identity provider."

**Step 2 — Create an IAM role with an OIDC trust policy**

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/ABC123"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.../id/ABC123:sub": "system:serviceaccount:payments-namespace:payments-sa",
      "oidc.eks.../id/ABC123:aud": "sts.amazonaws.com"
    }
  }
}
```

This policy says: only allow this role to be assumed if the token comes from our OIDC provider AND is for the specific service account in the specific namespace. Scoped to one pod identity.

**Step 3 — Annotate the Kubernetes service account**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-sa
  namespace: payments-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/payments-role
```

This tells Kubernetes: when a pod uses this service account, generate a JWT token and mount it in the pod.

---

### 3.3 What Happens at Runtime — Every Time a Pod Starts

```
1. payments pod starts using payments-sa service account

2. Kubernetes sees the role-arn annotation on the service account

3. Kubernetes generates a JWT token for this pod
   Signed with the cluster's PRIVATE key
   Payload includes:
     iss → your cluster's OIDC URL
     sub → system:serviceaccount:payments-namespace:payments-sa

4. Token mounted into pod at:
   /var/run/secrets/eks.amazonaws.com/serviceaccount/token

5. Pod's AWS SDK starts up
   Finds the token automatically (SDK knows where to look)

6. SDK calls AWS STS:
   "Here is my JWT token.
    I want credentials for arn:aws:iam::123456789012:role/payments-role"

7. STS verifies the token (full detail in section 3.4 below)

8. STS issues temporary credentials
   Valid for 1 hour, scoped to what payments-role allows

9. Pod uses those credentials to call DynamoDB
   Cannot call S3, cannot call SES, cannot touch anything else
```

---

### 3.4 How STS Verifies the Token — The Security Mechanism

This is the most important section. Understanding these six steps is understanding why IRSA cannot be forged.

**Step 1 — STS reads the `iss` field from the token payload**
```
Token claims: "iss": "https://oidc.eks.ap-south-1.amazonaws.com/id/ABC123"

This is just a CLAIM at this point — unverified
STS has not trusted anything yet
```

**Step 2 — STS checks if this OIDC URL is registered in IAM**
```
STS looks up its trusted OIDC providers list

Is https://oidc.eks.ap-south-1.amazonaws.com/id/ABC123 registered?

YES → continue
NO  → reject immediately — "I do not trust this issuer"
```

This is why you must create the OIDC provider in IAM. Without registration, every token from your cluster is rejected here.

**Step 3 — STS fetches the public key from the OIDC provider**
```
Every OIDC provider exposes a public discovery endpoint:
https://oidc.eks.../id/ABC123/.well-known/openid-configuration

This returns the JWKS endpoint (JSON Web Key Set)
JWKS contains the PUBLIC KEYS of this cluster

STS fetches the public key matching the "kid" in the token header
```

**Step 4 — STS verifies the cryptographic signature**
```
Token was signed with the cluster's PRIVATE key
STS now has the cluster's PUBLIC key

verify(header + payload, signature, public_key)

Valid   → this token was genuinely signed by that cluster
          nothing in the payload was tampered with
          TRUST IT

Invalid → token is forged or modified
          REJECT
```

**Step 5 — STS checks the trust policy conditions**
```
Token is cryptographically verified as genuine

Now STS checks the IAM role trust policy:

Condition requires:
  sub = system:serviceaccount:payments-namespace:payments-sa

Token's sub field:
  system:serviceaccount:payments-namespace:payments-sa

MATCH → proceed

Wrong namespace or wrong service account:
  NO MATCH → reject — "this pod is not authorised for this role"
```

**Step 6 — STS issues credentials**
```
All checks passed
Temporary credentials issued for payments-role
Expire in 1 hour
Scoped to what payments-role permits
```

**Why this is unbreakable:**

| Attack | Why It Fails |
|--------|-------------|
| Forge a token | No cluster private key → step 4 fails |
| Register a fake OIDC provider | No AWS account access → step 2 fails |
| Replay an expired token | `exp` checked by STS → rejected |
| Tamper with the `sub` field | Changing payload breaks signature → step 4 fails |
| Steal token, use for different service | `aud` is `sts.amazonaws.com` specifically, cannot reuse elsewhere |

---

### 3.5 The Thumbprint — Why It Exists

In step 3, STS connects to your OIDC provider URL over TLS. It needs to verify this TLS connection is genuine — not someone intercepting it with a fake certificate.

```
Thumbprint = SHA1 fingerprint of the ROOT CA certificate
             of the OIDC provider's TLS certificate chain

For EKS in any region, this value is always:
9e99a48a9960b14926bb7f3b02e22da2b0ab7280

Stable across all regions — safe to hardcode
```

Same concept as your browser showing a padlock before trusting a website. The thumbprint ensures STS is talking to the real OIDC provider, not an impersonator.

---

### 3.6 Why You Must Create the OIDC Provider Manually

This is the most common misunderstanding about IRSA.

```
EKS creates AUTOMATICALLY:
✓ The OIDC issuer URL (just a string)
✓ The private/public key pair
✓ The .well-known discovery endpoint

EKS does NOT create automatically:
✗ The IAM OIDC Provider resource

YOU must create it — in Pulumi:
aws.iam.OpenIdConnectProvider(
    url=cluster.identities[0].oidcs[0].issuer,
    client_id_lists=["sts.amazonaws.com"],
    thumbprint_lists=["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"],
)
```

Without this:
```
Pod presents token to STS
Step 2: Is this OIDC URL in IAM trusted providers?
Answer: NO
Result: REJECTED — IRSA completely non-functional
```

With this:
```
Pod presents token to STS
Step 2: Is this OIDC URL in IAM trusted providers?
Answer: YES
Result: Proceeds to full verification — IRSA works
```

---

## Part 4 — Solution 2: Pod Identity

---

### 4.1 What Pod Identity Is

Pod Identity was introduced by AWS in 2023. Same goal as IRSA — give each pod its own scoped AWS role — completely different mechanism.

The core idea: **a trusted agent on each node vouches for the pod.** The pod never talks to STS directly. The pod never generates or holds a JWT token. The architecture itself enforces isolation.

---

### 4.2 One-Time Setup

**Step 1 — Install the Pod Identity Agent addon**

A DaemonSet that runs on every node in the cluster. It intercepts all credential requests from pods. It has privileged access to the EKS API — AWS already trusts it.

**Step 2 — Create an IAM role with a simple trust policy**

```json
{
  "Principal": {
    "Service": "pods.eks.amazonaws.com"
  },
  "Action": ["sts:AssumeRole", "sts:TagSession"]
}
```

Notice what is NOT here compared to IRSA: no OIDC URL, no service account name, no namespace condition. The trust policy is simple because the scoping happens elsewhere.

**Step 3 — Create a Pod Identity Association in EKS**

```
Cluster:         eks-dev
Namespace:       payments-namespace
Service Account: payments-sa
IAM Role:        arn:aws:iam::123456789012:role/payments-role
```

This mapping lives in EKS, not in the IAM role. That is a key architectural difference from IRSA.

---

### 4.3 What Happens at Runtime

```
1. payments pod starts

2. Pod's AWS SDK needs credentials
   Calls the local metadata endpoint:
   http://169.254.170.23/v1/credentials

3. Pod Identity Agent intercepts this request
   It owns that IP address on every node's network
   The pod has no way to bypass it

4. Agent reads the pod's service account and namespace
   from the Kubernetes API

5. Agent calls EKS API:
   "Pod with payments-sa in payments-namespace needs credentials.
    What role should it get?"

6. EKS looks up the Pod Identity Association:
   payments-sa + payments-namespace → payments-role

7. Agent calls AWS STS using the NODE's trusted credentials:
   "Give me temporary credentials for payments-role"

8. STS issues credentials to the agent

9. Agent returns credentials to the pod via the local endpoint

10. Pod uses credentials to call DynamoDB
    Pod never talked to STS
    Pod never held a JWT token
    Everything went through the agent
```

**Why the pod cannot bypass the agent:**

```
169.254.170.23 is a link-local address
Properties:
- Only reachable from within the same node
- Pod Identity Agent owns and controls this address
- Pod has no other route to get AWS credentials
- Network architecture makes bypass impossible
```

---

## Part 5 — Comparison

---

### 5.1 IRSA vs Pod Identity Side by Side

| Property | IRSA | Pod Identity |
|----------|------|--------------|
| Introduced | 2019 | 2023 |
| How identity is proven | Pod presents signed JWT to STS | Agent vouches for pod to STS |
| Pod talks to STS directly | Yes | No |
| OIDC provider required | Yes | No |
| Trust policy | Complex — OIDC URL + namespace + SA | Simple — pods.eks.amazonaws.com |
| Where pod-to-role mapping lives | IAM role trust policy | EKS Pod Identity Association |
| Addon required | No | Yes — Pod Identity Agent |
| AWS SDK versions | All versions | Newer versions only |
| Cross-account | Supported | Supported |

---

### 5.2 Node Role vs IRSA vs Pod Identity

```
Node Role
Every pod → same permissions (node's role)
Blast radius: entire AWS account
No per-pod scoping

IRSA
Each pod → own JWT → own IAM role → own scoped permissions
Pod proves its OWN identity via cryptographic token
OIDC provider required, complex trust policy
Blast radius: what that one role permits

Pod Identity
Each pod → agent vouches → own IAM role → own scoped permissions
Agent proves identity ON BEHALF of pod via architecture
Simpler setup, no OIDC needed
Blast radius: what that one role permits
```

All three end at the same place — a pod with temporary AWS credentials. The difference is path, security model, and blast radius.

---

### 5.3 The Analogy That Makes It Permanent

Imagine you work at **XYZ** and need to enter a partner company's office — **AWS**.

**Node Role — The Master Keycard:**
```
AWS gives XYZ ONE master keycard
Every employee uses the same keycard
One employee loses it → everyone's access is gone
Security has no idea who entered — just "someone from XYZ"
Every employee accesses every room — finance, servers, everything
```

**IRSA — The Employee ID Card System:**
```
XYZ installs an ID card system
Each employee has their OWN card, issued and SIGNED by XYZ's ID office
When you (XYZs employee) arrive at AWS:
1. You show your XYZ ID card
2. AWS security scans it
3. AWS calls XYZ's ID office: "Is this card real?"
4. ID office confirms: "Yes, that's XYZs employee, developer, card is valid"
5. AWS gives XYZs employee a visitor pass — payments room only
6. Pass expires in 1 hour

Mapping:
XYZ's ID card office   = OIDC provider (issues and signs tokens)
Your ID card                 = JWT token (Kubernetes generates for each pod)
AWS trusting XYZ's system = IAM OIDC Provider registration (you create this)
Visitor pass — payments only = temporary credentials scoped to payments-role
```

**Why registering the OIDC provider matters:**
```
Before you register:
AWS security: "We don't know XYZ's ID system" → REJECTED

After you register (create IAM OIDC Provider):
AWS security: "We recognise XYZ's ID system" → proceeds to verify
```

**Pod Identity — The Concierge System:**
```
XYZ sends a CONCIERGE with you
The concierge already has permanent trusted access to AWS
When you arrive:
1. Concierge walks you to security
2. Security recognises the concierge immediately
3. Concierge tells security: "This is XYZs employee, she needs the payments room only"
4. Security trusts the concierge's word
5. Security gives XYZs employee a visitor pass
6. You never showed your own ID card at all

Mapping:
Concierge                   = Pod Identity Agent (trusted, runs on every node)
"Security trusts concierge" = IAM trust policy: trust pods.eks.amazonaws.com
"Concierge tells security"  = Agent calls EKS API to look up Pod Identity Association
"You never showed ID"       = Pod never generates JWT, never calls STS directly
```

**Both systems, same result:**
```
                IRSA                    Pod Identity
                (ID Card System)        (Concierge System)

Who proves?     You prove yourself       Concierge proves for you
How?            Show signed ID card      Concierge vouches directly
AWS trusts?     Your ID card system      The concierge
Register first? YES — OIDC provider      NO — concierge already trusted
Result?         Payments room only       Payments room only — same
```

Neither system gives master keycard access. One compromise of XYZs employee means only the payments room is affected. Finance room, server room — untouched.

---

## The One Paragraph That Makes It All Click

OIDC, IRSA, and Pod Identity all solve the same problem: how does a pod prove to AWS who it is, without hardcoding credentials?

IRSA solves it with cryptography — Kubernetes signs a JWT token with the cluster's private key, the pod presents it to STS, STS verifies the signature using the public key fetched from the registered OIDC provider. The signature cannot be forged. The claims cannot be tampered with. STS trusts the identity because the math proves it.

Pod Identity solves it with architecture — a privileged agent runs on every node and owns the local credential endpoint. Pods cannot bypass it. The agent uses the node's trusted identity to fetch credentials on the pod's behalf. No token exchange. No OIDC verification. The architecture enforces isolation.

Both ensure a payments pod gets only DynamoDB access, a frontend pod gets only S3 read access, and a compromised pod cannot escalate beyond its assigned role. The blast radius of any single compromise is contained to exactly what that pod was supposed to do. That containment — not the mechanism that achieves it — is the point.

---
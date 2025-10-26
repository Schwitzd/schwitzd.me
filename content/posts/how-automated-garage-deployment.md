+++
title = 'How I Automated Garage Deployment'
date = 2025-10-26T08:40:48Z
draft = false
+++

After MinIO's decision to [remove all the features](https://github.com/minio/object-browser/pull/3509) from the community edition, I switched to [Garage](https://garagehq.deuxfleurs.fr). After the initial deployment test, I quickly realized that many manual steps were required before it could be used.  For example, you must [create a layout](https://garagehq.deuxfleurs.fr/documentation/quick-start/#creating-a-cluster-layout) for storing the buckets and then create the buckets themselves.

As usual I decided to come up with my overcomplicated solution:

1. Use a Kubernetes job to bootstrap the layout.
1. Create an Admin Garage API token with a K3s job.
1. Use my [Terraform Garage provider](https://search.opentofu.org/provider/schwitzd/garage/latest) to deploy buckets.

The idea behind my home cluster is that everything should be automated as code and deployed with [Argo CD](https://argoproj.github.io/cd/). As I mentioned, I created a bootstrap process to prepare Garage to create buckets.

## Bootstrap jobs

### RBAC

To be able to run my jobs I first need to create a [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/):

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-garage-bootstrap
  namespace: storage

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rbac-garage-bootstrap
  namespace: storage
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rb-garage-bootstrap
  namespace: storage
subjects:
  - kind: ServiceAccount
    name: sa-garage-bootstrap
    namespace: storage
roleRef:
  kind: Role
  name: rbac-garage-bootstrap
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cr-garage-bootstrap-secrets
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: crb-garage-bootstrap-secrets
subjects:
  - kind: ServiceAccount
    name: sa-garage-bootstrap
    namespace: storage
roleRef:
  kind: ClusterRole
  name: cr-garage-bootstrap-secrets
  apiGroup: rbac.authorization.k8s.io
```

This RBAC configuration defines many things:

- A dedicated ServiceAccount (`sa-garage-bootstrap`) used by Kubernetes jobs that initialize the Garage instance.
- The Role (`rbac-garage-bootstrap`) allows the service account to get pods and execute commands inside them. Necessary for running garage CLI commands inside existing pod.
- The RoleBinding (`rb-garage-bootstrap`) attaches that role to the service account.
- The ClusterRole (`cr-garage-bootstrap-secrets`) grants permission to get and create Kubernetes Secrets across the cluster, so the bootstrap job can store the generated Garage API token as a Kubernetes secret.
- The ClusterRoleBinding (`crb-garage-bootstrap-secrets`) links this cluster-wide secret access to the same service account.

In short, this RBAC setup gives the bootstrap job just enough privileges to run commands inside Garage pods and manage secrets, without granting unnecessary cluster-wide access.

### Layout job

After the deployment of Garage using the official Helm chart, you have to create a layout before you can create buckets. From the documentation this can be done using Garage CLI, I automated this step by creating the following K3s job:

```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: job-garage-layout-bootstrap
  namespace: storage
  labels:
    app.kubernetes.io/name: garage
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 300
  template:
    metadata:
      labels:
        app.kubernetes.io/name: garage
    spec:
      restartPolicy: Never
      serviceAccountName: sa-garage-bootstrap
      containers:
        - name: run-kubectl
          image: bitnami/kubectl:latest
          env:
            - name: GARAGE_NS
              value: storage
            - name: GARAGE_ZONE
              value: farm
            - name: GARAGE_CAPACITY
              value: 10G
          command: ["/bin/sh", "-c"]
          args:
            - |
              set -e

              echo "[+] Using Garage namespace: $GARAGE_NS"
              echo "[+] Waiting for garage-0 to be Ready..."
              while [ "$(kubectl -n $GARAGE_NS get pod garage-0 -o jsonpath='{.status.containerStatuses[0].ready}')" != "true" ]; do
                sleep 2
              done

              echo "[+] Checking if layout needs to be bootstrapped..."
              NEEDS_BOOTSTRAP=$(kubectl -n $GARAGE_NS exec -c garage garage-0 -- /garage status | grep "NO ROLE ASSIGNED" || true)

              if [ -z "$NEEDS_BOOTSTRAP" ]; then
                echo "[✓] Layout already applied. Skipping bootstrap."
                exit 0
              fi

              echo "[+] Fetching Garage node ID..."
              NODE_ID=$(kubectl -n $GARAGE_NS exec -c garage garage-0 -- /garage status | awk '/HEALTHY NODES/ {getline; getline; print $1}')
              echo "[+] Found node ID: $NODE_ID"

              echo "[+] Assigning layout (zone: $GARAGE_ZONE, capacity: $GARAGE_CAPACITY)..."
              kubectl -n $GARAGE_NS exec -c garage garage-0 -- \
                /garage layout assign -z "$GARAGE_ZONE" -c "$GARAGE_CAPACITY" "$NODE_ID"

              echo "[+] Applying layout..."
              kubectl -n $GARAGE_NS exec -c garage garage-0 -- \
                /garage layout apply --version 1

              echo "[✓] Layout applied successfully"
```

It uses three environment variables to make the process flexible:

- `GARAGE_NS` — the namespace where Garage runs.
- `GARAGE_ZONE` — the logical zone name assigned to the node (in my case, `farm`).
- `GARAGE_CAPACITY` — the storage capacity to allocate to that node (e.g. `10G`).

The job waits for the `garage-0` pod to be ready, checks whether a layout already exists, and if not, assigns the node to the given zone with the specified capacity before applying the layout.

### Admin API job

Once the layout is created, we need an admin API token to use it with my Terraform provider to create the buckets. For this task, I also created a K3s job.

```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: job-garage-admin-token-bootstrap
  namespace: storage
  labels:
    app.kubernetes.io/name: garage
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 300
  template:
    metadata:
      labels:
        app.kubernetes.io/name: garage
    spec:
      serviceAccountName: sa-garage-bootstrap
      restartPolicy: Never
      containers:
        - name: admin-token-gen
          image: bitnami/kubectl:latest
          env:
            - name: GARAGE_NS
              value: storage
            - name: GARAGE_TOKEN_NAME
              value: admin
            - name: GARAGE_ADMIN_SCOPE
              value: GetClusterStatus,ListBuckets,GetBucketInfo,ListKeys,GetKeyInfo,CreateBucket,CreateKey,AllowBucketKey
            - name: K3S_SECRET_NAME
              value: auth-garage-admin-api
          command: ["/bin/sh", "-c"]
          args:
            - |
              set -e

              if [ -z "$K3S_SECRET_NAME" ] || [ -z "$GARAGE_TOKEN_NAME" ] || [ -z "$GARAGE_NS" ] || [ -z "$GARAGE_ADMIN_SCOPE" ]; then
                echo "ERROR: K3S_SECRET_NAME, GARAGE_TOKEN_NAME, GARAGE_NS, and GARAGE_ADMIN_SCOPE env variables must be set!" >&2
                exit 1
              fi

              echo "[+] Checking if secret $K3S_SECRET_NAME already exists in $GARAGE_NS..."
              if kubectl -n $GARAGE_NS get secret $K3S_SECRET_NAME >/dev/null 2>&1; then
                echo "[✓] Secret $K3S_SECRET_NAME already exists. Skipping token bootstrap."
                exit 0
              fi

              echo "[+] Waiting for garage-0 to be Ready in $GARAGE_NS..."
              while [ "$(kubectl -n $GARAGE_NS get pod garage-0 -o jsonpath='{.status.containerStatuses[0].ready}')" != "true" ]; do
                sleep 2
              done

              echo "[+] Creating Garage admin token ($GARAGE_TOKEN_NAME) with restricted scope..."
              ADMIN_TOKEN=$(kubectl -n $GARAGE_NS exec garage-0 -- /garage admin-token create -q --scope "$GARAGE_ADMIN_SCOPE" "$GARAGE_TOKEN_NAME")

              if [ -z "$ADMIN_TOKEN" ]; then
                echo "ERROR: Failed to generate admin token." >&2
                exit 1
              fi

              echo "[+] Creating Kubernetes secret: $K3S_SECRET_NAME"
              kubectl -n $GARAGE_NS create secret generic $K3S_SECRET_NAME \
                --from-literal=token="$ADMIN_TOKEN"

              echo "[✓] Admin token successfully created and stored in secret: $K3S_SECRET_NAME"
```

It takes four environment variables as input:

- `GARAGE_NS` — the namespace where Garage runs.
- `GARAGE_TOKEN_NAME` — the name assigned to the generated token.
- `GARAGE_ADMIN_SCOPE` — the list of allowed operations for the token.
- `K3S_SECRET_NAME` — the name of the secret that will store the token.

The job waits for the `garage-0` pod to be ready, checks if the secret already exists, and if not, creates a new admin token inside the Garage pod. The token is then stored in the specified Kubernetes secret.

## Provision Buckets

Now that the cluster layout and admin API token have been created, it is time to deploy the buckets. Initially, I created a series of jobs for each bucket, but that was not a sustainable solution. After searching online, I found multiple [Terraform providers](https://developer.hashicorp.com/terraform/language/providers), but they all seem to have been abandoned and for the 1.x branch. So, I had an idea. Why don't I create my [own provider](https://search.opentofu.org/provider/schwitzd/garage/latest)? After doing some research, I found that [Go](https://go.dev/) seems to be the most appropriate development language. However, I don't know anything about it. I decided to try the hype of [vibe coding](https://en.wikipedia.org/wiki/Vibe_coding) and use the official [VSCode extension](https://marketplace.visualstudio.com/items?itemName=OpenAI.chatgpt) with [OpenAI Codex](https://openai.com/codex/).

Below is an example of how to use my provider to deploy a NextCloud bucket. I organized the Terraform code into two files, `provider.tf`:

```tf
terraform {
  required_providers {
    garage = {
      source  = "schwitzd/garage"
    }
  }
}

provider "garage" {
  host   = "<garage-admin-endpoint>:3903"
  scheme = "https"
  token  = "<admin-secret-token>"
}

```

The job previously generated an Admin API secret that can be retrieved with the following command:

```sh
kubectl -n storage get secret auth-garage-admin-api -o jsonpath='{.data.token}' | base64 -d
```

And the `main.tf` file:

```tf
resource "garage_key" "nextcloud" {
  name       = "nextcloud-key"

  permissions {
    read  = true
    write = false
    admin = false
  }
}

# Create a bucket using the key
resource "garage_bucket" "nextcloud" {
  global_alias = "nextcloud-bucket"
}

resource "garage_bucket_key" "binding" {
  bucket_id     = garage_bucket.nextcloud.id
  access_key_id = garage_key.nextcloud.access_key_id
}
```

## Closing words

I spent a lot of time designing and implementing this flow for only two or three buckets that I will use to back up some workloads on my home cluster. Instead, I could have finished in three commands on the CLI. However, it was a valuable opportunity to develop my own Terraform provider with Codex. Honestly, I have absolutely no knowledge of Go, so I cannot evaluate whether the code written by the LLM is safe and scalable for the future. For this reason, I strongly discourage you from using it in production or anywhere losing data would be problematic.

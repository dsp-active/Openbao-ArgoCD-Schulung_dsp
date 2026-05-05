# Manuelles Setup — OpenBao + ArgoCD + VSO auf kind

Schritt-für-Schritt-Anleitung. Jeden Block einzeln ausführen und das Ergebnis verifizieren, **bevor** du zum nächsten Schritt gehst.

> **Tipp:** Lass dir alle Befehle in **eine** Shell-Session laufen, damit Variablen wie `$ROOT_TOKEN` erhalten bleiben.

---

## Schritt 0: Vorbereitung

```bash
# Container-Runtime prüfen — eines davon muss laufen
docker ps || podman ps

# Falls Podman: kind muss das wissen
export KIND_EXPERIMENTAL_PROVIDER=podman   # nur bei Podman nötig

# Tools prüfen
kind --version
kubectl version --client
helm version --short
jq --version

# Arbeitsordner anlegen
mkdir -p ~/openbao-argocd-schulung
cd ~/openbao-argocd-schulung
```

---

## Schritt 1: kind-Cluster anlegen

```bash
cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: openbao-argocd
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
      - containerPort: 30200
        hostPort: 30200
        protocol: TCP
  - role: worker
EOF

kind create cluster --config kind-config.yaml
```

**Verifizieren:**

```bash
kubectl get nodes
# 2 Nodes, beide Ready
```

---

## Schritt 2: ArgoCD installieren

```bash
kubectl create namespace argocd

kubectl create -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Warten — kann 1–2 Min dauern
kubectl -n argocd rollout status deployment/argocd-server --timeout=300s
```

**UI auf NodePort exponieren:**

```bash
kubectl -n argocd patch svc argocd-server -p '{
  "spec": {"type": "NodePort", "ports": [{"name":"http","port":80,"targetPort":8080,"nodePort":30080}]}
}'
```

**Initial-Passwort holen:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

**Verifizieren:**

```bash
kubectl -n argocd get pods
# Alle Running

# Browser: http://localhost:30080
# Login: admin / <Passwort von oben>
```

---

## Schritt 3: OpenBao installieren

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update

kubectl create namespace openbao

cat > openbao-values.yaml <<'EOF'
server:
  standalone:
    enabled: true
    config: |
      ui = true
      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
      }
      storage "file" {
        path = "/openbao/data"
      }
  service:
    type: NodePort
    nodePort: 30200
ui:
  enabled: true
EOF

helm install openbao openbao/openbao -n openbao -f openbao-values.yaml
```

**Auf Pod warten:**

```bash
# Pod muss erstmal existieren
kubectl -n openbao wait --for=jsonpath='{.status.phase}'=Running pod/openbao-0 --timeout=180s
```

**Verifizieren:**

```bash
kubectl -n openbao get pods
# openbao-0   0/1   Running
# 0/1 ist normal, weil noch sealed
```

---

## Schritt 4: OpenBao initialisieren und unsealen

```bash
# Initialisieren — Output sicher speichern!
kubectl -n openbao exec openbao-0 -- \
  bao operator init -key-shares=1 -key-threshold=1 -format=json > openbao-init.json

# Variablen setzen (in DIESER Shell merken!)
export UNSEAL_KEY=$(jq -r '.unseal_keys_b64[0]' openbao-init.json)
export ROOT_TOKEN=$(jq -r '.root_token' openbao-init.json)

echo "Unseal Key: $UNSEAL_KEY"
echo "Root Token: $ROOT_TOKEN"
```

> ⚠️ **WICHTIG:** Notiere dir den Root Token jetzt — du brauchst ihn für die OpenBao-UI!

**Unsealen:**

```bash
kubectl -n openbao exec openbao-0 -- bao operator unseal "$UNSEAL_KEY"
```

**Verifizieren:**

```bash
kubectl -n openbao exec openbao-0 -- bao status
# Sealed: false  ← muss false sein!
# Initialized: true
```

```bash
kubectl -n openbao get pods
# openbao-0   1/1   Running   ← jetzt 1/1 Ready
```

> ⚠️ **Wenn der Pod neu startet, ist OpenBao wieder sealed!** Dann nochmal:
> ```bash
> kubectl -n openbao exec openbao-0 -- bao operator unseal "$UNSEAL_KEY"
> ```

---

## Schritt 5: KV v2 Engine aktivieren

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao secrets enable -version=2 -path=secret kv
```

**Verifizieren:**

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao secrets list
# secret/   kv          # ← muss da stehen
```

---

## Schritt 6: Schulungs-Policy schreiben

```bash
# Hier kann es vorkommen, dass das -i switch weggelassen werden muss.
cat <<'EOF' | kubectl -n openbao exec -i openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" bao policy write schulung-user -
path "secret/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list", "delete"]
}
EOF
```

**Verifizieren:**

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao policy read schulung-user
# Zeigt den Inhalt der Policy
```

---

## Schritt 7: Kubernetes-Auth aktivieren

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao auth enable kubernetes
```

**Verifizieren:**

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao auth list
# kubernetes/   ...
```

---

**Auth-Role anlegen:**

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao write auth/kubernetes/role/schulung \
  bound_service_account_names="*" \
  bound_service_account_namespaces="*" \
  policies=schulung-user \
  ttl=1h
```

**Verifizieren:**

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN="$ROOT_TOKEN" \
  bao read auth/kubernetes/role/schulung
# Zeigt die Konfiguration der Role
```

---

## Schritt 8: Vault Secrets Operator installieren

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  --create-namespace -n vault-secrets-operator-system \
  --version 0.10.0
```

**Warten und verifizieren:**

```bash
kubectl -n vault-secrets-operator-system rollout status \
  deployment vault-secrets-operator-controller-manager --timeout=180s

kubectl -n vault-secrets-operator-system get pods
# vault-secrets-operator-controller-manager-...   2/2   Running
```

---

## Schritt 9: VaultConnection + VaultAuth anlegen

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: default
  namespace: vault-secrets-operator-system
spec:
  address: http://openbao.openbao.svc.cluster.local:8200
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: default
  namespace: vault-secrets-operator-system
spec:
  vaultConnectionRef: default
  method: kubernetes
  mount: kubernetes
  allowedNamespaces:
    - "*"
  kubernetes:
    role: schulung
    serviceAccount: default
EOF
```

**Verifizieren:**

```bash
kubectl get vaultconnection -A
kubectl get vaultauth -A
# Beide sollten "default" zeigen, im Namespace vault-secrets-operator-system
```

## Final-Check: Läuft alles?

```bash
echo "=== Cluster ==="
kubectl get nodes

echo ""
echo "=== ArgoCD ==="
kubectl -n argocd get pods | grep -v Completed

echo ""
echo "=== OpenBao ==="
kubectl -n openbao get pods
kubectl -n openbao exec openbao-0 -- bao status | grep -E "Sealed|Initialized"

echo ""
echo "=== VSO ==="
kubectl -n vault-secrets-operator-system get pods

echo ""
echo "=== VSO CRs ==="
kubectl get vaultconnection -A
kubectl get vaultauth -A


echo ""
echo "=== Zugangsdaten ==="
echo "ArgoCD UI:  http://localhost:30080"
echo "  admin / $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"
echo ""
echo "OpenBao UI: http://localhost:30200"
echo "  Token: $(jq -r '.root_token' ~/openbao-argocd-schulung/openbao-init.json)"
```



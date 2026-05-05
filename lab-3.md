# Lab 03: ApplicationSet verstehen — vom Ordner zur Application

**Dauer:** ca. 20 Minuten
**Format:** Theorie + Code-Lesen, kein Schreiben

## Ziel

Du verstehst, wie das im Setup vorhandene `ApplicationSet` funktioniert: warum aus einem Ordner im Git-Repo automatisch eine ArgoCD-Application wird, in welchen Namespace deployed wird und welche Konventionen du einhalten musst.

---

## 1. Warum ApplicationSet?

Wenn dein Cluster nur 3 Apps hat, kannst du jede `Application` per Hand schreiben. Bei 30 oder 300 Apps wird das zu Copy-Paste-Hölle. Ein **ApplicationSet** löst das, indem es eine *Vorlage* für Applications definiert und sie aus einem **Generator** (Ordner-Liste, Listen-Werte, Cluster-Liste, …) automatisch erzeugt.

Wir nutzen den **Git Files/Directories Generator**: Pro Ordner im Git-Repo eine Application.

---

## 2. Das ApplicationSet im Schulungs-Setup

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo-applicationset
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - git:
      repoURL: https://git.<firma>/<dein-user>/openbao-argocd-schulung.git
      revision: main
      directories:
      - path: applications/*
  template:
    metadata:
      finalizers:
        - "resources-finalizer.argocd.argoproj.io"
      name: '{{.path.basenameNormalized}}'
    spec:
      project: "default"
      sources:
      - repoURL: https://git.<firma>/<dein-user>/openbao-argocd-schulung.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        namespace: '{{.path.basenameNormalized}}'
        name: "in-cluster"
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
```

### Schauen wir uns das Stück für Stück an:

#### Generator

```yaml
generators:
- git:
    repoURL: https://git.<firma>/<dein-user>/openbao-argocd-schulung.git
    revision: main
    directories:
    - path: applications/*
```

Das sagt: „Schau im Git-Repo unter `applications/` nach. **Jeder direkte Unterordner** wird zu einem Datensatz, den wir gleich ins Template einsetzen."

Der Generator stellt unter anderem diese Variablen bereit:

- `{{.path.path}}` — der vollständige Pfad, z.B. `applications/demo-app`
- `{{.path.basename}}` — nur der Ordnername, z.B. `demo-app`
- `{{.path.basenameNormalized}}` — wie `basename`, aber DNS-konform (Underscores → Hyphens etc.)

#### Template — Name der Application

```yaml
metadata:
  finalizers:
    - "resources-finalizer.argocd.argoproj.io"
  name: '{{.path.basenameNormalized}}'
```

Heißt der Ordner `demo-app`, heißt die Application `demo-app`. Der Finalizer sorgt dafür, dass beim Löschen der Application auch alle deployten Resources sauber entfernt werden.

#### Template — wo holt sie die Manifeste her?

```yaml
sources:
- repoURL: https://git.<firma>/<dein-user>/openbao-argocd-schulung.git
  targetRevision: HEAD
  path: '{{.path.path}}'
```

`targetRevision: HEAD` heißt: der jeweils neueste Stand des Default-Branches. ArgoCD wendet alle YAML-Dateien aus dem Ordner an (rekursiv).

#### Template — wohin wird deployed?

```yaml
destination:
  namespace: '{{.path.basenameNormalized}}'
  name: "in-cluster"
```

Der Ziel-Namespace ist **gleich dem Ordnernamen** (also gleich dem App-Namen). `name: "in-cluster"` ist der Default-Cluster, in dem ArgoCD selbst läuft.

> ⚠️ Das ist die Konvention, an die du dich halten musst:
> **Ordner-Name = App-Name = Namespace-Name**

#### syncOptions

```yaml
syncOptions:
  - CreateNamespace=true
  - PruneLast=true
```

- `CreateNamespace=true` — der Namespace wird automatisch angelegt, wenn er noch nicht existiert
- `PruneLast=true` — beim Löschen werden Resources erst nach allem anderen gelöscht (verhindert Race-Conditions)

---

## 3. Was bedeutet das konkret für deinen Workflow?

Wenn du eine neue App `my-app` aufsetzen willst, machst du genau das:

```
applications/
└── my-app/                    ← AppSet sieht: neuer Ordner → neue Application "my-app"
    └── my-app/                ← Ordner mit Namespace-Name (Konvention!)
        ├── deployment.yaml
        ├── service.yaml
        └── vaultstaticsecret.yaml
```

Pushst du das, läuft folgendes ab:

1. **ArgoCD ApplicationSet-Controller** pollt das Repo (Default: alle 3 Minuten, oder per Webhook sofort).
2. Er sieht den neuen Ordner `applications/my-app/` → erzeugt `Application my-app`.
3. Diese Application sagt: „Deploy alles aus `applications/my-app/` (rekursiv) in den Namespace `my-app`."
4. ArgoCD legt den Namespace `my-app` an (`CreateNamespace=true`) und wendet alle YAMLs an.
5. Eines der YAMLs ist die `VaultStaticSecret` — der **VSO** sieht sie und beginnt mit dem Secret-Sync.

> 💡 Die zweite Verschachtelung (`my-app/my-app/...`) wirkt erst seltsam, hat aber ihren Grund: Der äußere Ordner ist die Identität der **App**, der innere Ordner ist der **Namespace**. Beide sind in unserem Setup gleich, aber theoretisch könntest du das ApplicationSet so umbauen, dass eine App in mehrere Namespaces deployed wird.

---

## 4. Live-Demo: existierende Application anschauen

Wenn dein Setup frisch ist, ist das Repo leer und es gibt noch keine generierten Applications. Du siehst dann nur das `apps` ApplicationSet selbst, ohne Children. Sobald du in Lab 04 deinen ersten Ordner anlegst, taucht hier eine Application auf.

```bash
kubectl -n argocd get applications
kubectl -n argocd get applicationset demo-applicationset -o yaml
```

🔧 **Aufgabe:** In der ArgoCD-UI

1. Klicke auf das `demo-applicationset` ApplicationSet.
2. Schau dir an, welche Applications generiert wurden (anfangs: keine).
3. Sobald wir in Lab 04 eine erste App anlegen, kommst du wieder hierher und schaust den **Manifest**- und **Resources**-Tab der erzeugten Application an.

---

## 5. Das ApplicationSet manuell triggern

Manchmal pollt ArgoCD nicht schnell genug. Du kannst einen Refresh erzwingen:

```bash
# In der UI: Application öffnen → "Refresh" oben rechts

# Oder per CLI
kubectl -n argocd annotate applicationset demo-applicationset \
  argocd.argoproj.io/refresh=hard --overwrite
```

---

## 🔧 Lab-Check

Fragen, die du beantworten können solltest:

1. Wenn du im Repo einen Ordner `applications/billing-api/` anlegst — wie heißt die erzeugte Application und in welchen Namespace deployed sie?
2. Was bewirkt `{{.path.basenameNormalized}}` im Vergleich zu `{{.path.basename}}`?
3. Wo legst du die `VaultStaticSecret`-Resource ab — direkt unter `applications/<app>/` oder unter `applications/<app>/<namespace>/`?
4. Was passiert, wenn du im Repo nur den Ordner `applications/empty-app/` anlegst, aber keine YAMLs darin?

> Antworten:
> 1. App heißt `billing-api`, Namespace heißt `billing-api`.
> 2. `basenameNormalized` macht den Wert DNS-konform (kleinbuchstaben, Underscores werden zu Hyphens). Wichtig, weil K8s-Namespaces & Application-Names DNS-konform sein müssen.
> 3. Unter `applications/<app>/<namespace>/` — zusammen mit allen anderen Manifesten.
> 4. ArgoCD legt eine leere Application an. Status `Synced` (es gibt nichts zu syncen), `Healthy` (nichts ungesund). In der UI sieht das aus wie eine leere Hülle.

---

➡️ Weiter mit **Lab 04: Erste eigene Application — Ordner anlegen und Secret syncen**
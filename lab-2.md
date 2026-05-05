# Lab 02: OpenBao kennenlernen — KV-v2-Secrets in der Web-UI anlegen

**Dauer:** ca. 20 Minuten
**Format:** Hands-on (Web-UI)

## Ziel

Du legst in OpenBao über die Web-UI dein erstes Secret an. Du verstehst, wie KV v2 funktioniert, wie Secrets versioniert werden und welche Pfad-Konvention wir in der Schulung verwenden.

---

## 1. Was ist KV v2?

KV v2 (Key-Value, Version 2) ist die Standard-Secret-Engine in OpenBao für statische Secrets. Sie speichert Schlüssel-Wert-Paare und behält **alte Versionen** eines Secrets, wenn du es überschreibst — das hilft beim Rollback.

Beispiel-Secret:

```
Pfad:  secret/demo-app/database
Daten: {
  "username": "appuser",
  "password": "s3cr3t!"
}
```

### Pfad-Schreibweise — wichtig zu wissen

KV v2 hat eine kleine Eigenheit: in der **UI** siehst du den Pfad als `secret/demo-app/database`, aber bei der **API** (und beim VSO) sieht der Pfad intern so aus: `secret/data/demo-app/database`. Das `data/` wird automatisch eingefügt.

Für dich heißt das:

- **In der UI:** Pfad ohne `data/` eingeben.
- **In der `VaultStaticSecret`-Resource (Lab 04):** im Feld `path:` schreibst du **auch ohne `data/`** — der VSO ergänzt es selbst, weil er weiß, dass es KV v2 ist.

> ⚠️ Wenn dein Sync später nicht funktioniert mit „permission denied" oder „path not found", ist es zu 80% ein Pfad-Problem. Wir gehen das in Lab 04 noch einmal ganz konkret durch.

---

## 2. In der OpenBao-UI anmelden

Öffne die OpenBao-UI im Browser (URL bekommst du vom Trainer).

Auswahl der Login-Methode: **Token** → Token vom Trainer einfügen → **Sign In**.

> 💡 In Produktion ist Token-Login natürlich tabu — dort nutzt man OIDC/SSO. Für die Schulung okay.

---

## 3. KV-Engine finden

Nach dem Login siehst du das Dashboard. In der linken Navigation:

**Secrets Engines** → klicke auf den Mount **`secret/`**.

Du landest auf einer Übersicht der bereits angelegten Pfade. Die meisten Pfade darunter gehören anderen Teilnehmern oder zur Plattform — geh dort nicht rein.

---

## 4. Pfad-Konvention für die Schulung

Damit wir uns nicht in die Quere kommen, halten wir uns an dieses Schema:

```
secret/<dein-username>/<app-name>/<purpose>
```

**Beispiele** (Username `anna`):

```
secret/anna/demo-app/database
secret/anna/billing/api-key
```

> ⚠️ Bitte **bleib in deinem Pfad** `secret/<dein-username>/...`. Deine Policy erlaubt dir nur dort zu schreiben.

---

## 5. Erstes Secret anlegen

Klicke oben rechts in der `secret/`-Übersicht auf **+ Create secret**.

### 5.1 Pfad eingeben

Im Feld **Path for this secret:**

```
<dein-username>/demo-app/database
```

> 💡 Du musst nur den Teil **nach** `secret/` eingeben — die UI hat den Mount-Namen bereits oben angezeigt.

### 5.2 Secret-Daten eingeben

Du hast zwei Modi: **Form (Default)** und **JSON**. Wir nutzen die Form-Ansicht für den ersten Versuch.

Trage als Key-Value-Paare ein (jedes „Add" fügt eine neue Zeile hinzu):

| Key | Value |
|-----|-------|
| `username` | `appuser` |
| `password` | `demo-pass-123` |
| `host` | `postgres.demo-app.svc.cluster.local` |
| `port` | `5432` |

Klicke **Save**.

### 5.3 Secret prüfen

Du landest auf der Detail-Ansicht des Secrets. Du solltest sehen:

- Oben den vollen Pfad: `secret/<dein-username>/demo-app/database`
- Den Tab **Secret** mit deinen vier Feldern
- Den Tab **Metadata** mit Erstelldatum und Version (sollte Version 1 zeigen)
- Den Tab **Version History** — aktuell mit nur einer Version

> 💡 Werte sind in der UI standardmäßig versteckt (Sternchen). Klicke auf das Auge-Symbol, um sie sichtbar zu machen — bzw. auf **Show JSON** für die komplette Ausgabe.

---

## 6. Eine zweite Version erzeugen

KV v2 erlaubt dir, ein Secret zu überschreiben — die alte Version bleibt aber zugänglich.

In der Detail-Ansicht des Secrets:

1. Klicke oben rechts auf **Create new version**.
2. Du siehst die aktuellen Werte vorausgefüllt.
3. Ändere `password` auf `demo-pass-NEW` (nur zum Testen — wir machen das später wieder rückgängig).
4. Klicke **Save**.

Jetzt:

- Im Tab **Version History** siehst du **Version 1** und **Version 2**.
- Klick auf Version 1 zeigt dir den **alten** Wert.
- Standardmäßig wird Version 2 als „aktuelle" verwendet.

> 💡 KV v2 speichert standardmäßig die letzten 10 Versionen. Das kannst du pro Mount konfigurieren.

---

## 7. Wieder auf den Original-Wert zurück

Damit Lab 04 mit den richtigen Werten startet, setze das Secret auf `demo-pass-123` zurück:

1. **Create new version**
2. `password` zurück auf `demo-pass-123`
3. **Save**

Du hast jetzt drei Versionen, aber die aktuell aktive ist wieder die ursprüngliche.

> 💡 Alternative: Im Tab **Version History** auf Version 1 klicken → **Promote this version** (falls die Policy es erlaubt). Aber „Create new version" funktioniert immer.

---

## 8. Was darf dein Token überhaupt?

Falls du dich fragst, warum du auf `secret/<dein-username>/...` schreiben darfst, aber nicht auf `secret/admin/...`: das steuert eine **Policy**.

In der OpenBao-UI:

**Top-Right (Avatar)** → **Profile** (oder ähnlich) zeigt dir die Policies, die an deinem Token hängen.

Beispiel: `default` und `schulung-user`.

> 💡 In Produktion bekommt jede Application **ihren eigenen Pfad und ihre eigene Policy** — und keine App darf in fremde Pfade lesen. Das passiert automatisch über die `VaultAuth`, die Pods anhand ihres ServiceAccounts authentifiziert. Den Mechanismus nutzen wir gleich passiv mit, ohne ihn selbst aufzusetzen.

---

## 🔧 Lab-Check

- [ ] Du hast in der UI ein Secret unter `secret/<dein-username>/demo-app/database` angelegt
- [ ] Das Secret hat die vier Felder `username`, `password`, `host`, `port`
- [ ] Du hast eine zweite (und dritte) Version erzeugt und im Versions-Reiter gesehen
- [ ] Der aktuelle `password`-Wert ist wieder `demo-pass-123` (für Lab 04 wichtig!)
- [ ] Du verstehst den Unterschied zwischen `secret/foo` (UI) und `secret/data/foo` (API-intern)

---

## 🔧 Bonus-Aufgabe (optional)

Lege noch ein zweites Secret an unter:

```
secret/<dein-username>/demo-app/api-keys
```

mit z.B. `external_api_key=abc123` und `webhook_secret=xyz789`.

> Das brauchen wir nicht zwingend für die nächsten Labs — aber es zeigt, dass eine App problemlos **mehrere** Secrets aus OpenBao beziehen kann (in Lab 04 sieht man dann, dass du dafür zwei `VaultStaticSecret`-Resourcen anlegst).


# SCREENSHOT RUNBOOK — kako uslikati svih 21 slika

Slijedi redom. Svaki blok ima: **što pokrenuti** i **što mora biti na slici (Slika N)**.
Sve `<GHCR_USER>` zamijeni svojim GitHub korisničkim imenom.

---

## PREDUVJETI (instalirati jednom)
```bash
podman --version
kubectl version --client
minikube version
trivy --version
```
Klon aplikacije + ubaci ove datoteke iz scaffolda na iste putanje u repo.
```bash
cp .env.example .env
# generiraj lockfile (treba za npm ci):
cd api && npm install && cd ..
cd frontend && npm install && cd ..
cd worker && npm install && cd ..
```

---

# FAZA 1 — Lokalni stack (Slike 1–5)

### Slika 1 — Containerfile (tri faze + non-root)
```bash
cat api/Containerfile
```
**Slikaj:** sadržaj `api/Containerfile` gdje se vide faze `deps-prod / dev / runtime` i `USER node`.

### Pokreni cijeli stack jednom naredbom
```bash
podman compose up --build -d
```

### Slika 2 — stanje servisa
```bash
podman compose ps
```
**Slikaj:** svih 5 servisa u stanju `healthy` + portovi (5432, 6379, 8080, 3000).

### Slika 3 — veličine slika
```bash
podman images | grep ticketing
```
**Slikaj:** redove s veličinama tvojih slika (male, alpine osnovica).

### Slika 4 — health/ready/events
```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:8080/events
```
**Slikaj:** terminal s tri JSON odgovora (`ok`, `ready`, popis događaja).

### Slika 5 — kupnja karte (UI + obrada)
1. Otvori `http://localhost:3000`, odaberi događaj, klikni **Purchase**.
2. U drugom prozoru:
```bash
curl http://localhost:8080/tickets/orders
```
**Slikaj:** preglednik s `orderId` u Output polju **i** `curl` izlaz gdje narudžba ima `"status":"processed"`.

> Hot-reload provjera (nije obavezna slika): promijeni tekst u `frontend/src/public/index.html`, refresh — promjena bez rebuilda.

---

# FAZA 2 — Lokalno Trivy skeniranje (priprema za Slike 9–10)
Možeš skenirati i lokalno (brže od čekanja CI-a):
```bash
trivy fs --scanners secret .
podman build -f api/Containerfile --target runtime -t ticketing-api:scan api
trivy image --severity CRITICAL,HIGH --ignore-unfixed ticketing-api:scan
```
Rezultate prepiši u `docs/security/image-scan-report.md`.

---

# FAZA 3 — CI/CD na GitHubu (Slike 6–11)

1. Stavi `.github/workflows/ci.yaml` u repo, commitaj i pushaj na `main`.
2. GitHub → **Actions** → otvori zadnji run.

### Slika 6 — zeleni pipeline
**Slikaj:** prikaz runa gdje su `lint-and-test`, `secret-scan`, `build-scan-push` zeleni.

### Slika 9 — Trivy secret scan
Otvori job `secret-scan` → korak Trivy.
**Slikaj:** izlaz koji pokazuje da nema pronađenih tajni.

### Slika 10 — Trivy image scan
Otvori job `build-scan-push` (npr. service=worker) → korak "Trivy image scan".
**Slikaj:** izlaz skeniranja slike (tablica ranjivosti / "0 CRITICAL, 0 HIGH").

### Slika 7 — objavljene slike u GHCR
GitHub profil → **Packages** → `ticketing-api/-frontend/-worker`.
**Slikaj:** popis paketa s oznakama `v1.0.0` i `<sha>`.
> Da `minikube` može povući slike: na svakom packageu **Package settings → Change visibility → Public** (najlakše), ili koristi imagePullSecret.

### Slika 8 — .env u .gitignore
```bash
cat .gitignore
git status        # .env se NE pojavljuje kao tracked
```
**Slikaj:** `.gitignore` s linijom `.env` + `git status` bez `.env`.

### Slika 11 — quality gate (padne pa prođe)
1. Privremeno "pokvari" da Trivy nađe ranjivost — npr. u `api/Containerfile` promijeni prvu liniju u staru osnovicu:
   `FROM node:18.0.0-buster AS base` (stara, ranjiva).
2. Commit + push → **Slikaj** crveni run gdje `build-scan-push` padne na CRITICAL/HIGH.
3. Vrati na `node:22-alpine`, push → run opet zelen.
**Slikaj:** crveni (pao gate) i zeleni run jedan do drugog.

---

# FAZA 4 — Kubernetes na minikube (Slike 15–21)

### Pokreni klaster + Ingress
```bash
minikube start --driver=podman
minikube addons enable ingress
```
U `k8s/07-api.yaml`, `07-frontend.yaml`, `07-worker.yaml` zamijeni `<GHCR_USER>`:
```bash
sed -i "s/<GHCR_USER>/TVOJ_GITHUB/g" k8s/07-*.yaml
```

### Slika 15 — struktura manifesta
```bash
ls -1 k8s/
```
**Slikaj:** popis svih manifesta u `k8s/` (ili prikaz `k8s/` foldera na GitHubu).

### Primijeni sve
```bash
kubectl apply -f k8s/
kubectl -n ticketing get pods -w     # čekaj da sve bude Running/Ready
```

### Slika 16 — Secret + ConfigMap
```bash
kubectl -n ticketing get configmap ticketing-config -o yaml
kubectl -n ticketing get secret ticketing-db-secret -o yaml
```
**Slikaj:** ConfigMap (čitljive vrijednosti) + Secret (base64, placeholder) — dokaz odvojene konfiguracije.

### Slika 17 — probe + secretKeyRef + resursi
```bash
kubectl -n ticketing describe deployment api | sed -n '/Containers/,/Conditions/p'
```
**Slikaj:** dio gdje se vide `Liveness /healthz`, `Readiness /readyz`, `Limits 300m/192Mi`, `Requests 100m/96Mi` i env iz Secreta.

### Slika 18 — ServiceAccount least-privilege
```bash
kubectl -n ticketing get serviceaccount ticketing-sa -o yaml
```
**Slikaj:** `automountServiceAccountToken: false`.

### Slika 19 — NetworkPolicy
```bash
kubectl -n ticketing get networkpolicy
kubectl -n ticketing describe networkpolicy db-ingress
```
**Slikaj:** popis (`default-deny`, `db-ingress`, `cache-ingress`, `allow-web-ingress`) + detalj `db-ingress` pravila.

### Vanjski pristup (Ingress)
```bash
echo "$(minikube ip) ticketing.local" | sudo tee -a /etc/hosts
curl http://ticketing.local/api/healthz
```

### Slika 21 — app kroz Ingress
Otvori `http://ticketing.local` u pregledniku i kupi kartu.
**Slikaj:** aplikacija radi preko `ticketing.local` (vidi se host u adresnoj traci).

### Slika 20 — rolling update + rollback
```bash
kubectl -n ticketing set image deployment/api api=ghcr.io/TVOJ_GITHUB/ticketing-api:<novi-sha>
kubectl -n ticketing rollout status deployment/api
kubectl -n ticketing rollout undo deployment/api
kubectl -n ticketing rollout status deployment/api
```
**Slikaj:** izlaz `rollout status` za update i nakon `rollout undo`.

---

# FAZA 5 — Incident screenshots (Slike 12–14)
Slijedi `docs/runbook.md`, sekcije "Kako reproducirati":
- **Slika 12** — Incident 1 (pad baze): `/readyz` not-ready + `get pods`.
- **Slika 13** — Incident 2 (loš tag): `ImagePullBackOff` + `rollout undo`.
- **Slika 14** — Incident 3 (krivi secret): `/readyz` auth failed + ispravak.

---

## Redoslijed snimanja (sažeto)
1→5 (lokalno) → 8,9,10 (trivy/CI) → 6,7,11 (CI/GHCR) → 15,16,17,18,19,21,20 (k8s) → 12,13,14 (incidenti).

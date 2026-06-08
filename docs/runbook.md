# Runbook — Secure Event Ticketing Platform

Format svakog incidenta: **Simptom → Dijagnoza → Ispravak → Prevencija**.
Na dnu svakog incidenta je "Kako reproducirati" — koristi to da uslikaš dokaz (Slika 12–14).

---

## Incident 1 — Pad PostgreSQL baze

**Simptom**
- `GET /tickets/orders` vraća 500
- `GET /readyz` vraća `not-ready`
- `postgres` pod nije u stanju `Running`

**Dijagnoza**
```bash
kubectl -n ticketing get pods
kubectl -n ticketing logs deployment/postgres --previous
kubectl -n ticketing describe pod -l app=postgres   # traži OOMKilled, FailedMount
kubectl -n ticketing get pvc
```

**Ispravak**
- Ako je pod pao zbog resursa: povećaj `limits.memory` u `k8s/05-postgres.yaml` pa `kubectl apply`.
- Ako je PVC problem: provjeri `kubectl -n ticketing describe pvc postgres-pvc`.
- Vrati repliku: `kubectl -n ticketing scale deploy/postgres --replicas=1`.

**Prevencija**
- Realni resource requests/limits, monitoring memorije, backup PVC-a.

**Kako reproducirati (za Sliku 12)**
```bash
kubectl -n ticketing scale deploy/postgres --replicas=0     # ugasi bazu
curl http://ticketing.local/api/readyz                      # -> not-ready  (SLIKAJ)
kubectl -n ticketing get pods                               # postgres nestao (SLIKAJ)
kubectl -n ticketing scale deploy/postgres --replicas=1     # vrati
```

---

## Incident 2 — Neispravna oznaka slike (deploy prošao, app pao)

**Simptom**
- `/healthz` vraća 502 ili timeout
- podovi u `CrashLoopBackOff` ili `ImagePullBackOff` odmah nakon deploya

**Dijagnoza**
```bash
kubectl -n ticketing get pods
kubectl -n ticketing describe deployment/api | grep -i image
kubectl -n ticketing describe pod -l app=api | tail -20     # Events: ErrImagePull
kubectl -n ticketing logs deployment/api --previous
```

**Ispravak**
```bash
kubectl -n ticketing rollout undo deployment/api
kubectl -n ticketing rollout status deployment/api
```

**Prevencija**
- Deploy isključivo po nepromjenjivom SHA tagu; provjera da slika postoji u registru prije applya.

**Kako reproducirati (za Sliku 13)**
```bash
kubectl -n ticketing set image deployment/api api=ghcr.io/<GHCR_USER>/ticketing-api:nepostojeci-tag
kubectl -n ticketing get pods            # ImagePullBackOff   (SLIKAJ)
kubectl -n ticketing rollout undo deployment/api
kubectl -n ticketing rollout status deployment/api   # vraćeno (SLIKAJ)
```

---

## Incident 3 — Neispravan Secret (kriva lozinka baze)

**Simptom**
- `/readyz` vraća grešku `password authentication failed for user`
- worker logovi ponavljaju istu grešku
- `postgres` pod radi normalno

**Dijagnoza**
```bash
kubectl -n ticketing get secret ticketing-db-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d; echo
kubectl -n ticketing logs deployment/worker | tail -20
kubectl -n ticketing exec -it deployment/postgres -- psql -U ticketing_user -d ticketing -c '\dt'
```

**Ispravak**
```bash
# ažuriraj Secret na ispravnu lozinku pa restartaj potrošače
kubectl -n ticketing apply -f k8s/03-secret.yaml
kubectl -n ticketing rollout restart deployment/api deployment/worker
```

**Prevencija**
- Rotacija lozinki kroz isti Secret, restart potrošača nakon rotacije, izbjegavati ručno mijenjanje samo jedne strane.

**Kako reproducirati (za Sliku 14)**
```bash
kubectl -n ticketing patch secret ticketing-db-secret \
  -p '{"stringData":{"POSTGRES_PASSWORD":"kriva-lozinka"}}'
kubectl -n ticketing rollout restart deployment/api
curl http://ticketing.local/api/readyz     # -> auth failed   (SLIKAJ)
# vrati ispravnu lozinku:
kubectl -n ticketing apply -f k8s/03-secret.yaml
kubectl -n ticketing rollout restart deployment/api deployment/worker
```

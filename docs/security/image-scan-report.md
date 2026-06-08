# Sigurnosno izvješće skeniranja slika

Alat: **Trivy** (skeniranje repozitorija na tajne + skeniranje OCI slika na ranjivosti).
Quality gate: pipeline pada na ranjivostima **CRITICAL** i **HIGH** (`exit-code: 1`).

## 1. Skeniranje repozitorija na tajne
Naredba (lokalno):
```bash
trivy fs --scanners secret .
```
Rezultat: _ovdje zalijepi izlaz (očekivano: nema pronađenih tajni)._

## 2. Skeniranje slika na ranjivosti
Naredbe (lokalno, nakon builda runtime slika):
```bash
trivy image --severity CRITICAL,HIGH --ignore-unfixed ghcr.io/<GHCR_USER>/ticketing-api:v1.0.0
trivy image --severity CRITICAL,HIGH --ignore-unfixed ghcr.io/<GHCR_USER>/ticketing-frontend:v1.0.0
trivy image --severity CRITICAL,HIGH --ignore-unfixed ghcr.io/<GHCR_USER>/ticketing-worker:v1.0.0
```

| Slika | CRITICAL | HIGH | Napomena |
|-------|----------|------|----------|
| ticketing-api | _x_ | _x_ | _popuni iz izlaza_ |
| ticketing-frontend | _x_ | _x_ | _popuni iz izlaza_ |
| ticketing-worker | _x_ | _x_ | _popuni iz izlaza_ |

## 3. Politika označavanja i objave
- Svaka slika se gradi iz `runtime` faze (minimalna, non-root `node`).
- Oznake: `vMAJOR.MINOR.PATCH` (semver) + kratki `git SHA` (nepromjenjivi trag commita).
- Slika se objavljuje u GHCR **samo** ako oba skeniranja prođu (quality gate).
- `latest` se ne koristi za produkcijski deploy.

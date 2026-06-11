# Image Scan Report

Skeniranje ranjivosti provedeno je Trivy alatom u tri navrata: lokalno pri izradi slika i u CI pipelineu na svakom pushu na main granu.

---

## Nalaz 1 – CVE-2026-33671

| Polje | Vrijednost |
|---|---|
| CVE | CVE-2026-33671 |
| Severity | HIGH |
| Paket | picomatch |
| Instalirana verzija | 4.0.3 |
| Ispravljena verzija | >= 4.0.4 |
| Lokacija | `/usr/local/lib/node_modules/npm/node_modules/` (bundlani npm unutar node:22-alpine) |
| Slika | ticketing-api |
| Datum nalaza | 2026-06-08 |
| Datum ispravka | 2026-06-08 |
| Status | FIXED |

**Analiza:** Ranjivost nije bila u aplikacijskim ovisnostima nego u npm-u koji dolazi bundlan s Node.js Docker imageom. Aplikacijski `package.json` ne koristi picomatch direktno.

**Ispravak:** Dodan `RUN npm install -g npm@latest` u runtime stage sva tri Containerfile-a (api, frontend, worker). Ažurirani npm povlači picomatch >= 4.0.4.

**Verifikacija:** Ponovljeno skeniranje ticketing-api slike nakon ispravka – nema HIGH ni CRITICAL nalaza. Preostao je jedan MEDIUM nalaz (CVE-2026-41907, uuid u bundlanom npm-u) koji ne blokira quality gate jer pipeline filtrira samo CRITICAL i HIGH.

---

## Nalaz 2 – CVE-2026-45447

| Polje | Vrijednost |
|---|---|
| CVE | CVE-2026-45447 |
| Severity | HIGH |
| Paket | libssl3, libcrypto3 (OpenSSL) |
| Instalirana verzija | 3.5.6-r0 |
| Ispravljena verzija | 3.5.7-r0 |
| Lokacija | Alpine 3.24 OS sloj unutar node:22-alpine |
| Slike | ticketing-api, ticketing-frontend, ticketing-worker |
| Datum nalaza | 2026-06-10 |
| Datum ispravka | 2026-06-10 |
| Status | FIXED |

**Analiza:** Zakrpana verzija libssl3/libcrypto3 bila je dostupna u službenom Alpine repozitoriju, ali bazni Docker image node:22-alpine još nije sadržavao update. Ranjivost nije u aplikacijskom kodu.

**Ispravak:** Dodan `RUN apk update && apk upgrade --no-cache` u runtime stage sva tri Containerfile-a, čime se OS paketi nadograđuju na najnovije dostupne verzije pri svakom buildu.

**Verifikacija:** Ponovljeno skeniranje sve tri slike nakon ispravka – nema HIGH ni CRITICAL nalaza.

---

## Pregled statusa

| CVE | Severity | Paket | Status |
|---|---|---|---|
| CVE-2026-33671 | HIGH | picomatch 4.0.3 | FIXED |
| CVE-2026-45447 | HIGH | libssl3/libcrypto3 3.5.6-r0 | FIXED |
| CVE-2026-41907 | MEDIUM | uuid 10.0.0 | ACCEPTED (ne blokira quality gate) |

---

## Quality gate

CI pipeline (`build-scan-push` job) pokreće `trivy image --exit-code 1 --severity CRITICAL,HIGH` za svaku sliku prije pusha na GHCR. Push je moguć samo ako skeniranje prođe bez nalaza u tim kategorijama. Skeniranje repozitorija na tajne (`secret-scan` job) pokreće `trivy fs --scanners secret --exit-code 1` i mora proći bez ijednog nalaza.

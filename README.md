# GitOps Repo — BooksLib

Repo ini adalah sumber kebenaran (single source of truth) untuk deployment **BooksLib** menggunakan **Argo CD** + **Helm** + **External Secrets Operator (ESO)** + **Vault** di Kubernetes (RKE2).

Yang disimpan di sini adalah **deklarasi deploy** (Argo CD Application, values per environment, manifest ESO, dan routing Gateway). Source code aplikasi dan Helm chart berada di repo aplikasi `walsptr/bookslib`.

## Gambaran singkat

- **Repo aplikasi** (`walsptr/bookslib`)
  - Berisi source code + Helm chart (`helm/*`)
  - Jenkins membangun image Docker dan mendorongnya ke DockerHub
- **Repo GitOps** (repo ini)
  - Berisi values Helm per environment (`environments/*`)
  - Berisi manifest Argo CD (`argocd/*`)
  - Berisi manifest ESO (`eso/*`)
  - Berisi manifest Gateway API (`gateway/*`)
  - Jenkins mengubah **hanya `image.tag`** pada file values env (dev langsung, prod lewat promote)

## Struktur folder

- `argocd/`
  - `project-bookslib.yaml` — AppProject Argo CD
  - `app-of-apps/` — root application untuk dev/prod
  - `apps/` — application per komponen (services, secrets, gateway)
- `environments/`
  - `dev/*.values.yaml` — values Helm untuk environment dev
  - `prod/*.values.yaml` — values Helm untuk environment prod
- `eso/`
  - `dev/` dan `prod/` — SecretStore, ExternalSecret, ServiceAccount untuk Vault + ESO
- `gateway/`
  - `shared.yaml` — Gateway (listener) untuk `*.syawal.local`
  - `dev.yaml` — HTTPRoute dev (subdomain `*-dev.syawal.local`)
  - `prod.yaml` — HTTPRoute prod (subdomain `*.syawal.local`)

## Environment & namespace

- Dev: namespace `bookslib-dev`
- Prod: namespace `bookslib-prod`

## Bootstrap (sekali di awal)

Prasyarat:
- Argo CD sudah terinstall di cluster
- External Secrets Operator (ESO) sudah terinstall (CRD ada)
- Vault sudah siap (KV v2 + Kubernetes auth + policy + role)

1) Apply AppProject:

```bash
kubectl apply -n argocd -f argocd/project-bookslib.yaml
```

2) Apply root App-of-Apps:

```bash
kubectl apply -n argocd -f argocd/app-of-apps/dev.yaml
kubectl apply -n argocd -f argocd/app-of-apps/prod.yaml
```

Setelah root app dibuat, Argo CD akan menarik semua application di `argocd/apps/*`.

## Secrets (Vault + ESO)

ESO menarik secret dari Vault dan membuat Kubernetes Secret yang dibaca aplikasi melalui `existingSecretName` di values Helm.

- Dev: `eso/dev/*`
- Prod: `eso/prod/*`

Catatan:
- Jika Vault memakai self-signed certificate, SecretStore dapat menggunakan `caProvider` (Secret berisi CA cert) atau `caBundle`.
- Role Vault dev/prod dibuat terpisah (contoh: `bookslib-dev-eso`, `bookslib-prod-eso`).

## Routing (NGINX Gateway Fabric)

Repo ini memakai Gateway API untuk subdomain:

- Dev:
  - `auth-dev.syawal.local`
  - `books-dev.syawal.local`
  - `reviews-dev.syawal.local`
  - `bookslib-dev.syawal.local`
- Prod:
  - `auth.syawal.local`
  - `books.syawal.local`
  - `reviews.syawal.local`
  - `bookslib.syawal.local`

Pastikan DNS/hosts kamu mengarah ke IP Gateway (LoadBalancer/NodePort sesuai setup lab).

## Alur CI/CD

### Dev (langsung deploy)

1) Push commit ke branch `dev` di repo aplikasi
2) Jenkins build & push image dengan tag immutable (short SHA)
3) Jenkins update `environments/dev/*.values.yaml` (hanya `image.tag`) dan commit ke branch `main` repo GitOps
4) Argo CD auto-sync dev → deploy ke namespace `bookslib-dev`

### Prod (promote via PR)

1) Push commit ke branch `prod` di repo aplikasi
2) Jenkins build & push image tag immutable (short SHA)
3) Jenkins update `environments/prod/*.values.yaml` lalu push ke branch GitOps `promote/<tag>`
4) Buat PR `promote/<tag>` → `main` di repo GitOps
5) Setelah PR di-merge, Argo CD deploy prod (manual/auto sesuai `syncPolicy` application prod)

### Main (tag `latest`, tanpa GitOps)

Branch `main` di repo aplikasi dipakai untuk publish image `:latest` agar orang yang mau mencoba bisa install via Helm chart saja (tanpa flow GitOps). Repo GitOps tidak perlu menambah environment khusus untuk `main`.

## Troubleshooting singkat

- HTTPRoute healthy tapi akses 404:
  - pastikan Hostname yang diakses sesuai `hostnames` HTTPRoute
  - pastikan DNS mengarah ke IP Gateway yang benar
  - cek status `Accepted/ResolvedRefs` di `kubectl describe httproute ...`
- ESO SecretStore gagal:
  - masalah DNS (`no such host`) → periksa CoreDNS / mapping domain lokal
  - masalah TLS (`unknown authority`) → pasang CA cert via `caProvider` / `caBundle`
  - 403 permission denied → biasanya `token_reviewer_jwt` Vault perlu diperbarui atau RBAC token reviewer belum benar


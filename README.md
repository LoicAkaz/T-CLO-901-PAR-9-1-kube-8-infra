# T-CLO-901-PAR-9-1-kube-8-infra

Dépôt GitOps **infrastructure** du projet KUBE : tous les composants de plateforme, réconciliés par
ArgoCD. Il remplace le dossier `platform/` du dépôt principal (ADR-0020 de ce dépôt-là).

- Dépôt principal (Ansible, documentation, ADR) : `EpitechMscProPromo2027/T-CLO-901-PAR-9-1-kube-8`
- Dépôt applicatif : `T-CLO-901-PAR-9-1-kube-8-app`

> **Ce dépôt est public** : ArgoCD le lit sans identifiants. Aucun secret en clair, jamais :
> uniquement des `SealedSecret` (Sealed Secrets, ADR-0009). Relire chaque PR dans cet esprit.

## Responsabilité

- Application root (app-of-apps) et une Application ArgoCD par composant, ordonnées par sync waves.
- Exposition et TLS : configuration Traefik, cert-manager, ClusterIssuers.
- Secrets : contrôleur Sealed Secrets.
- Stockage : EFS CSI driver, PV / StorageClass.
- Identité, observabilité, dashboard, politiques d'admission (waves 2 à 5).
- Application ArgoCD qui pointe vers le dépôt applicatif (wave 6).

## Limite

- En amont : le cluster et ArgoCD sont fournis par Ansible (dépôt principal). Le bootstrap crée
  l'Application `root`, qui pointe sur le dossier `root/` de ce dépôt (branche `main`).
- En aval : l'application, son chart et sa base de données sont dans le dépôt applicatif.

## Structure

```
root/                  # cible du bootstrap Ansible : référence apps/
apps/                  # une Application ArgoCD par composant (annotation sync-wave)
components/
└── <composant>/
    ├── base/
    └── overlays/prod/
```

Convention : un composant livré par un chart tiers n'a que son Application (`apps/`), avec la version du
chart figée dans `targetRevision`. Un composant fait de manifests a son dossier dans `components/`.

## Sync waves

| Wave | Contenu | État |
|---|---|---|
| 0 | cert-manager (CRDs inclus), Sealed Secrets, EFS CSI | Application écrite |
| 1 | ClusterIssuers, config Traefik, StorageClass EFS | ClusterIssuers écrits ; StorageClass EFS statique écrite |
| 2 | PostgreSQL Keycloak, Keycloak | PostgreSQL écrit ; Keycloak à faire |
| 3 | dex | à faire |
| 4 | SSO ArgoCD, monitoring, logging, Headlamp | Métriques et alertes écrites, sans Ingress ni SSO ; le reste à faire |
| 5 | Admission policies, RBAC | Policies écrites en mode audit ; RBAC à faire |
| 6 | Application de test `whoami` (M3), puis Application → dépôt applicatif | `whoami` écrite ; dépôt applicatif à faire |

## TLS des outils

Un certificat par Ingress (ADR-0024 du dépôt principal) : annotation `cert-manager.io/cluster-issuer`,
`tls.hosts` et `secretName` du tableau. Pas de `Certificate` multi-SAN ni de `TLSStore`.

| Hostname (`.56.228.56.225.sslip.io`) | Issuer | `secretName` |
|---|---|---|
| `keycloak` | `letsencrypt-prod` | `keycloak-tls` |
| `dex` | `letsencrypt-prod` | `dex-tls` |
| `argocd` | `letsencrypt-staging` | `argocd-tls` |
| `grafana` | `letsencrypt-staging` | `grafana-tls` |
| `headlamp` | `letsencrypt-staging` | `headlamp-tls` |

- L'API server doit faire confiance à dex, et dex à Keycloak : ces deux hôtes exigent un certificat prod.
  `argocd`, `grafana` et `headlamp` passent en prod pour la soutenance, un commit par Ingress.
- **Un seul essai par hostname prod.** Le quota de `sslip.io` est partagé et ne se récupère pas : en cas de
  `rateLimited`, pas de nouvel essai, on remonte le cas. Un revert vers staging ne rétablit pas l'OIDC
  (repli : `PLAN.md` §7 du dépôt principal, M4-Q1). Plafond : 4 rebuilds par semaine.
- Tout Ingress prod demande la validation d'une seconde personne avant le merge.

## Documentation

Guide GitOps (dépôts, Application root, waves, ajout d'un composant, déploiement par commit, staging vers prod) :
`docs/gitops.md` du dépôt principal. Ce README ne garde que le tableau des waves ci-dessus.

## Suivi

Tâches : [PLAN.md](PLAN.md).

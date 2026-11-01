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
| 1 | ClusterIssuers, config Traefik | ClusterIssuers écrits |
| 2 | PostgreSQL Keycloak, Keycloak | à faire |
| 3 | dex | à faire |
| 4 | SSO ArgoCD, monitoring, logging, Headlamp | à faire |
| 5 | Admission policies, RBAC | à faire |
| 6 | Application → dépôt applicatif | à faire |

## Suivi

Tâches : [PLAN.md](PLAN.md).

# Plan — dépôt infra

Périmètre et limites : [README.md](README.md). Jalons visés : **M3** (waves 0 et 1), **M4** (waves 2 à 6).
Migré depuis `platform/PLAN.md` du dépôt principal (ADR-0020). Jalons et contrats avec les autres domaines :
`PLAN.md` du dépôt principal.

## Terminé quand

- Un cluster neuf converge seul après le bootstrap Ansible, toutes les Applications `Synced` + `Healthy`.
- Chaque outil est exposé en HTTPS et accessible par SSO.
- Chaque composant porte les labels `app.kubernetes.io/*` et des requests/limits.
- Les livrables du contrat (`PLAN.md` du dépôt principal, §3) sont en place.
- `docs/gitops.md` et `docs/security.md` sont écrits.

---

## 1. Squelette (M3)

- [x] Créer l'arborescence décrite dans le [README](README.md#structure)
- [x] Application root dans `root/`, pattern app-of-apps (écrit, jamais synchronisé sur un cluster)
- [x] Convention kustomize : `base/` + `overlays/prod/` par composant
- [ ] Versions figées : `targetRevision` de chaque chart, tag de chaque image (charts des waves 0 figés ; reste les futurs composants)
- [ ] Application `argocd` : ArgoCD se gère lui-même depuis ce dossier, même version de chart que le bootstrap (ADR-0019)

## 2. Wave 0 — Fondations (M3)

- [x] CRDs (cert-manager : `crds.enabled`)
- [x] cert-manager v1.21.2 (écrit, jamais synchronisé)
- [x] Sealed Secrets, chart 2.20.0 (écrit ; le contrôleur doit reprendre la clé restaurée par Ansible, rôle `sealed_secrets_key` pas encore écrit)
- [ ] EFS CSI driver : chart 4.5.2 écrit ; StorageClass ou PV statiques, `persistentVolumeReclaimPolicy: Retain`, à faire (ID du système de fichiers et instance profile IAM, ADR-0014 « À trancher ») (PV statiques retenus après l'essai `efs-ap`, refusé par l'IAM ; StorageClass `efs-static` écrite, #49)
- [ ] Essai PostgreSQL sur un access point EFS (permissions UID/GID, démarrage, redémarrage du pod) — valide ADR-0014 et ADR-0018 avant Keycloak

## 3. Wave 1 — Exposition (M3)

- [x] ClusterIssuers `letsencrypt-staging` et `letsencrypt-prod` (écrits, sans adresse e-mail ACME)
- [ ] `HelmChartConfig` Traefik
- [ ] Ingress HTTPS d'ArgoCD : `argocd.<ip>.sslip.io`
- [ ] Issuer staging par défaut ; l'issuer prod est réservé à dex et à la soutenance
- [ ] ValidatingAdmissionPolicies déployées en mode audit (`validationActions: [Warn, Audit]`) (écrites, à déployer)
- [ ] Mesurer la mémoire consommée à ce stade et la comparer au budget

## 4. Wave 2 — Fournisseur d'identité (M4)

- [ ] PostgreSQL de Keycloak sur EFS (écrit sur PV statique, access point à créer)
- [ ] Keycloak : realm, clients, groupes `k8s-admins` / `k8s-viewers`
- [ ] Ingress HTTPS : `keycloak.<ip>.sslip.io`

## 5. Wave 3 — Fédération OIDC (M4)

- [ ] dex : connecteur OIDC Keycloak
- [ ] Clients statiques ArgoCD, Grafana, Headlamp, kubectl
- [ ] Ingress HTTPS `dex.<ip>.sslip.io` avec **certificat prod**
- [ ] Vérifier que l'issuer et le client-id correspondent aux flags k3s posés par Ansible

## 6. Wave 4 — Outils (M4)

- [ ] SSO ArgoCD via dex + RBAC ArgoCD par groupe
- [ ] VictoriaMetrics k8s stack + Grafana (OIDC) + règles d'alerte, `grafana.<ip>.sslip.io` (métriques et règles d'alerte écrites, alertes sans destinataire ; Ingress et SSO à faire)
- [ ] Loki monolithique (EFS) + Alloy
- [ ] Headlamp (OIDC), `headlamp.<ip>.sslip.io`
- [ ] Dashboards : nœuds, ressources par namespace, Traefik, ArgoCD

## 7. Wave 5 — Sécurité (M4)

- [ ] Passer les ValidatingAdmissionPolicies en refus (`Deny`), bindings `namespaceSelector policy.kube/enforced=true`, une fois les avertissements du mode audit corrigés
  - Prérequis : `failurePolicy: Fail`, owner GHCR et registry BDD figés, image de `whoami` déplacée sur GHCR ou namespace retiré du label, filtre `ownerReferences` resserré (solveur HTTP-01 de cert-manager, opérateurs), objets en suppression exclus, conteneurs éphémères couverts, warnings relevés par `apiserver_validating_admission_policy_check_total` (ArgoCD ne les affiche pas)
- [ ] ClusterRoleBindings sur les groupes OIDC
- [ ] Vérifier `kubectl` via kubelogin pour un admin et un viewer

## 8. Wave 6 — Raccordement de l'application

- [ ] Application de test `whoami` en TLS staging (écrite, à déployer ; en place jusqu'au dépôt app, M5)
- [ ] Application ArgoCD pointant sur `deploy/overlays/prod/` du dépôt app
- [ ] Namespace applicatif labellisé `policy.kube/enforced=true`

## 9. Documentation

- `docs/gitops.md` : tâche suivie dans le dépôt principal (ticket #42), pas ici
- [ ] `docs/security.md` : OIDC, RBAC, admission policies, gestion des secrets
- [ ] `docs/operations.md` (partie plateforme) : troubleshooting, renouvellement des certificats

## Bonus

- [ ] kube-rbac-proxy devant VictoriaMetrics et Loki (multi-tenancy)
- [ ] NetworkPolicies (deny-all par défaut + ouvertures explicites)
- [ ] Scan de sécurité des manifests (Checkov)

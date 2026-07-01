# Feuille de route Smeltry

> Mise à jour : 2026-06-26
> Architecture détaillée : [CLAUDE.md](./CLAUDE.md)

---

## Ordre d'implémentation recommandé

Le chemin critique suit les dépendances techniques : l'auth est un pré-requis de tout, le mécanisme de reboot doit être décidé avant le provisioning, et les interfaces (UI + CLI) se construisent sur un backend stabilisé.

```
Epic 8 ──────────────────────────────────────────────────── (continu dès le début)
Epic 1 → Epic 2 → Epic 9 → Epic 3 → Epic 5 → Epic 4 → Epic 10 → Epic 6 + Epic 7 → Epic 11 → Epic 12 → Epic 13 → Epic 14
```

| Ordre | Epic | Statut | Justification |
|-------|------|--------|---------------|
| 0 | **Epic 8 — Supply chain** | ✅ SBOM + Cosign mergé (PR #2) ; CI sans Nix (PR #5) | CI/CD et SBOM en place dès le premier commit sur chaque repo |
| 1 | **Epic 1 — Auth** | ✅ ADR-002 + ClusterRoles smeltry-admin/catalog-reader mergés (PR #10) | Pré-requis absolu : sans OIDC sur le kube-apiserver, rien ne se teste de bout en bout |
| 2 | **Epic 2 — Tenants** | ✅ Controller + 9 tests envtest mergés (PR #1) | Dépend de l'Epic 1 (RoleBindings sur groupes Authentik) ; crée le socle RBAC |
| 3 | **Epic 9 — Reboot PXE** | ✅ ADR-001 rédigé et mergé (PR #3) ; prototype à valider | L'ADR doit être rédigé avant de coder les Epics 3 et 4 — mécanisme sur le chemin critique du provisioning |
| 4 | **Epic 3 — ServerClaim** | ✅ Controller + 10 tests mergés (PR #4) | Plus simple qu'un cluster (1 machine, pas de CAPI, pas d'addons) ; valide le pipeline Netbox → machinecfg → Tinkerbell |
| 5 | **Epic 5 — Addons** | ✅ 7 tests stepWatchAddons mergés (PR #9) ; stories 5.1–5.4 couvertes | Pré-requis direct de l'Epic 4 (étape 6 de la séquence ClusterClaim) |
| 6 | **Epic 4 — ClusterClaim** | ✅ Controller + 18 tests (PR #6/7) ; scale up/down + délai de grâce (PR #8) | Feature centrale ; implémentée une fois les Epics 3, 5 et 9 stabilisés |
| 7 | **Epic 10 — Audit log** | ✅ Implémenté (PR #11) | CRD `AuditEvent`, purge controller TTL, émission best-effort depuis `ClusterClaim` |
| 8 | **Epic 6 — Headlamp** | ⏳ À faire | UI sur le backend stabilisé |
| 8 | **Epic 7 — CLI** | ⏳ À faire | Peut être développée en parallèle de l'Epic 6 (même kube-apiserver en backend) |
| 9 | **Epic 11 — Accounting** | ⏳ À faire | Métriques Prometheus ; nécessite un backend opérationnel pour avoir des données utiles |
| 10 | **Epic 12 — Release train** | ⏳ À faire | Umbrella chart et `smeltry install` ; a du sens une fois les composants matures |
| 11 | **Epic 13 — Sécurité opérationnelle** | ⏳ À faire | Durcissement et Network Policies ; finalise le produit avant ouverture publique |
| 12 | **Epic 14 — Documentation** | ⏳ À faire | En continu, finalisée quand le comportement observable est stable |

> **Décision tranchée** : l'Epic 5 utilise `capi-addon-provider` (azimuth-cloud) — pas Sveltos. Voir CLAUDE.md §3.6. Le controller ClusterClaim implémenté (PR #4) est déjà aligné sur cette décision.

---

## Definition of Done

Toute story est considérée comme terminée lorsque :

- [ ] Les tests passent (`go test ./...` ou équivalent selon le repo)
- [ ] La pull request a été approuvée par au moins un mainteneur
- [ ] Chaque commit porte un `Signed-off-by:` (DCO)
- [ ] La documentation est mise à jour si le comportement observable a changé
- [ ] Un SBOM est généré pour tout livrable binaire ou image de conteneur

---

## Échelle de complexité

| Taille | Ordre de grandeur |
|--------|-------------------|
| XS | Moins de 2h — changement isolé, fichier unique |
| S  | Demi-journée — 1 à 2 fichiers |
| M  | 1 à 3 jours — plusieurs composants |
| L  | 1 à 2 semaines — nouveau composant ou nouveau CRD |
| XL | Plus de 2 semaines — intégration système complète |

---

## v1

### Epic 1 — Authentification et identité

> Stack : Authentik, OIDC, kube-apiserver

---

#### Story — Connexion administrateur via Authentik `S`

```gherkin
Feature: Authentification administrateur

  Scenario: L'administrateur se connecte avec ses identifiants Authentik
    Given un compte Authentik appartenant au groupe "smeltry-admins"
    When l'administrateur accède à l'interface Headlamp et lance l'authentification
    Then il est redirigé vers Authentik, s'authentifie et revient sur Headlamp avec un token valide
    And il peut accéder aux vues d'administration

  Scenario: L'administrateur utilise des identifiants incorrects
    Given un compte Authentik avec un mot de passe erroné
    When il tente de s'authentifier
    Then Authentik retourne une erreur d'authentification
    And Headlamp affiche un message d'erreur sans token émis

  Scenario: Le compte n'appartient pas au groupe "smeltry-admins"
    Given un compte Authentik valide hors du groupe "smeltry-admins"
    When l'utilisateur s'authentifie et tente d'accéder aux vues d'administration
    Then le kube-apiserver refuse la requête avec un code 403
    And un message indique que l'accès est restreint aux administrateurs
```

---

#### Story — Connexion utilisateur via Authentik `S`

```gherkin
Feature: Authentification utilisateur

  Scenario: L'utilisateur se connecte avec ses identifiants Authentik
    Given un compte Authentik appartenant au groupe d'un tenant existant
    When l'utilisateur accède à Headlamp et lance l'authentification
    Then il est redirigé vers Authentik, s'authentifie et revient avec un token valide
    And il voit uniquement les ressources de son namespace tenant

  Scenario: L'utilisateur utilise des identifiants incorrects
    Given un compte Authentik avec un mot de passe erroné
    When il tente de s'authentifier
    Then Authentik retourne une erreur et aucun token n'est émis
```

---

#### Story — Validation des tokens OIDC par le kube-apiserver `M`

```gherkin
Feature: Validation OIDC côté kube-apiserver

  Scenario: Une requête avec un token valide est acceptée
    Given le kube-apiserver est configuré avec l'issuer Authentik et le client-id "smeltry-cli"
    And un token JWT valide émis par Authentik avec le claim "email" et le claim "groups"
    When une requête API Kubernetes est envoyée avec ce token en Bearer
    Then le kube-apiserver accepte la requête et applique le RBAC correspondant au groupe

  Scenario: Une requête avec un token expiré est rejetée
    Given un token JWT expiré (champ "exp" dépassé)
    When une requête API est envoyée avec ce token
    Then le kube-apiserver retourne HTTP 401 Unauthorized

  Scenario: Un token émis par un issuer inconnu est rejeté
    Given un token JWT valide signé par un issuer différent d'Authentik
    When une requête API est envoyée avec ce token
    Then le kube-apiserver retourne HTTP 401 Unauthorized
```

---

#### Story — Injection du scope `groups` dans le JWT `S`

```gherkin
Feature: Scope groups dans le JWT Authentik

  Scenario: Un utilisateur membre d'un groupe voit ses groupes dans le token
    Given un utilisateur Authentik appartenant au groupe "acme"
    And un Property Mapping de scope "groups" configuré dans Authentik
    When l'utilisateur s'authentifie et obtient un token
    Then le JWT contient le claim "groups" avec la valeur ["acme"]

  Scenario: Un utilisateur sans groupe reçoit un claim groups vide
    Given un utilisateur Authentik n'appartenant à aucun groupe
    When il s'authentifie et obtient un token
    Then le JWT contient le claim "groups" avec une liste vide []
    And le kube-apiserver ne lui accorde aucun droit via les RoleBindings de groupe
```

---

### Epic 2 — Gestion des tenants

> Stack : NetboxTenant operator, Netbox, Kubernetes RBAC

---

#### Story — Création automatique du namespace tenant `M` ✅

```gherkin
Feature: Réconciliation du namespace tenant depuis Netbox

  Scenario: Un nouveau tenant apparaît dans Netbox
    Given un tenant Netbox avec le slug "acme"
    When le NetboxTenant operator effectue sa réconciliation (cycle de 5 min)
    Then un namespace Kubernetes "tenant-acme" est créé s'il n'existe pas

  Scenario: Le tenant est supprimé dans Netbox
    Given un namespace "tenant-acme" existant
    And le tenant "acme" a été supprimé dans Netbox
    When le NetboxTenant operator effectue sa réconciliation
    Then le namespace "tenant-acme" est supprimé avec ses ressources

  Scenario: Le namespace existe déjà (réconciliation idempotente)
    Given un namespace "tenant-acme" déjà présent dans le cluster
    When le NetboxTenant operator réconcilie le tenant "acme"
    Then aucun namespace supplémentaire n'est créé
    And l'état du namespace existant n'est pas altéré
```

---

#### Story — Application du ResourceQuota depuis les custom fields Netbox `M` ✅

```gherkin
Feature: ResourceQuota par tenant

  Scenario: Les custom fields Netbox définissent les quotas
    Given un tenant Netbox avec les champs "k8s_max_clusters: 3" et "k8s_max_nodes: 12"
    When le NetboxTenant operator réconcilie ce tenant
    Then un ResourceQuota est créé dans "tenant-acme" avec clusters=3 et nodes=12

  Scenario: Les custom fields sont absents
    Given un tenant Netbox sans les champs "k8s_max_clusters" ni "k8s_max_nodes"
    When le NetboxTenant operator réconcilie ce tenant
    Then un ResourceQuota est créé avec les valeurs par défaut définies dans la configuration de l'opérateur

  Scenario: Les quotas sont mis à jour dans Netbox
    Given un ResourceQuota "tenant-acme" avec clusters=3
    And le champ "k8s_max_clusters" passe à 5 dans Netbox
    When le NetboxTenant operator réconcilie ce tenant
    Then le ResourceQuota est mis à jour avec clusters=5
```

---

#### Story — Création du Role et RoleBinding RBAC `S` ✅

```gherkin
Feature: RBAC namespace tenant

  Scenario: Le Role et le RoleBinding sont créés pour un nouveau tenant
    Given un namespace "tenant-acme" venant d'être créé
    When le NetboxTenant operator complète la réconciliation
    Then un Role "cluster-user" existe dans "tenant-acme" avec les droits sur "clusterclaims" et "serverclaims"
    And un RoleBinding lie ce Role au groupe Authentik "acme"

  Scenario: Le RoleBinding est recréé s'il est supprimé manuellement
    Given un RoleBinding supprimé manuellement dans "tenant-acme"
    When le NetboxTenant operator effectue sa prochaine réconciliation
    Then le RoleBinding est recréé
```

---

### Epic 3 — Provisioning de serveurs (ServerClaim)

> Stack : ServerClaim CRD, Tinkerbell, machinecfg

---

#### Story — Commander un ou plusieurs serveurs OS nu `L` ✅

```gherkin
Feature: Provisioning d'un serveur physique avec Flatcar Linux

  Scenario: Un utilisateur crée un ServerClaim et obtient son serveur
    Given un utilisateur authentifié dans le namespace "tenant-acme"
    And des machines disponibles dans Netbox correspondant à la classe "standard" sur le site "paris-dc1"
    When il crée un ServerClaim avec machineClass="standard", site="paris-dc1", os="flatcar"
    Then l'opérateur alloue une machine dans Netbox (status=staged, tenant=acme)
    And un Job machinecfg est lancé pour provisionner la machine
    And le ServerClaim atteint la phase "Ready" avec l'IP du serveur dans status.serverIP

  Scenario: Aucune machine disponible pour la classe demandée
    Given aucune machine active de classe "standard" disponible dans Netbox pour "paris-dc1"
    When un utilisateur crée un ServerClaim
    Then le ServerClaim passe en phase "Failed"
    And une condition explicite indique "NoMachineAvailable"

  Scenario: Le Job machinecfg échoue
    Given une machine allouée dans Netbox
    When le Job Kubernetes machinecfg se termine en erreur
    Then le ServerClaim passe en phase "Failed"
    And la machine est restituée dans Netbox (status=active, tenant=vide)
```

---

#### Story — Configuration des clés SSH sur la machine `M`

```gherkin
Feature: Clés SSH sur les serveurs provisionnés

  Scenario: Les clés SSH sont appliquées lors du provisioning
    Given un ServerClaim avec une liste de clés SSH publiques dans spec.sshKeys
    When la machine est provisionnée par Tinkerbell via machinecfg
    Then les clés SSH sont présentes dans ~/.ssh/authorized_keys de l'utilisateur système
    And l'utilisateur peut se connecter en SSH à l'IP du serveur

  Scenario: Une clé SSH est au mauvais format
    Given un ServerClaim avec une clé SSH malformée
    When le ServerClaim est soumis au kube-apiserver
    Then le webhook d'admission rejette la ressource avec un message d'erreur explicite
```

---

#### Story — Configuration optionnelle de Tailscale `M`

```gherkin
Feature: Client Tailscale sur les serveurs provisionnés

  Scenario: Tailscale est activé avec une auth key valide
    Given un ServerClaim avec spec.tailscale.enabled=true et une auth key Tailscale valide
    When la machine est provisionnée
    Then le client Tailscale est installé et enregistré sur le réseau Tailscale
    And une condition "TailscaleReady" est présente dans le status du ServerClaim

  Scenario: L'auth key Tailscale est invalide ou expirée
    Given un ServerClaim avec spec.tailscale.enabled=true et une auth key invalide
    When la machine est provisionnée
    Then la machine est opérationnelle (phase "Ready") mais Tailscale n'est pas connecté
    And une condition "TailscaleDegraded" avec le motif d'erreur est ajoutée au status
```

---

#### Story — Suivi de l'état d'avancement du provisioning `S`

```gherkin
Feature: Visibilité de la progression du ServerClaim

  Scenario: L'utilisateur suit la progression en temps réel
    Given un ServerClaim en cours de provisioning
    When l'utilisateur consulte le status dans Headlamp ou via kubectl
    Then il voit la phase courante (Pending → Provisioning → Ready)
    And les conditions Kubernetes indiquent l'étape en cours

  Scenario: Le provisioning est bloqué au-delà du délai configuré
    Given un ServerClaim en phase "Provisioning" depuis plus longtemps que le timeout
    When l'opérateur détecte le dépassement de délai
    Then le ServerClaim passe en phase "Failed"
    And une condition "ProvisioningTimeout" est ajoutée au status
```

---

### Epic 4 — Provisioning de clusters Kubernetes (ClusterClaim)

> Stack : ClusterClaim CRD, CAPI, CAPT, Kamaji, Cilium, Netbox IPAM, CoreDNS

---

#### Story — Créer un cluster Kubernetes `XL` 🔄

```gherkin
Feature: Création d'un cluster Kubernetes sur bare metal

  Scenario: Un utilisateur crée un cluster complet avec succès
    Given un utilisateur authentifié dans "tenant-acme"
    And des machines disponibles de classe "gpu-large" sur le site "paris-dc1"
    And un AddonProfile "gpu-compute" existant dans "portal-system"
    When il crée un ClusterClaim avec machineClass="gpu-large", machineCount=3, site="paris-dc1", addonProfile="gpu-compute"
    Then l'opérateur enchaîne les phases : Provisioning → ClusterReady → AddonsReady → Ready
    And un TenantControlPlane Kamaji, un Cluster CAPI et un MachineDeployment sont créés
    And le cluster est accessible via le kubeconfig disponible dans Headlamp

  Scenario: La classe de machines n'est pas compatible avec l'AddonProfile
    Given un AddonProfile "gpu-compute" exigeant le tag Netbox "gpu"
    And des machines de classe "standard" sans le tag "gpu"
    When un ClusterClaim est créé avec cet AddonProfile et cette classe
    Then le ClusterClaim passe en phase "Failed" dès la validation
    And une condition "IncompatibleMachineClass" est présente dans le status

  Scenario: Nombre de machines insuffisant
    Given seulement 2 machines disponibles pour la classe et le site demandés
    When un ClusterClaim est créé avec machineCount=3
    Then le ClusterClaim passe en phase "Failed"
    And une condition "InsufficientMachines" indique le nombre disponible
```

---

#### Story — Réservation de l'IP du control plane dans Netbox IPAM `M` ✅

```gherkin
Feature: Réservation IP control plane

  Scenario: L'IP du control plane est réservée automatiquement
    Given un ClusterClaim valide avec site="paris-dc1"
    And un prefix de provisioning "10.0.1.0/24" défini dans le SiteConfig
    When l'opérateur commence la phase de réservation IP
    Then une IP est réservée dans Netbox IPAM avec dns_name="ml-training-api.acme.infra.example.com"
    And l'ID de cette IP est stocké dans status.netboxIPAMIDs pour libération au finalizer

  Scenario: Le prefix Netbox est saturé
    Given un prefix "10.0.1.0/24" sans adresse disponible
    When l'opérateur tente de réserver une IP
    Then le ClusterClaim passe en phase "Failed"
    And une condition "IPAMExhausted" est ajoutée au status
```

---

#### Story — Réservation de l'IP webhook ingress dans Netbox IPAM `M` ✅

```gherkin
Feature: Réservation IP webhook

  Scenario: L'IP du webhook est réservée automatiquement
    Given un ClusterClaim avec une IP control plane déjà réservée
    When l'opérateur réserve l'IP du webhook
    Then une seconde IP est réservée dans Netbox avec dns_name="ml-training-wh.acme.infra.example.com"
    And son ID est ajouté à status.netboxIPAMIDs

  Scenario: La réservation de l'IP webhook échoue après succès de l'IP control plane
    Given l'IP control plane réservée avec succès
    And le prefix IPAM est désormais saturé pour la seconde réservation
    When l'opérateur tente de réserver l'IP webhook
    Then le ClusterClaim passe en phase "Failed"
    And l'IP control plane déjà réservée est libérée par le finalizer
```

---

#### Story — Configuration automatique du DNS via CoreDNS et Netbox `M`

```gherkin
Feature: DNS automatique via Netbox et CoreDNS

  Scenario: Le DNS est résolvable dès la réservation IP
    Given une IP réservée dans Netbox avec dns_name="ml-training-api.acme.infra.example.com"
    And CoreDNS configuré pour interroger Netbox
    When une résolution DNS est effectuée pour "ml-training-api.acme.infra.example.com"
    Then l'IP réservée est retournée, même avant que le cluster soit opérationnel

  Scenario: La libération IP supprime aussi l'entrée DNS
    Given une IP avec un dns_name actif dans Netbox
    When le finalizer du ClusterClaim supprime l'IP via l'API Netbox
    Then la résolution DNS de ce nom échoue (NXDOMAIN)
```

---

#### Story — Suivi de l'avancement de la création du cluster `S`

```gherkin
Feature: Progression du ClusterClaim

  Scenario: L'utilisateur voit la phase courante dans Headlamp
    Given un ClusterClaim en cours de création
    When l'utilisateur consulte la ressource dans Headlamp
    Then il voit la phase courante parmi : Pending, Provisioning, ClusterReady, AddonsReady, Ready
    And les conditions Kubernetes détaillent chaque étape (MachinesAllocated, ControlPlaneReady, AddonsReady…)

  Scenario: Une phase reste bloquée trop longtemps
    Given un ClusterClaim en phase "ClusterReady" depuis plus longtemps que le timeout configuré
    When l'opérateur détecte le dépassement
    Then le ClusterClaim passe en phase "Failed"
    And la condition "Timeout" précise la phase qui a échoué
```

---

#### Story — Kubeconfig disponible depuis Headlamp `M`

```gherkin
Feature: Accès au kubeconfig du cluster tenant

  Scenario: L'utilisateur télécharge le kubeconfig depuis Headlamp
    Given un ClusterClaim en phase "Ready" avec un kubeconfig stocké dans un Secret Kubernetes
    And le Role "cluster-user" du namespace inclut ce Secret dans ses resourceNames
    When l'utilisateur clique sur "Télécharger le kubeconfig" dans Headlamp
    Then le Secret kubeconfig lui est retourné
    And il peut l'utiliser pour accéder au cluster tenant avec kubectl

  Scenario: L'utilisateur d'un autre tenant tente d'accéder au kubeconfig
    Given un kubeconfig dans le namespace "tenant-acme"
    When un utilisateur du namespace "tenant-other" tente d'y accéder
    Then le kube-apiserver retourne HTTP 403
```

---

#### Story — Libération des IPs et machines à la suppression `M` ✅

```gherkin
Feature: Finalizer du ClusterClaim

  Scenario: La suppression libère toutes les ressources
    Given un ClusterClaim en phase "Ready" avec des IPs IPAM et des machines allouées
    When l'utilisateur supprime le ClusterClaim
    Then les IPs sont supprimées dans Netbox IPAM (DELETE /ipam/ip-addresses/{id}/)
    And les machines sont restituées dans Netbox (status=active, tenant=vide)
    And les objets CAPI sont supprimés (cascade via ownerReference)
    And le nom du Secret kubeconfig est retiré du Role "cluster-user"

  Scenario: La suppression d'une IP IPAM échoue en cours de finalizer
    Given une erreur réseau lors de la suppression de la seconde IP IPAM
    When le finalizer tente la suppression
    Then la suppression est replanifiée (le finalizer reste en place)
    And une condition d'erreur indique l'IP non libérée
```

---

#### Story — Remplacement d'un nœud défaillant `L`

```gherkin
Feature: Remplacement d'un nœud défaillant sans recréer le cluster

  Scenario: Un nœud passe en état NotReady et est remplacé automatiquement
    Given un nœud "worker-42" en état "NotReady" depuis plus longtemps que le seuil configuré
    And des machines disponibles de la même classe sur le même site
    When l'opérateur détecte le dépassement du seuil de tolérance
    Then il cordon et drain le nœud défaillant
    And alloue une nouvelle machine dans Netbox (status=staged)
    And lance un Job machinecfg pour provisionner la nouvelle machine
    And la nouvelle machine rejoint le cluster en remplacement

  Scenario: Aucune machine de remplacement n'est disponible
    Given un nœud défaillant et aucune machine disponible de la classe requise
    When l'opérateur tente d'allouer un remplacement
    Then une condition "NoReplacementAvailable" est ajoutée au ClusterClaim
    And une alerte Prometheus est émise
    And le nœud défaillant reste dans le cluster jusqu'à disponibilité d'une machine

  Scenario: Le remplacement est déclenché manuellement par l'administrateur
    Given un nœud dégradé mais encore en état "Ready"
    When l'administrateur annote le nœud avec "portal.smeltry.io/replace: true"
    Then l'opérateur déclenche la même procédure de remplacement qu'en cas de défaillance automatique
```

---

#### Story — Scale up d'un cluster existant `M`

```gherkin
Feature: Augmentation du nombre de nœuds d'un cluster

  Scenario: L'utilisateur augmente le nombre de nœuds
    Given un ClusterClaim "ml-training" en phase "Ready" avec machineCount=3
    And des machines disponibles de la même classe sur le même site
    When l'utilisateur modifie machineCount à 5 (via l'UI ou kubectl)
    Then l'opérateur alloue 2 machines supplémentaires dans Netbox (status=staged)
    And le MachineDeployment CAPI est patché avec replicas=5
    And les nouvelles machines rejoignent le cluster et la phase reste "Ready"

  Scenario: Pas assez de machines disponibles pour le scale up
    Given un ClusterClaim avec machineCount=3 et seulement 1 machine supplémentaire disponible
    When l'utilisateur tente de passer à machineCount=5
    Then l'opérateur pose une condition "InsufficientMachines" sur le ClusterClaim
    And machineCount reste à 3 jusqu'à disponibilité des ressources
```

---

#### Story — Scale down d'un cluster sans Rook-Ceph `M`

```gherkin
Feature: Réduction du nombre de nœuds d'un cluster

  Scenario: L'utilisateur réduit le nombre de nœuds sur un cluster sans Ceph
    Given un ClusterClaim "build-cluster" en phase "Ready" avec machineCount=5
    And l'AddonProfile associé ne contient pas rook-ceph
    When l'utilisateur modifie machineCount à 3
    Then l'opérateur sélectionne 2 nœuds à retirer, les draine (cordon + drain)
    And le MachineDeployment CAPI est patché avec replicas=3
    And les 2 machines libérées sont restituées dans Netbox (status=active, tenant=vide)

  Scenario: Le scale down est refusé si Rook-Ceph est installé
    Given un ClusterClaim "storage-cluster" avec rook-ceph dans l'AddonProfile
    When l'utilisateur tente de diminuer machineCount
    Then le webhook d'admission rejette la modification
    And le message d'erreur indique que le scale down est interdit en présence de Rook-Ceph
```

---

#### Story — Suppression d'un cluster avec délai de grâce `M`

```gherkin
Feature: Suppression sécurisée d'un ClusterClaim

  Scenario: L'utilisateur supprime un cluster et le délai de grâce s'écoule
    Given un ClusterClaim "ml-training" en phase "Ready"
    When l'utilisateur déclenche la suppression depuis l'UI
    Then l'annotation "portal.smeltry.io/delete-at" est posée avec l'heure d'expiry (now + délai configuré)
    And le cluster reste fonctionnel pendant le délai de grâce
    And à l'expiry, le finalizer s'exécute et libère toutes les ressources

  Scenario: L'utilisateur annule la suppression pendant le délai de grâce
    Given un ClusterClaim avec l'annotation "portal.smeltry.io/delete-at" active
    When l'utilisateur clique sur "Annuler la suppression" avant l'expiry
    Then l'annotation est retirée du ClusterClaim
    And le cluster reste en phase "Ready" sans modification

  Scenario: Le délai de grâce expire sans annulation
    Given un ClusterClaim avec "portal.smeltry.io/delete-at" dépassé
    When l'opérateur détecte l'expiry lors de sa réconciliation
    Then le finalizer s'exécute : IPs IPAM libérées, machines restituées, objets CAPI supprimés
```

---

### Epic 5 — Gestion des addons (Sveltos)

> Stack : Sveltos, AddonProfile CRD, Cilium, Rook-Ceph, Netbox

---

#### Story — Installation automatique des addons définis dans l'AddonProfile `L`

```gherkin
Feature: Déploiement des addons via Sveltos

  Scenario: Les addons requis sont installés dans l'ordre défini
    Given un ClusterClaim en phase "ClusterReady" référençant l'AddonProfile "gpu-compute"
    And l'AddonProfile définit cilium (order=1), ingress (order=2), rook-ceph (order=3)
    When l'opérateur crée les objets Sveltos pour ce cluster
    Then Cilium est déployé en premier (dès que le control plane est prêt)
    And ingress et rook-ceph sont déployés une fois le cluster entièrement Ready

  Scenario: Un addon requis échoue à s'installer
    Given un HelmRelease Sveltos pour "ingress" en erreur (chart introuvable)
    When Sveltos tente de réconcilier cet addon
    Then le ClusterClaim reste en phase "AddonsReady" non atteinte
    And une condition "AddonFailed" précise le nom de l'addon et le motif d'erreur
```

---

#### Story — Installation prioritaire de Cilium (bootstrap) `M`

```gherkin
Feature: Bootstrap de Cilium

  Scenario: Cilium est installé dès que le control plane est disponible
    Given un TenantControlPlane Kamaji en état "Ready"
    And Cilium marqué bootstrap=true dans l'AddonProfile
    When l'opérateur crée l'objet Sveltos pour Cilium
    Then Cilium est déployé avant que les workers rejoignent le cluster
    And les workers peuvent joindre le cluster avec le réseau CNI fonctionnel

  Scenario: Cilium n'est pas encore prêt quand le second addon démarre
    Given Cilium en cours d'installation
    When Sveltos tente de déployer "ingress" (bootstrap=false)
    Then le déploiement d'"ingress" est différé jusqu'à ce que le cluster soit Ready
```

---

#### Story — Configuration OSD Rook-Ceph depuis Netbox `M`

```gherkin
Feature: Configuration des disques OSD pour Rook-Ceph

  Scenario: Les disques OSD sont injectés depuis les données Netbox
    Given des machines allouées avec des inventory-items tagués "ceph-osd" dans Netbox
    And ces disques sont présents dans status.osdDevices du ClusterClaim
    When l'opérateur crée l'objet Sveltos pour rook-ceph
    Then les valeurs du chart incluent la liste des disques OSD par nœud (ex. {"42": ["sdb","sdc"]})
    And Rook-Ceph utilise uniquement ces disques pour les OSD

  Scenario: Aucun disque OSD trouvé dans Netbox pour les machines allouées
    Given des machines sans inventory-items tagués "ceph-osd"
    When l'opérateur prépare la configuration rook-ceph
    Then une condition "NoCephOSDFound" est ajoutée au ClusterClaim
    And le déploiement de rook-ceph est ignoré si l'addon est marqué required=false
```

---

#### Story — Phase AddonsReady atteinte quand tous les addons requis sont prêts `S`

```gherkin
Feature: Condition AddonsReady du ClusterClaim

  Scenario: Tous les addons requis sont Ready
    Given tous les HelmRelease Sveltos marqués required=true en état "Ready"
    When l'opérateur réconcilie le ClusterClaim
    Then la phase passe à "AddonsReady"
    And si aucune autre étape n'est en attente, la phase passe à "Ready"

  Scenario: Un addon optionnel est en erreur, les requis sont prêts
    Given rook-ceph (required=false) en erreur et cilium + ingress (required=true) Ready
    When l'opérateur réconcilie
    Then la phase "AddonsReady" est atteinte malgré l'erreur rook-ceph
    And une condition "AddonDegraded" signale l'état de rook-ceph sans bloquer
```

---

#### Story — Mises à jour automatiques des nœuds via Kured ou FLUO `M`

```gherkin
Feature: Addon de mise à jour automatique des nœuds

  Scenario: Kured est déployé comme addon optionnel sur un cluster
    Given un AddonProfile incluant "kured" avec required=false
    And un cluster en phase "Ready"
    When Sveltos installe l'addon kured
    Then Kured est déployé sur chaque nœud comme DaemonSet
    And les nœuds sont redémarrés automatiquement en cas de mise à jour noyau, un par un (cordon/drain)

  Scenario: FLUO est sélectionné à la place de Kured
    Given un AddonProfile incluant "fluo" avec required=false
    When Sveltos installe l'addon fluo
    Then FLUO orchestre les mises à jour OS de façon coordonnée sur le cluster

  Scenario: Ni Kured ni FLUO ne sont dans l'AddonProfile
    Given un AddonProfile sans addon de mise à jour automatique
    When le cluster atteint la phase "Ready"
    Then aucun mécanisme de mise à jour automatique n'est actif
    And aucune condition d'erreur n'est générée (c'est un choix explicite de l'administrateur)
```

---

### Epic 6 — Interface web (plugin Headlamp)

> Stack : Headlamp, plugin TypeScript, kube-apiserver

---

#### Story — Liste des clusters de l'utilisateur `M`

```gherkin
Feature: Vue liste des clusters

  Scenario: L'utilisateur voit ses clusters
    Given un utilisateur authentifié dans le namespace "tenant-acme"
    And deux ClusterClaims existants dans ce namespace
    When il accède à la vue "Mes clusters" dans Headlamp
    Then il voit uniquement les deux clusters du namespace "tenant-acme"
    And chaque ligne affiche le nom, la phase, le nombre de nœuds et le site

  Scenario: L'utilisateur n'a aucun cluster
    Given un namespace "tenant-acme" sans aucun ClusterClaim
    When l'utilisateur accède à la vue "Mes clusters"
    Then un message vide ("Aucun cluster") est affiché avec un bouton de création

  Scenario: Un utilisateur tente d'accéder aux clusters d'un autre tenant
    Given une requête API vers le namespace "tenant-other"
    When le kube-apiserver évalue les droits
    Then la requête est refusée avec HTTP 403 et le plugin affiche une erreur d'autorisation
```

---

#### Story — Vue de détail d'un cluster `M`

```gherkin
Feature: Vue de détail d'un cluster dans Headlamp

  Scenario: L'utilisateur consulte le détail de son cluster
    Given un ClusterClaim "ml-training" en phase "Ready"
    When l'utilisateur clique sur le cluster dans la liste
    Then il voit : la phase courante, la liste des nœuds avec leur état (Ready/NotReady), la version Kubernetes, les addons installés et leur état (via Sveltos), le FQDN Tailscale du control plane, l'adresse IP interne du control plane sur le réseau de bootstrap, et l'IP de webhook du cluster

  Scenario: Un nœud est NotReady
    Given un nœud du cluster en état "NotReady"
    When l'utilisateur consulte la vue de détail
    Then le nœud est mis en évidence avec son état et les conditions Kubernetes associées

  Scenario: Le cluster est en cours de création
    Given un ClusterClaim en phase "Provisioning"
    When l'utilisateur consulte la vue de détail
    Then il voit la progression étape par étape et les conditions en attente
    And les actions scale/suppression sont désactivées jusqu'à la phase "Ready"
```

---

#### Story — Scale up et scale down depuis l'UI `M`

```gherkin
Feature: Redimensionnement d'un cluster depuis Headlamp

  Scenario: L'utilisateur augmente le nombre de nœuds depuis l'UI
    Given un ClusterClaim "ml-training" en phase "Ready" avec machineCount=3
    When l'utilisateur clique sur "Ajouter des nœuds" et saisit 2 nœuds supplémentaires
    Then le ClusterClaim est patché avec machineCount=5 via l'API Kubernetes
    And la vue de détail reflète la progression de l'ajout

  Scenario: L'utilisateur tente un scale down sur un cluster avec Rook-Ceph
    Given un cluster avec rook-ceph installé
    When l'utilisateur tente de diminuer le nombre de nœuds dans l'UI
    Then le bouton "Retirer des nœuds" est désactivé
    And une info-bulle explique que le scale down est interdit en présence de Rook-Ceph

  Scenario: L'utilisateur réduit le nombre de nœuds sur un cluster sans Ceph
    Given un cluster sans rook-ceph avec machineCount=5
    When l'utilisateur clique sur "Retirer des nœuds" et saisit 2
    Then le ClusterClaim est patché avec machineCount=3
    And la vue de détail affiche la progression du drain et de la désallocation
```

---

#### Story — Suppression avec délai de grâce depuis l'UI `S`

```gherkin
Feature: Suppression sécurisée d'un cluster depuis Headlamp

  Scenario: L'utilisateur supprime un cluster et voit le compte à rebours
    Given un ClusterClaim "ml-training" en phase "Ready"
    When l'utilisateur clique sur "Supprimer" et confirme dans la boîte de dialogue
    Then l'annotation de suppression différée est posée sur le ClusterClaim
    And la vue de détail affiche un bandeau rouge avec le compte à rebours restant
    And un bouton "Annuler la suppression" est visible

  Scenario: L'utilisateur annule la suppression avant la fin du délai
    Given un cluster avec un compte à rebours de suppression en cours
    When l'utilisateur clique sur "Annuler la suppression"
    Then l'annotation est retirée via l'API Kubernetes
    And le bandeau disparaît, le cluster revient à son état normal

  Scenario: Le délai expire sans annulation
    Given un compte à rebours arrivé à zéro
    When l'opérateur réconcilie et détecte l'expiry
    Then la suppression s'exécute automatiquement et le cluster disparaît de la liste
```

---

#### Story — Choix du type de machine `M`

```gherkin
Feature: Sélection du type de machine par l'utilisateur

  Scenario: L'utilisateur consulte les types de machines disponibles sur un site
    Given un utilisateur authentifié souhaitant créer un cluster ou un serveur
    And des machines actives de plusieurs classes ("standard", "gpu-large", "storage") sur le site "paris-dc1"
    When il consulte le catalogue des types de machines dans l'interface
    Then il voit la liste des classes disponibles avec pour chacune : le nombre de machines actives, les caractéristiques matérielles (CPU, RAM, stockage) issues de Netbox et les tags associés

  Scenario: L'utilisateur sélectionne un type de machine incompatible avec l'AddonProfile choisi
    Given un AddonProfile "gpu-compute" exigeant le tag Netbox "gpu"
    And l'utilisateur sélectionne la classe "standard" sans tag "gpu"
    When il tente de valider son choix dans le formulaire
    Then l'interface signale l'incompatibilité avant soumission
    And le bouton de confirmation est désactivé avec un message explicite

  Scenario: Aucun type de machine n'est disponible sur le site sélectionné
    Given aucune machine active sur le site "lyon-dc2"
    When l'utilisateur sélectionne ce site dans le formulaire
    Then l'interface indique qu'aucune classe de machine n'est disponible sur ce site
    And il est invité à choisir un autre site
```

---

#### Story — Disponibilité des machines avant création de cluster `M`

```gherkin
Feature: Vérification de disponibilité des machines

  Scenario: Des machines sont disponibles pour la sélection faite
    Given des machines actives de classe "gpu-large" sur le site "paris-dc1" dans Netbox
    When l'utilisateur sélectionne machineClass="gpu-large", site="paris-dc1", machineCount=3 dans le formulaire
    Then le formulaire indique "3 machines disponibles" en vert
    And le bouton de création est actif

  Scenario: Pas assez de machines disponibles
    Given seulement 1 machine active de classe "gpu-large" sur "paris-dc1"
    When l'utilisateur sélectionne machineCount=3
    Then le formulaire indique "1 machine disponible sur 3 demandées" en orange
    And le bouton de création est désactivé avec un message explicite
```

---

#### Story — Formulaire de création d'un ClusterClaim `L`

```gherkin
Feature: Création de ClusterClaim depuis l'interface

  Scenario: L'utilisateur crée un cluster avec succès
    Given l'utilisateur est sur le formulaire de création de cluster
    And il sélectionne machineClass="standard", machineCount=3, site="paris-dc1", addonProfile="default"
    When il soumet le formulaire
    Then un ClusterClaim est créé dans son namespace via l'API Kubernetes
    And il est redirigé vers la vue de détail du cluster avec la phase "Provisioning"

  Scenario: Le formulaire est soumis avec des données invalides
    Given l'utilisateur laisse le champ "site" vide
    When il tente de soumettre le formulaire
    Then le formulaire affiche une erreur de validation côté client sans appel API

  Scenario: L'API Kubernetes retourne une erreur à la création
    Given un quota tenant déjà atteint (k8s_max_clusters épuisé)
    When l'utilisateur soumet le formulaire
    Then l'API retourne HTTP 403 avec le motif de quota
    And le formulaire affiche le message d'erreur retourné par le kube-apiserver
```

---

#### Story — Vue exhaustive des machines pour l'administrateur `M`

```gherkin
Feature: Vue administration des machines

  Scenario: L'administrateur voit toutes les machines de tous les tenants
    Given plusieurs machines provisionnées sur différents tenants et sites
    When l'administrateur accède à la vue "Machines" dans Headlamp
    Then il voit toutes les machines avec leur statut Netbox, leur tenant et leur site
    And il peut filtrer par site, classe et statut

  Scenario: Une machine est en état dégradé
    Given une machine avec une condition "ProvisioningFailed" dans son ServerClaim
    When l'administrateur consulte la vue machines
    Then cette machine est mise en évidence avec son statut d'erreur et le motif
```

---

#### Story — Vue exhaustive des clusters pour l'administrateur `M`

```gherkin
Feature: Vue administration des clusters

  Scenario: L'administrateur voit tous les clusters de tous les tenants
    Given des ClusterClaims répartis sur plusieurs namespaces tenants
    When l'administrateur accède à la vue "Clusters" dans Headlamp
    Then il voit tous les clusters avec leur phase, tenant, site et nombre de nœuds

  Scenario: L'administrateur filtre par tenant
    Given plusieurs clusters répartis sur les tenants "acme" et "beta"
    When l'administrateur sélectionne le filtre tenant="acme"
    Then seuls les clusters du namespace "tenant-acme" sont affichés
```

---

#### Story — Supervision de l'état de santé des machines `M`

```gherkin
Feature: Santé des machines dans l'interface admin

  Scenario: L'administrateur consulte l'état de santé des machines
    Given des machines provisionnées avec des métriques disponibles (statut Netbox + métriques système)
    When l'administrateur accède à la vue de santé des machines dans Headlamp
    Then il voit pour chaque machine : statut Netbox, disponibilité réseau, état du nœud Kubernetes si applicable

  Scenario: Une machine devient injoignable
    Given une machine précédemment en état "Ready"
    When la machine ne répond plus aux vérifications de disponibilité
    Then la vue de santé signale la machine comme dégradée
    And une alerte est visible dans l'interface administrateur

  Scenario: L'administrateur exporte l'état de santé
    Given plusieurs machines avec des statuts variés
    When l'administrateur clique sur "Exporter"
    Then un fichier CSV ou JSON est généré avec l'état de santé de toutes les machines
```

---

#### Story — Gestion des SiteConfigs et AddonProfiles depuis l'UI admin `L`

```gherkin
Feature: Catalogue d'infrastructure administrable depuis Headlamp

  Scenario: L'administrateur crée un nouveau SiteConfig
    Given l'administrateur accède à la vue "Sites" dans Headlamp
    When il remplit le formulaire (nom, CIDR provisioning, zone DNS, pool Cilium L2, paramètres OIDC)
    Then un SiteConfig est créé dans le namespace "portal-system" via le kube-apiserver
    And le site apparaît immédiatement dans le sélecteur de site des formulaires tenant

  Scenario: L'administrateur modifie un AddonProfile existant
    Given un AddonProfile "gpu-compute" avec cilium, ingress et rook-ceph
    When l'administrateur ajoute le composant "gpu-operator" depuis la vue de détail
    Then l'AddonProfile est patché dans portal-system
    And les nouveaux ClusterClaims référençant ce profil incluront gpu-operator

  Scenario: Un tenant tente d'accéder à la gestion du catalogue
    Given un utilisateur sans le ClusterRole "smeltry-admin"
    When il tente d'accéder aux vues de gestion des SiteConfigs et AddonProfiles
    Then le kube-apiserver retourne HTTP 403 et Headlamp affiche une erreur d'autorisation
```

---

#### Story — Téléchargement du kubeconfig `S`

```gherkin
Feature: Téléchargement du kubeconfig depuis Headlamp

  Scenario: L'utilisateur télécharge son kubeconfig
    Given un ClusterClaim en phase "Ready" avec un Secret kubeconfig accessible
    When l'utilisateur clique sur "Télécharger le kubeconfig" dans la vue de détail
    Then le fichier kubeconfig est téléchargé dans son navigateur
    And il peut l'utiliser immédiatement avec kubectl

  Scenario: Le cluster n'est pas encore prêt
    Given un ClusterClaim en phase "Provisioning"
    When l'utilisateur consulte la vue de détail
    Then le bouton "Télécharger le kubeconfig" est désactivé avec le message "Cluster non disponible"
```

---

### Epic 7 — CLI (`smeltry`)

> Stack : Go, Cobra, device flow OIDC

> **Sources de données par sous-commande** :
> - `smeltry auth` → Authentik (IDP) — device flow, refresh token
> - `smeltry cluster`, `smeltry server`, `smeltry admin tenant/addon-profile/site/audit` → kube-apiserver (Bearer token OIDC)
> - `smeltry admin machine list` → Netbox (CMDB) — source de vérité des machines
> - `smeltry install`, `smeltry status`, `smeltry version` → kube-apiserver + ghcr.io (release manifest)
>
> Le kube-apiserver n'est pas exposé publiquement. Le mécanisme de connectivité (Tailscale, VPN, accès réseau direct) est laissé à l'appréciation de l'opérateur — Smeltry n'impose pas de solution réseau spécifique. Pas de backend applicatif intermédiaire (décision CLAUDE.md §3.1).

---

#### Story — Vérification de la connectivité aux APIs `S`

```gherkin
Feature: Vérification de la connectivité réseau avant toute commande

  Scenario: Toutes les APIs sont joignables
    Given le kube-apiserver, Authentik et Netbox accessibles depuis le poste (via Tailscale, VPN ou réseau direct selon l'installation)
    When l'utilisateur exécute n'importe quelle commande smeltry
    Then la CLI atteint les trois sources de données sans erreur réseau

  Scenario: Le kube-apiserver est injoignable
    Given le kube-apiserver non accessible depuis le poste
    When l'utilisateur exécute une commande smeltry nécessitant le kube-apiserver
    Then la CLI détecte le timeout de connexion
    And affiche un message explicite : "Le cluster de management est injoignable — vérifiez votre accès réseau"
    And quitte avec un code non-zéro

  Scenario: smeltry status vérifie la connectivité vers chaque source
    Given un utilisateur souhaitant diagnostiquer son installation
    When il exécute "smeltry status"
    Then la CLI vérifie et affiche l'état de la connectivité vers kube-apiserver, Authentik et Netbox séparément
```

---

#### Story — `smeltry auth` — Authentification `M`

```gherkin
Feature: Gestion de l'authentification CLI

  Scenario: Première authentification via device flow
    Given aucun token stocké dans ~/.config/smeltry/token.json
    When l'utilisateur exécute "smeltry auth login"
    Then la CLI affiche un code et une URL Authentik dans le terminal
    And après validation dans le navigateur, le token est stocké localement

  Scenario: Le token est expiré — rafraîchissement automatique
    Given un token expiré avec un refresh token valide
    When l'utilisateur exécute n'importe quelle commande smeltry
    Then la CLI rafraîchit silencieusement le token sans interaction

  Scenario: L'utilisateur vérifie son identité courante
    Given un token valide
    When il exécute "smeltry auth status"
    Then la CLI affiche l'email, le ou les groupes et la date d'expiration du token

  Scenario: L'utilisateur se déconnecte
    Given un token stocké localement
    When il exécute "smeltry auth logout"
    Then le fichier ~/.config/smeltry/token.json est supprimé
    And la prochaine commande nécessitant une authentification redemande un login
```

---

#### Story — `smeltry cluster` — Gestion des clusters `M`

```gherkin
Feature: Gestion des ClusterClaims via la CLI

  Scenario: Lister ses clusters
    Given un token valide et deux ClusterClaims dans le namespace tenant
    When l'utilisateur exécute "smeltry cluster list"
    Then la CLI affiche nom, phase, site, machineCount et age pour chaque cluster

  Scenario: Créer un cluster
    Given des machines disponibles de classe "gpu-large" sur "paris-dc1"
    When l'utilisateur exécute "smeltry cluster create --name ml-training --class gpu-large --count 3 --site paris-dc1 --addon-profile gpu-compute"
    Then un ClusterClaim est créé dans le namespace tenant
    And la CLI retourne le nom et la phase initiale "Pending"

  Scenario: Voir le détail d'un cluster
    Given un cluster "ml-training" en phase "Ready"
    When l'utilisateur exécute "smeltry cluster status ml-training"
    Then la CLI affiche : phase, nœuds et leur état, addons installés, FQDN Tailscale, IP bootstrap du control plane, IP webhook

  Scenario: Redimensionner un cluster
    Given un cluster "ml-training" en phase "Ready" avec machineCount=3
    When l'utilisateur exécute "smeltry cluster scale ml-training --count 5"
    Then le ClusterClaim est patché avec machineCount=5
    And la CLI confirme la prise en compte et invite à suivre avec "smeltry cluster status"

  Scenario: Supprimer un cluster avec délai de grâce
    Given un cluster "ml-training" en phase "Ready"
    When l'utilisateur exécute "smeltry cluster delete ml-training"
    Then l'annotation de suppression différée est posée sur le ClusterClaim
    And la CLI affiche le délai de grâce restant et la commande pour annuler

  Scenario: Obtenir le lien kubeconfig
    Given un cluster "ml-training" en phase "Ready"
    When l'utilisateur exécute "smeltry cluster kubeconfig ml-training"
    Then la CLI affiche l'URL deep-link Headlamp pour télécharger le kubeconfig

  Scenario: Quota épuisé à la création
    Given un quota tenant atteint
    When l'utilisateur exécute "smeltry cluster create ..."
    Then la CLI affiche l'erreur du kube-apiserver et quitte avec un code non-zéro
```

---

#### Story — `smeltry server` — Gestion des serveurs `M`

```gherkin
Feature: Gestion des ServerClaims via la CLI

  Scenario: Lister ses serveurs
    Given un token valide et un ServerClaim "build-01" dans le namespace tenant
    When l'utilisateur exécute "smeltry server list"
    Then la CLI affiche nom, phase, site, classe OS et age

  Scenario: Provisionner un serveur
    Given des machines disponibles de classe "standard" sur "paris-dc1"
    When l'utilisateur exécute "smeltry server create --name build-01 --class standard --site paris-dc1 --os flatcar"
    Then un ServerClaim est créé dans le namespace tenant
    And la CLI retourne le nom et la phase "Pending"

  Scenario: Voir le détail d'un serveur
    Given un serveur "build-01" en phase "Ready"
    When l'utilisateur exécute "smeltry server status build-01"
    Then la CLI affiche : phase, IP, DNS, OS, clés SSH configurées

  Scenario: Supprimer un serveur
    Given un serveur "build-01" en phase "Ready"
    When l'utilisateur exécute "smeltry server delete build-01"
    Then le ServerClaim est supprimé et la machine restituée dans Netbox
    And la CLI confirme la suppression
```

---

#### Story — `smeltry admin` — Sous-commandes d'administration `M`

```gherkin
Feature: Sous-commandes admin de la CLI (accès restreint par RBAC)

  Scenario: Lister toutes les machines
    Given un token admin avec le ClusterRole "smeltry-admin"
    When l'administrateur exécute "smeltry admin machine list"
    Then la CLI affiche toutes les machines Netbox : nom, site, classe, statut, tenant alloué
    And aucune action de modification n'est disponible (lecture seule — machinecfg gère le provisioning)

  Scenario: Lister tous les tenants et leurs quotas
    When l'administrateur exécute "smeltry admin tenant list"
    Then la CLI affiche chaque tenant avec ses quotas (max_clusters, max_nodes) et sa consommation courante

  Scenario: Lister les AddonProfiles disponibles
    When l'utilisateur exécute "smeltry admin addon-profile list"
    Then la CLI affiche les AddonProfiles du namespace portal-system avec leurs composants
    And cette commande est accessible aux tenants authentifiés (RBAC smeltry-catalog-reader)

  Scenario: Lister les sites disponibles
    When l'utilisateur exécute "smeltry admin site list"
    Then la CLI affiche les SiteConfigs avec leur zone DNS, CIDR de provisioning et pool Cilium L2
    And cette commande est accessible aux tenants authentifiés

  Scenario: Consulter l'audit log
    Given un administrateur souhaitant investiguer une suppression inattendue
    When il exécute "smeltry admin audit list --tenant acme --type ClusterDeleted"
    Then la CLI affiche les AuditEvents filtrés avec acteur, timestamp et ressource concernée

  Scenario: Un utilisateur non-admin tente d'accéder aux commandes admin restreintes
    Given un token tenant sans le ClusterRole "smeltry-admin"
    When l'utilisateur exécute "smeltry admin machine list"
    Then le kube-apiserver retourne HTTP 403
    And la CLI affiche "Permission refusée" et quitte avec un code non-zéro
```

---

#### Story — `smeltry version` — Version de la CLI et de l'installation `XS`

```gherkin
Feature: Affichage des versions

  Scenario: L'utilisateur consulte les versions
    Given la CLI smeltry v1.2.0 et Smeltry v1.2.0 installé sur le cluster
    When il exécute "smeltry version"
    Then la CLI affiche sa propre version et la version de chaque composant installé sur le cluster

  Scenario: La CLI n'est pas connectée à un cluster
    Given aucun kubeconfig ou token configuré
    When l'utilisateur exécute "smeltry version"
    Then la CLI affiche uniquement sa propre version et indique que la version cluster est indisponible
```

---

### Epic 8 — Supply chain et livraisons

> Stack : GitHub Actions, Syft/Cosign, ghcr.io

---

#### Story — Livrables cohérents pour tous les repos `L`

```gherkin
Feature: Pipeline de livraison uniforme

  Scenario: Un tag Git déclenche la publication d'un livrable
    Given un tag "v1.0.0" poussé sur le repo "smeltry-operator"
    When le workflow GitHub Actions se déclenche
    Then une image de conteneur est publiée sur ghcr.io/smeltry-io/smeltry-operator:v1.0.0
    And un binaire de la CLI smeltry est publié pour linux/amd64 et linux/arm64

  Scenario: La pipeline échoue si les tests ne passent pas
    Given un tag poussé avec des tests unitaires en échec
    When le workflow tente de publier le livrable
    Then la pipeline s'arrête avant la publication
    And aucune image ni binaire n'est publié sur ghcr.io
```

---

#### Story — SBOM associé et signé à chaque livrable `M` ✅

```gherkin
Feature: SBOM des livrables Smeltry

  Scenario: Un SBOM est généré et attaché à l'image de conteneur
    Given une image publiée sur ghcr.io/smeltry-io/smeltry-operator:v1.0.0
    When Syft génère le SBOM et Cosign le signe
    Then le SBOM est attaché à l'image dans le registre OCI
    And la signature peut être vérifiée avec "cosign verify ghcr.io/smeltry-io/smeltry-operator:v1.0.0"

  Scenario: La génération du SBOM échoue
    Given une erreur lors de l'exécution de Syft
    When la pipeline tente de générer le SBOM
    Then la pipeline échoue et le livrable n'est pas marqué comme publié
    And une notification est envoyée au mainteneur
```

---

### Epic 14 — Documentation et onboarding

> Stack : site de documentation `smeltry.io` (Hugo ou Docusaurus, à définir)

---

#### Story — Guide d'installation (quickstart) `M`

```gherkin
Feature: Quickstart pour un nouvel opérateur

  Scenario: Un opérateur installe Smeltry pour la première fois
    Given la documentation smeltry.io
    When il suit le guide "Getting Started"
    Then en moins de 30 minutes il a : un cluster de management avec Smeltry installé, un SiteConfig configuré, un premier tenant créé et un ClusterClaim qui atteint la phase "Ready"

  Scenario: Le quickstart liste les prérequis clairement
    Given un opérateur évaluant Smeltry
    When il consulte la page "Prerequisites"
    Then il voit la liste complète : cluster Kubernetes de management, Netbox avec les custom fields requis, Authentik, Tinkerbell, accès réseau aux APIs (kube-apiserver, Authentik, Netbox)
```

---

#### Story — Guide utilisateur tenant `S`

```gherkin
Feature: Documentation pour les utilisateurs tenants

  Scenario: Un tenant crée son premier cluster sans aide de l'admin
    Given la documentation smeltry.io section "User Guide"
    When le tenant suit le guide "Create your first cluster"
    Then il comprend : comment s'authentifier, comment choisir un AddonProfile et un site, comment suivre la progression, comment récupérer son kubeconfig

  Scenario: Un tenant trouve la réponse à une erreur courante
    Given un ClusterClaim en phase "Failed" avec la condition "InsufficientMachines"
    When le tenant consulte la section "Troubleshooting"
    Then il trouve l'explication de cette condition et les actions possibles (attendre, contacter l'admin, choisir un autre site)
```

---

#### Story — Documentation de référence des CRDs `M`

```gherkin
Feature: Référence des CRDs portal.smeltry.io

  Scenario: Un opérateur consulte la spec complète d'un ClusterClaim
    Given la documentation smeltry.io section "API Reference"
    When il cherche "ClusterClaim"
    Then il trouve tous les champs spec et status documentés avec leur type, leur caractère obligatoire et un exemple YAML complet

  Scenario: La documentation de référence est générée depuis les markers kubebuilder
    Given les markers "// +kubebuilder:..." dans le code Go de l'opérateur
    When la pipeline de documentation s'exécute
    Then la référence API est générée automatiquement et publiée sur smeltry.io sans saisie manuelle
```

---

### Epic 9 — Redémarrage PXE piloté (nouveau repo)

> Stack : Tinkerbell gRPC API, agent sur machine ou Job Kubernetes / SSH

> ⚠️ Sujet à étude préalable — les machines n'ont pas de BMC. Trois pistes à évaluer :
> 1. **Agent léger** installé sur chaque machine (service systemd ou conteneur), joignable par le cluster de management pour déclencher un redémarrage.
> 2. **Accès SSH** depuis un Job Kubernetes éphémère lancé par `smeltry-operator`.
> 3. **API gRPC Tinkerbell** (`github.com/tinkerbell/tink` — `HardwareService`, `WorkflowService`) pour piloter le cycle de vie des machines depuis le cluster.
>
> Un nouveau dépôt dédié devra être créé dans l'org `smeltry-io` pour héberger ce composant.

---

#### Story — Étude et choix du mécanisme de redémarrage `L` ✅

```gherkin
Feature: Étude du mécanisme de redémarrage PXE sans BMC

  Scenario: L'étude produit un ADR (Architecture Decision Record)
    Given les trois pistes identifiées (agent, SSH, gRPC Tinkerbell)
    When l'équipe évalue chaque piste sur les critères : sécurité, complexité opérationnelle, fiabilité
    Then un ADR est rédigé et fusionné dans le repo smeltry-operator
    And une piste est sélectionnée pour la mise en œuvre

  Scenario: Un prototype valide la piste retenue
    Given l'ADR approuvé avec la piste choisie
    When un prototype est développé et testé sur une machine réelle
    Then le prototype déclenche avec succès un redémarrage PXE sur une machine provisionnée
    And les résultats sont documentés dans l'ADR (section "Résultats du prototype")
```

---

#### Story — Création et scaffolding du nouveau repo `S`

```gherkin
Feature: Nouveau repo smeltry-io/tinkerbell-reboot-agent (nom à confirmer)

  Scenario: Le repo est créé avec la structure minimale
    Given la décision de créer un composant dédié au redémarrage PXE
    When le repo est créé dans l'org smeltry-io
    Then il contient : LICENSE (Apache 2.0), CONTRIBUTING.md, SECURITY.md, un Dockerfile minimal, un workflow de CI
    And le header SPDX est présent dans chaque fichier source

  Scenario: Le repo hérite des templates de l'org .github
    Given le repo .github de smeltry-io avec des templates d'issue et de PR
    When le nouveau repo est créé dans l'org
    Then les templates sont automatiquement disponibles sans duplication
```

---

#### Story — Déclenchement d'un redémarrage PXE depuis smeltry-operator `L`

```gherkin
Feature: Redémarrage PXE piloté par l'opérateur

  Scenario: L'opérateur déclenche un redémarrage PXE sur une machine
    Given une machine provisionnée identifiée par son ID Netbox
    And le mécanisme de redémarrage retenu est opérationnel (agent / SSH / gRPC)
    When smeltry-operator ordonne un redémarrage PXE (ex. pour re-provisioning)
    Then la machine redémarre en PXE et entre dans le workflow Tinkerbell correspondant
    And le status du ServerClaim ou ClusterClaim reflète la phase de re-provisioning

  Scenario: Le redémarrage n'aboutit pas dans le délai imparti
    Given une commande de redémarrage PXE envoyée
    And la machine ne répond pas au workflow Tinkerbell après le timeout
    Then une condition "RebootTimeout" est ajoutée au status de la ressource concernée
    And une alerte est visible dans l'interface administrateur
```

---

### Epic 10 — Audit log applicatif (CRD `AuditEvent`) ✅ PR #11

> Stack : CRD `AuditEvent` (`portal.smeltry.io/v1alpha1`), controller-runtime, Headlamp

> Chaque action métier significative (changement de phase, allocation de machine, réservation IP, suppression) crée une ressource `AuditEvent` dans le namespace concerné (`tenant-<slug>` pour les actions tenant, `portal-system` pour les actions admin). Le RBAC existant délimite naturellement la visibilité. Un champ `spec.ttl` et un controller de purge évitent la croissance non bornée de l'etcd.

> **Question ouverte — sémantique de l'émission :** l'implémentation actuelle traite l'émission d'un `AuditEvent` comme *best-effort* : une erreur est loguée mais ne bloque pas la réconciliation. Faut-il adopter une sémantique *blocking* pour certains types d'événements ? Arguments pour le *best-effort* : un échec d'écriture etcd (quota, indisponibilité temporaire) ne doit pas empêcher l'allocation d'une machine ou le passage de phase — l'infrastructure prime sur la traçabilité. Arguments pour le *blocking* : dans un contexte réglementaire ou de conformité, un événement non produit est un trou d'audit inacceptable ; on préfère alors échouer l'action plutôt que de la laisser passer silencieusement. Piste intermédiaire : rendre la sémantique configurable par type d'événement (`spec.required: true/false` sur l'appel, ou un flag opérateur `--audit-strict`). **À trancher avant d'exposer le log d'audit à des exigences de conformité.**

---

#### Story — Émission d'un AuditEvent par l'opérateur `L` ✅

```gherkin
Feature: Création d'AuditEvents par smeltry-operator

  Scenario: Un AuditEvent est créé à chaque changement de phase d'un ClusterClaim
    Given un ClusterClaim "ml-training" passant de la phase "Provisioning" à "ClusterReady"
    When l'opérateur réconcilie et met à jour le status
    Then un AuditEvent est créé dans le namespace "tenant-acme" avec : le nom de la ressource, l'ancienne phase, la nouvelle phase, le timestamp, et l'identité de l'acteur (email du token OIDC si disponible)

  Scenario: Un AuditEvent est créé lors de l'allocation d'une machine
    Given un ClusterClaim en cours de provisioning
    When l'opérateur alloue une machine dans Netbox
    Then un AuditEvent est créé avec le type "MachineAllocated", l'ID de la machine et le tenant

  Scenario: Un AuditEvent est créé lors de la suppression d'un ClusterClaim
    Given un ClusterClaim supprimé par un utilisateur
    When le finalizer s'exécute
    Then un AuditEvent de type "ClusterDeleted" est créé avec l'identité de l'acteur et le timestamp de la demande de suppression
```

---

#### Story — Vue audit log pour le tenant `M` ✅

```gherkin
Feature: Historique des actions pour l'utilisateur tenant

  Scenario: L'utilisateur consulte l'historique de ses ressources
    Given plusieurs AuditEvents dans le namespace "tenant-acme"
    When l'utilisateur accède à la vue "Historique" dans Headlamp
    Then il voit la liste chronologique des événements : type, ressource concernée, acteur, timestamp
    And il peut filtrer par ressource (nom du ClusterClaim ou ServerClaim)

  Scenario: Un membre de l'équipe a supprimé un cluster
    Given un AuditEvent "ClusterDeleted" avec l'acteur "alice@example.com" dans "tenant-acme"
    When un autre membre du tenant consulte l'historique
    Then il voit l'événement avec le nom de l'acteur et le timestamp exact

  Scenario: L'utilisateur n'a aucun événement dans son namespace
    Given un namespace "tenant-acme" sans AuditEvents
    When l'utilisateur accède à la vue "Historique"
    Then un message vide ("Aucun événement") est affiché
```

---

#### Story — Vue audit log global pour l'administrateur `M`

```gherkin
Feature: Historique global des actions pour l'administrateur

  Scenario: L'administrateur consulte tous les événements de tous les tenants
    Given des AuditEvents répartis sur les namespaces "tenant-acme", "tenant-beta" et "portal-system"
    When l'administrateur accède à la vue "Audit" dans Headlamp
    Then il voit tous les événements, tous tenants confondus, triés par timestamp décroissant
    And il peut filtrer par tenant, type d'événement et plage de dates

  Scenario: L'administrateur détecte une suppression inattendue
    Given un AuditEvent "ClusterDeleted" dans "tenant-acme" à 03h00
    When l'administrateur filtre sur le type "ClusterDeleted"
    Then l'événement apparaît avec l'acteur, le timestamp et le nom du cluster supprimé

  Scenario: Un tenant tente d'accéder aux AuditEvents d'un autre tenant
    Given un utilisateur du namespace "tenant-acme"
    When il tente de lister les AuditEvents du namespace "tenant-beta" via l'API
    Then le kube-apiserver retourne HTTP 403
```

---

#### Story — Purge automatique des AuditEvents via TTL `S` ✅

```gherkin
Feature: Expiration automatique des AuditEvents

  Scenario: Un AuditEvent est supprimé après expiration de son TTL
    Given un AuditEvent avec spec.ttl="720h" (30 jours) créé il y a 31 jours
    When le controller de purge effectue son passage
    Then l'AuditEvent est supprimé du namespace

  Scenario: Le TTL par défaut s'applique si non spécifié
    Given un AuditEvent créé sans spec.ttl explicite
    When le controller de purge évalue cet événement
    Then le TTL par défaut défini dans la configuration de l'opérateur s'applique

  Scenario: Un AuditEvent récent n'est pas purgé
    Given un AuditEvent créé il y a 5 jours avec spec.ttl="720h"
    When le controller de purge effectue son passage
    Then l'AuditEvent est conservé
```

---

### Epic 11 — Accounting et showback (Prometheus)

> Stack : controller-runtime metrics server, Prometheus, Grafana

> `smeltry-operator` expose des métriques custom sur son endpoint `/metrics` (fourni nativement par controller-runtime). Ces métriques décrivent la consommation par tenant, site et classe de machine. Un dashboard Grafana livré as-code permet aux équipes de visualiser leur consommation sans configuration supplémentaire.

---

#### Story — Exposition des métriques de consommation par l'opérateur `M`

```gherkin
Feature: Métriques Prometheus de consommation Smeltry

  Scenario: Les métriques reflètent l'état courant des clusters actifs
    Given deux ClusterClaims en phase "Ready" dans "tenant-acme" sur le site "paris-dc1"
    When Prometheus scrape l'endpoint /metrics de smeltry-operator
    Then la métrique "smeltry_clusters_active{tenant="acme", site="paris-dc1"}" vaut 2

  Scenario: Les métriques reflètent les machines allouées par classe
    Given 6 machines de classe "gpu-large" allouées sur l'ensemble des tenants
    When Prometheus scrape l'endpoint /metrics
    Then la métrique "smeltry_machines_allocated{class="gpu-large"}" vaut 6
    And des déclinaisons par tenant sont disponibles : "smeltry_machines_allocated{tenant="acme", class="gpu-large"}"

  Scenario: La durée de provisioning est mesurée
    Given un ClusterClaim passé de la phase "Pending" à "Ready" en 45 minutes
    When Prometheus scrape les métriques
    Then la métrique "smeltry_cluster_provisioning_duration_seconds{tenant="acme"}" enregistre cette durée dans son histogramme
```

---

#### Story — Dashboard Grafana showback livré as-code `M`

```gherkin
Feature: Dashboard Grafana de showback par tenant

  Scenario: L'administrateur importe le dashboard fourni par Smeltry
    Given un fichier JSON de dashboard Grafana versionné dans le repo helm-charts
    When l'administrateur l'importe dans Grafana (ou via le provisionning automatique Grafana)
    Then il voit un dashboard avec : nombre de clusters actifs par tenant, machines allouées par tenant et par classe, durée moyenne de provisioning, et évolution dans le temps

  Scenario: Un tenant consulte sa propre consommation
    Given un dashboard Grafana filtré par variable "tenant"
    When le tenant sélectionne son identifiant dans le filtre
    Then il voit uniquement ses propres métriques de consommation

  Scenario: Aucune donnée disponible pour un tenant
    Given un tenant "beta" sans ClusterClaim ni ServerClaim actif
    When le dashboard affiche les métriques du tenant "beta"
    Then les graphiques affichent zéro sans erreur
```

---

#### Story — Alerting proactif sur les événements critiques `M`

```gherkin
Feature: Alertes Prometheus sur les événements critiques Smeltry

  Scenario: Une alerte se déclenche quand un ClusterClaim reste en Failed
    Given un ClusterClaim en phase "Failed" depuis plus de 15 minutes
    When Alertmanager évalue les règles d'alerte Smeltry
    Then une alerte "SmeltryClusterFailed" est déclenchée avec le nom du cluster, le tenant et le motif
    And l'alerte est visible dans Grafana et transmise au canal configuré (email, Slack, PagerDuty…)

  Scenario: Une alerte se déclenche quand un nœud est NotReady sans remplacement en cours
    Given un nœud en état "NotReady" et la condition "NoReplacementAvailable" sur le ClusterClaim
    When Alertmanager évalue les règles
    Then une alerte "SmeltryNodeUnreplaceable" est déclenchée avec le contexte complet

  Scenario: Une alerte se déclenche quand la capacité en machines est critique
    Given moins de 10% de machines libres sur un site
    When Alertmanager évalue les règles
    Then une alerte "SmeltryLowMachineCapacity" est déclenchée avec le site, la classe et le nombre restant

  Scenario: Les règles d'alerte sont livrées as-code avec Smeltry
    Given le repo helm-charts avec les règles PrometheusRule
    When l'administrateur installe Smeltry
    Then les règles d'alerte sont déployées automatiquement si le CRD PrometheusRule est présent dans le cluster
```

---

#### Story — Documentation des métriques exposées `S`

```gherkin
Feature: Documentation des métriques Smeltry

  Scenario: Chaque métrique est documentée avec sa sémantique et ses labels
    Given la liste des métriques exposées par smeltry-operator
    When un opérateur consulte la documentation
    Then il trouve pour chaque métrique : le nom, le type (gauge/counter/histogram), la description, et la liste des labels avec leur signification

  Scenario: Les métriques internes controller-runtime sont distinguées des métriques métier
    Given l'endpoint /metrics exposant à la fois les métriques controller-runtime et les métriques Smeltry
    When un opérateur consulte la documentation
    Then les métriques métier Smeltry (préfixe "smeltry_") sont clairement identifiées et documentées séparément des métriques techniques
```

---

### Epic 12 — Release train (umbrella chart)

> Stack : Helm umbrella chart, `helm-charts` repo, GitHub Actions, `smeltry install`

> Un tag Git sur le repo `helm-charts` (ex. `v1.2.0`) définit une release atomique Smeltry : il épingle les versions exactes de l'opérateur, du plugin Headlamp et de la CLI dans un chart umbrella et un fichier `release.yaml`. `smeltry install` lit ce manifest pour tout installer en une commande. Un manifeste GitOps de référence (`HelmRelease` Flux ou `Application` ArgoCD) est livré pour les environnements qui gèrent leur infrastructure as-code.

---

#### Story — Orchestration de la construction de tous les artefacts depuis le release train `L`

```gherkin
Feature: Pipeline de release centralisée

  Scenario: Un tag déclenche la construction de tous les artefacts
    Given un tag "v1.2.0" poussé sur le repo release train (helm-charts)
    When la pipeline GitHub Actions se déclenche
    Then elle déclenche via repository_dispatch la construction de l'opérateur, de la CLI et du plugin Headlamp dans leurs repos respectifs
    And elle attend la complétion de chaque build avant de continuer
    And tous les artefacts publiés (images, binaires, charts) portent le tag v1.2.0

  Scenario: La construction d'un composant échoue
    Given le build de smeltry-operator qui se termine en erreur
    When la pipeline de release train reçoit l'événement d'échec
    Then la release est marquée comme échouée
    And aucun artefact partiel n'est publié sur ghcr.io ni dans les GitHub Releases
    And le mainteneur est notifié avec le lien vers le build en erreur

  Scenario: Les artefacts sont publiés de manière cohérente
    Given tous les builds de composants terminés avec succès
    When la pipeline de release train finalise la publication
    Then l'umbrella chart, le release.yaml, les images et les binaires CLI sont publiés atomiquement sous le même tag v1.2.0
```

---

#### Story — Lancement des tests d'intégration depuis le release train `L`

```gherkin
Feature: Tests d'intégration end-to-end dans la pipeline de release

  Scenario: Les tests d'intégration s'exécutent sur un cluster éphémère
    Given tous les artefacts v1.2.0 construits avec succès
    When la pipeline de release train lance la phase de tests d'intégration
    Then un cluster Kubernetes éphémère est provisionné (kind ou cluster de CI dédié)
    And Smeltry v1.2.0 est installé via "smeltry install" sur ce cluster
    And la suite de tests d'intégration valide les scénarios clés : création d'un ClusterClaim, provisioning d'un ServerClaim, réconciliation d'un tenant Netbox

  Scenario: Un test d'intégration échoue
    Given un test de création de ClusterClaim qui échoue en phase "Provisioning"
    When la pipeline détecte l'échec
    Then la release est bloquée — aucun artefact n'est publié
    And les logs du test sont attachés au run GitHub Actions pour investigation

  Scenario: Les tests passent et la release est validée
    Given tous les tests d'intégration terminés avec succès
    When la pipeline finalise la release
    Then les artefacts sont promus de l'environnement de CI vers ghcr.io et GitHub Releases
    And la GitHub Release est marquée comme "latest"
```

---

#### Story — Génération et attachement des SBOMs depuis le release train `M`

```gherkin
Feature: SBOM centralisé par la pipeline de release

  Scenario: Un SBOM est généré pour chaque artefact de la release
    Given les images et binaires CLI publiés pour la release v1.2.0
    When la pipeline de release train exécute la phase SBOM
    Then Syft génère un SBOM pour chaque image de conteneur (smeltry-operator, smeltry-headlamp)
    And un SBOM est généré pour chaque binaire CLI (linux/amd64, linux/arm64, darwin/amd64, darwin/arm64)
    And chaque SBOM est signé avec Cosign et attaché à l'artefact correspondant dans ghcr.io ou la GitHub Release

  Scenario: La génération du SBOM échoue pour un artefact
    Given une erreur Syft sur l'image smeltry-headlamp
    When la pipeline détecte l'échec
    Then la release est bloquée
    And un message d'erreur précise l'artefact concerné

  Scenario: Un utilisateur vérifie le SBOM d'un artefact
    Given l'image ghcr.io/smeltry-io/smeltry-operator:v1.2.0 avec son SBOM attaché
    When l'utilisateur exécute "cosign verify-attestation --type spdxjson ghcr.io/smeltry-io/smeltry-operator:v1.2.0"
    Then le SBOM au format SPDX est retourné et sa signature vérifiée
```

---

#### Story — Umbrella chart Smeltry avec dépendances versionnées `M`

```gherkin
Feature: Chart Helm umbrella pour la release atomique

  Scenario: L'umbrella chart installe tous les composants à des versions cohérentes
    Given un tag "v1.2.0" sur le repo helm-charts
    And l'umbrella chart déclarant smeltry-operator=1.2.0 et smeltry-headlamp=1.2.0 comme dépendances
    When un administrateur exécute "helm install smeltry smeltry-io/smeltry --version 1.2.0"
    Then l'opérateur et le plugin Headlamp sont installés aux versions exactes déclarées dans le chart
    And les CRDs sont appliqués avant le déploiement de l'opérateur

  Scenario: Une version incohérente est détectée
    Given un administrateur tentant d'installer smeltry-operator=1.2.0 avec smeltry-headlamp=1.1.0 manuellement
    When il soumet la commande Helm
    Then le chart umbrella refuse l'installation avec un message indiquant les versions attendues
```

---

#### Story — Fichier `release.yaml` épinglant la version de la CLI `S`

```gherkin
Feature: Manifest de release pour la CLI smeltry

  Scenario: La version de la CLI correspondant à la release est identifiable
    Given un fichier release.yaml versionné dans helm-charts à la racine
    And contenant : version globale, sha256 des binaires CLI par plateforme (linux/amd64, linux/arm64, darwin/amd64, darwin/arm64)
    When un administrateur consulte le fichier pour la release v1.2.0
    Then il peut télécharger le binaire CLI exact correspondant à cette release et vérifier son intégrité

  Scenario: Le release.yaml est mis à jour automatiquement lors du tag
    Given un tag "v1.2.0" poussé sur helm-charts
    When le workflow GitHub Actions de release s'exécute
    Then le release.yaml est mis à jour avec les checksums des binaires CLI publiés dans la même GitHub Release
```

---

#### Story — Commande `smeltry install` pour bootstrapper l'installation `M`

```gherkin
Feature: Bootstrapping de Smeltry via la CLI

  Scenario: L'administrateur installe Smeltry avec une seule commande
    Given un cluster Kubernetes de management accessible via kubeconfig
    And la CLI smeltry téléchargée pour la release v1.2.0
    When il exécute "smeltry install --version v1.2.0"
    Then la CLI lit le release.yaml correspondant depuis ghcr.io/smeltry-io/helm-charts
    And installe l'umbrella chart Helm avec les valeurs par défaut
    And confirme que l'opérateur est Running et les CRDs sont installés

  Scenario: Les prérequis sont vérifiés avant l'installation
    Given un cluster sans la StorageClass requise par Rook-Ceph
    When l'administrateur exécute "smeltry install"
    Then la CLI détecte le prérequis manquant et affiche un avertissement explicite avant de continuer

  Scenario: L'installation est rejouable sans effet de bord
    Given Smeltry déjà installé en v1.1.0
    When l'administrateur exécute "smeltry install --version v1.2.0"
    Then la CLI effectue une mise à jour Helm (upgrade) sans recréer les ressources existantes
    And les CRDs sont mis à jour sans perte de données
```

---

#### Story — Commande `smeltry status` pour vérifier l'installation `M`

```gherkin
Feature: État opérationnel de Smeltry via la CLI

  Scenario: Tous les composants sont opérationnels
    Given Smeltry v1.2.0 installé et opérationnel
    When l'administrateur exécute "smeltry status"
    Then la CLI affiche la version globale installée (v1.2.0)
    And pour chaque composant : smeltry-operator, smeltry-headlamp, les CRDs — la version exacte et l'état (Running / Degraded / Missing)
    And le code de sortie est 0

  Scenario: Un composant est dégradé
    Given le Deployment smeltry-operator avec 0 pod Ready sur 1 attendu
    When l'administrateur exécute "smeltry status"
    Then la ligne smeltry-operator est marquée "Degraded" avec le motif (ex. CrashLoopBackOff)
    And le code de sortie est non-zéro pour permettre l'intégration dans des scripts de vérification

  Scenario: Smeltry n'est pas installé sur le cluster ciblé
    Given un cluster Kubernetes sans Smeltry déployé
    When l'administrateur exécute "smeltry status"
    Then la CLI indique "Smeltry not found" avec une suggestion d'exécuter "smeltry install"
    And le code de sortie est non-zéro

  Scenario: La version de la CLI diffère de la version installée
    Given la CLI smeltry v1.3.0 utilisée contre un cluster avec Smeltry v1.2.0
    When l'administrateur exécute "smeltry status"
    Then un avertissement indique la discordance de versions
    And les informations d'état sont tout de même affichées
```

---

#### Story — Manifeste GitOps de référence livré avec la release `S`

```gherkin
Feature: Installation GitOps de Smeltry via Flux ou ArgoCD

  Scenario: Un administrateur installe Smeltry via Flux
    Given un fichier "gitops/flux/smeltry-helmrelease.yaml" fourni dans helm-charts
    And contenant un objet HelmRelease pointant vers le chart umbrella smeltry-io/smeltry à la version v1.2.0
    When l'administrateur applique ce manifeste dans son cluster Flux
    Then Flux installe et réconcilie Smeltry automatiquement
    And toute dérive de configuration est corrigée par Flux sans intervention manuelle

  Scenario: Un administrateur installe Smeltry via ArgoCD
    Given un fichier "gitops/argocd/smeltry-application.yaml" fourni dans helm-charts
    When l'administrateur applique ce manifeste dans son cluster ArgoCD
    Then ArgoCD synchronise l'installation de Smeltry et signale tout écart dans son interface
```

---

### Epic 13 — Sécurité opérationnelle

> Smeltry ne fournit pas de passerelle API. Pour les déploiements exposant le service à des utilisateurs non maîtrisés (publics ou multi-organisations), l'opérateur est responsable de placer une passerelle devant le kube-apiserver (Istio, Envoy, Cloudflare, nginx…). Smeltry documente cette exigence et avertit lorsque la configuration semble insuffisamment protégée.

---

#### Story — Guide de durcissement pour les déploiements publics `M`

```gherkin
Feature: Documentation de sécurité pour déploiements publics

  Scenario: Un opérateur déploie Smeltry pour des utilisateurs externes
    Given la documentation de Smeltry
    When l'opérateur consulte la section "Hardening"
    Then il trouve les recommandations explicites : placer une passerelle API devant le kube-apiserver, restreindre les endpoints de découverte Kubernetes, activer les Network Policies Cilium entre namespaces tenants, configurer le rate limiting au niveau de la passerelle

  Scenario: La documentation précise que Smeltry ne fournit pas la passerelle
    Given un opérateur évaluant Smeltry pour un usage public
    When il lit la documentation d'architecture
    Then il comprend clairement que la passerelle est de sa responsabilité
    And des exemples de configuration sont fournis pour les passerelles courantes (Envoy, nginx, Cloudflare Tunnel)
```

---

#### Story — Avertissement de `smeltry status` si le kube-apiserver est exposé publiquement `S`

```gherkin
Feature: Détection d'une exposition non sécurisée du kube-apiserver

  Scenario: Le kube-apiserver est accessible depuis une IP publique
    Given un kube-apiserver dont l'adresse de connexion résout sur une IP publique
    When l'administrateur exécute "smeltry status"
    Then la CLI affiche un avertissement explicite : "Le kube-apiserver semble exposé sur une IP publique — un déploiement public requiert une passerelle API devant le kube-apiserver"
    And le reste des informations de status est tout de même affiché

  Scenario: Le kube-apiserver est sur une adresse privée ou Tailscale
    Given un kube-apiserver accessible uniquement via une adresse RFC1918 ou un réseau overlay
    When l'administrateur exécute "smeltry status"
    Then aucun avertissement d'exposition n'est affiché
```

---

#### Story — Network Policies inter-namespaces tenants `M`

```gherkin
Feature: Isolation réseau entre namespaces tenants via Cilium

  Scenario: Un tenant ne peut pas joindre les ressources d'un autre tenant
    Given deux namespaces tenants "tenant-acme" et "tenant-beta" avec des Network Policies Cilium actives
    When un pod du namespace "tenant-acme" tente de joindre un service dans "tenant-beta"
    Then Cilium bloque la connexion
    And la tentative est visible dans les logs Cilium Hubble

  Scenario: Les composants Smeltry dans portal-system peuvent joindre tous les namespaces tenants
    Given smeltry-operator dans le namespace "portal-system"
    When il réconcilie un ClusterClaim dans "tenant-acme"
    Then la connexion vers le namespace "tenant-acme" est autorisée par la Network Policy
```

---

## v2

| Epic | Description |
|------|-------------|
| Intégration Rancher | Auto-import des clusters CAPI via le label `cluster-api.cattle.io/rancher-auto-import: "true"` et Rancher Turtles |
| Intégration Azimuth | Interface entre Azimuth (StackHPC) et les CRDs Smeltry — deux options envisagées (adaptateur CRD ou accès direct au kube-apiserver), à décider en v2 |

---

## Futur (non planifié)

| Thème | Description |
|-------|-------------|
| Virtualisation | Intégration Proxmox ou équivalent |
| OpenStack | Déploiement d'OpenStack sur bare metal |
| Audit log externe | Agrégation des AuditEvents vers Loki ou Elasticsearch pour rétention longue durée, recherche avancée et corrélation avec les métriques système |
| OCI bundle | Packaging de l'ensemble des composants Smeltry (opérateur, CLI, plugin Headlamp) dans un artefact OCI unique sur ghcr.io pour les environnements air-gapped et la distribution offline |

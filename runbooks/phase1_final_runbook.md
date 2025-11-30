# Phase 1 - Mass Rebuild Runbook

**Version:** 1.0  
**Date:** 2024-11-30  
**Status:** Ready for Execution

---

## Table of Contents

1. [But de la Phase 1](#but-de-la-phase-1)
2. [Sources de vérité](#sources-de-vérité)
3. [Principe général](#principe-général)
4. [Ordre exact d'exécution](#ordre-exact-dexécution)
5. [Fichiers produits](#fichiers-produits)
6. [Fiche de validation pré-dry-run (PH1-08)](#fiche-de-validation-pré-dry-run-ph1-08)
7. [Schéma d'architecture PH1-09](#schéma-darchitecture-ph1-09)
8. [Risques et mitigations](#risques-et-mitigations)

---

## But de la Phase 1

### Pourquoi on fait cette étape

La Phase 1 consiste à **rebuild massivement les 47 serveurs KeyBuzz v3** dans Hetzner Cloud avec Ubuntu 24.04, dans le but de :

1. **Standardiser l'infrastructure** : Tous les serveurs auront la même version d'OS (Ubuntu 24.04)
2. **Nettoyer l'environnement** : Suppression des volumes existants et reconstruction propre
3. **Préparer la base** : Serveurs prêts pour les phases suivantes (SSH mesh, XFS volumes, Kubernetes)
4. **Optimiser les performances** : Rebuild complet pour un état "clean slate"

### Ce qu'elle garantit

La Phase 1 garantit :

- ✅ **47 serveurs rebuildés** avec Ubuntu 24.04
- ✅ **Volumes supprimés** avant rebuild (nettoyage complet)
- ✅ **État "running"** pour tous les serveurs rebuildés
- ✅ **Port SSH 22 ouvert** sur toutes les IPs publiques
- ✅ **Bastions intacts** : `install-01` et `install-v3` ne sont **jamais touchés**
- ✅ **Traçabilité complète** : Logs détaillés pour chaque batch
- ✅ **Cohérence totale** : Tous les serveurs alignés avec `servers_v3.tsv`

### Pourquoi elle doit absolument être parfaite

Cette phase est **critique** car :

1. **Point de non-retour** : Une fois les volumes supprimés, les données sont perdues
2. **Base de toute l'infrastructure** : Les phases suivantes dépendent de cette base stable
3. **Impact massif** : 47 serveurs concernés - une erreur aurait des conséquences majeures
4. **Timing critique** : Le rebuild doit être rapide (< 1 heure) pour minimiser le downtime
5. **Cohérence obligatoire** : Tous les serveurs doivent être rebuildés dans le même état

**Toute erreur dans cette phase bloque les phases suivantes.**

---

## Sources de vérité

### Fichiers de configuration

#### 1. `servers/servers_v3.tsv`

**Rôle:** Source de vérité unique pour tous les serveurs KeyBuzz v3

**Contenu:**
- 49 serveurs au total
- Colonnes : `ENV`, `IP_PUBLIQUE`, `HOSTNAME`, `IP_PRIVEE`, `FQDN`, `USER_SSH`, `POOL`, `ROLE`, `SUBROLE`, `DOCKER_STACK`, `CORE`, `NOTES`, `ROLE_V3`, `LOGICAL_NAME_V3`

**Utilisation:**
- Génération de `ansible/inventory/hosts.yml`
- Génération de `servers/rebuild_order_v3.json`
- Référence pour toutes les opérations

**Localisation:** `/opt/keybuzz/keybuzz-infra/servers/servers_v3.tsv`

#### 2. `servers/rebuild_order_v3.json`

**Rôle:** Ordre de rebuild avec batching et métadonnées

**Contenu:**
```json
{
  "metadata": {
    "total_servers": 47,
    "total_batches": 10,
    "batch_size": 5,
    "excluded_servers": ["install-01", "install-v3"]
  },
  "servers": [...],
  "batches": [...]
}
```

**Utilisation:**
- Lecture par `reset_hetzner.yml`
- Définition des batches
- Exclusion des bastions

**Localisation:** `/opt/keybuzz/keybuzz-infra/servers/rebuild_order_v3.json`

#### 3. `ansible/playbooks/reset_hetzner.yml`

**Rôle:** Playbook Ansible optimisé pour rebuild massif en parallèle

**Caractéristiques:**
- Parallélisation avec `throttle: 10`
- Traitement par batches
- 100% API Hetzner (aucun SSH)
- Idempotent

**Localisation:** `/opt/keybuzz/keybuzz-infra/ansible/playbooks/reset_hetzner.yml`

### Scripts d'exécution

#### 1. `scripts/setup-hetzner-token.sh`

**Rôle:** Configuration du token Hetzner Cloud de manière sécurisée

**Fonctions:**
- Charge le token depuis variable d'environnement `HETZNER_API_TOKEN`
- Crée `/opt/keybuzz/credentials/hcloud.env` (chmod 600)
- Configure `~/.config/hcloud/cli.toml`
- Ajoute auto-loading dans `~/.bashrc`
- Teste la connexion hcloud

**Utilisation:**
```bash
export HETZNER_API_TOKEN="your-token"
bash scripts/setup-hetzner-token.sh
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/setup-hetzner-token.sh`

#### 2. `scripts/rename-postgres-api.py`

**Rôle:** Renommage des serveurs PostgreSQL via API Hetzner (avec pagination)

**Fonctions:**
- Renomme `db-master-01` → `db-postgres-01`
- Renomme `db-slave-01` → `db-postgres-02`
- Renomme `db-slave-02` → `db-postgres-03`
- Utilise pagination exhaustive (toutes les pages API)
- Gestion des erreurs avec `raise_for_status()`
- Idempotent (peut être ré-exécuté)

**Utilisation:**
```bash
python3 scripts/rename-postgres-api.py
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/rename-postgres-api.py`

#### 3. `scripts/generate_inventory.py`

**Rôle:** Génération automatique de l'inventaire Ansible depuis `servers_v3.tsv`

**Fonctions:**
- Lit `servers/servers_v3.tsv`
- Génère `ansible/inventory/hosts.yml`
- Crée 21 groupes Ansible
- Configure IP privée comme `ansible_host`
- Définit clé SSH et variables globales

**Utilisation:**
```bash
python3 scripts/generate_inventory.py > ansible/inventory/hosts.yml
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/generate_inventory.py`

#### 4. `scripts/generate_rebuild_order.py`

**Rôle:** Génération automatique de `rebuild_order_v3.json` depuis `servers_v3.tsv`

**Fonctions:**
- Lit `servers/servers_v3.tsv`
- Exclut `install-01` et `install-v3`
- Crée 10 batches de 5 serveurs (dernier batch: 2)
- Définit tailles de volumes par rôle
- Génère JSON structuré

**Utilisation:**
```bash
python3 scripts/generate_rebuild_order.py
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/generate_rebuild_order.py`

#### 5. `scripts/verify-inventory-rebuildorder.py`

**Rôle:** Vérification de cohérence entre inventaire et rebuild_order

**Fonctions:**
- Vérifie 49 serveurs dans l'inventaire
- Vérifie 47 serveurs dans rebuild_order
- Vérifie 10 batches
- Vérifie 21 groupes
- Vérifie volumes valides
- Vérifie exclusions

**Utilisation:**
```bash
python3 scripts/verify-inventory-rebuildorder.py
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/verify-inventory-rebuildorder.py`

#### 6. `scripts/execute-phase1.sh`

**Rôle:** Orchestration complète de la Phase 1

**Fonctions:**
- Exécute tous les scripts dans l'ordre
- Gère les erreurs à chaque étape
- Lance le playbook Ansible
- Génère le rapport final

**Ordre d'exécution:**
1. Setup token Hetzner
2. Rename PostgreSQL servers
3. Generate inventory
4. Generate rebuild_order
5. Verify coherence
6. Execute playbook
7. Generate report

**Utilisation:**
```bash
bash scripts/execute-phase1.sh
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/execute-phase1.sh`

#### 7. `scripts/phase1-report.sh`

**Rôle:** Génération de rapports détaillés post-rebuild

**Fonctions:**
- Liste tous les serveurs via API (pagination)
- Génère `/opt/keybuzz/reports/phase1/phase1-final.json`
- Génère `/opt/keybuzz/reports/phase1/phase1-final.md`
- Vérifie statut "running" pour 47 serveurs
- Vérifie que les bastions sont intacts

**Utilisation:**
```bash
bash scripts/phase1-report.sh
```

**Localisation:** `/opt/keybuzz/keybuzz-infra/scripts/phase1-report.sh`

---

## Principe général

### Architecture

```
install-v3 (bastion)
    ↓
    ├─→ Token Hetzner (setup-hetzner-token.sh)
    ├─→ Rename PostgreSQL (rename-postgres-api.py)
    ├─→ Generate Inventory (generate_inventory.py)
    ├─→ Generate Rebuild Order (generate_rebuild_order.py)
    ├─→ Verify Coherence (verify-inventory-rebuildorder.py)
    ├─→ Execute Rebuild (reset_hetzner.yml)
    │      ├─→ Batch 1 (5 serveurs) [parallel]
    │      ├─→ Batch 2 (5 serveurs) [parallel]
    │      ├─→ ...
    │      └─→ Batch 10 (2 serveurs) [parallel]
    └─→ Generate Report (phase1-report.sh)
```

### Caractéristiques

#### 47 serveurs rebuildables

- **Total serveurs:** 49
- **Serveurs rebuildables:** 47
- **Exclus:** `install-01`, `install-v3`

#### 10 batches

- **Batch size:** 5 serveurs (batch 10: 2 serveurs)
- **Traitement séquentiel des batches** (batch 1 → batch 2 → ... → batch 10)
- **Traitement parallèle dans chaque batch** (`throttle: 10`)

#### Rebuild massif optimisé

- **Parallélisation:** `throttle: 10` sur toutes les opérations
- **Opérations parallèles:**
  - Get server info
  - Detach volumes
  - Delete volumes
  - Rebuild servers
  - Poll status
  - Check SSH port

#### Aucune opération sur bastions

- ✅ `install-01` **jamais touché**
- ✅ `install-v3` **jamais touché**
- ✅ Vérifications dans tous les scripts
- ✅ Exclusion dans `rebuild_order_v3.json`

#### Tout en API Hetzner

- ✅ **100% API Hetzner** via `community.general.hcloud_*` modules
- ✅ **Aucun SSH** vers serveurs rebuildés
- ✅ **Aucun sshpass**
- ✅ **Aucun mot de passe root**

#### Jamais de SSH

- ✅ Pas de connexion SSH pendant le rebuild
- ✅ Vérification port 22 uniquement (pas de connexion)
- ✅ SSH sera déployé en Phase 2 (après rebuild)

#### Idempotence

- ✅ Tous les scripts sont idempotents
- ✅ Ré-exécution possible sans erreur
- ✅ Gestion des cas "déjà fait"

#### Logging complet

- ✅ Logs par batch : `/opt/keybuzz/logs/phase1/batch-{N}-complete.log`
- ✅ Rapport JSON : `/opt/keybuzz/reports/phase1/phase1-final.json`
- ✅ Rapport Markdown : `/opt/keybuzz/reports/phase1/phase1-final.md`

---

## Ordre exact d'exécution

### Étape 1 : Token Hetzner

**Script:** `scripts/setup-hetzner-token.sh`

**Actions:**
1. Vérifie que `HETZNER_API_TOKEN` est défini
2. Crée `/opt/keybuzz/credentials/hcloud.env` (chmod 600)
3. Configure `~/.config/hcloud/cli.toml`
4. Ajoute auto-loading dans `~/.bashrc`
5. Teste la connexion hcloud

**Commande:**
```bash
export HETZNER_API_TOKEN="your-token"
bash scripts/setup-hetzner-token.sh
```

**Validation:**
```bash
source /opt/keybuzz/credentials/hcloud.env
export HETZNER_API_TOKEN
hcloud server list | head -5
```

### Étape 2 : Renommage PostgreSQL

**Script:** `scripts/rename-postgres-api.py`

**Actions:**
1. Charge le token depuis env ou fichier
2. Récupère tous les serveurs via API (pagination)
3. Renomme `db-master-01` → `db-postgres-01`
4. Renomme `db-slave-01` → `db-postgres-02`
5. Renomme `db-slave-02` → `db-postgres-03`
6. Vérifie le résultat final

**Commande:**
```bash
python3 scripts/rename-postgres-api.py
```

**Validation:**
```bash
hcloud server list | grep db-postgres
# Doit afficher: db-postgres-01, db-postgres-02, db-postgres-03
```

### Étape 3 : Génération inventaire

**Script:** `scripts/generate_inventory.py`

**Actions:**
1. Lit `servers/servers_v3.tsv`
2. Génère `ansible/inventory/hosts.yml`
3. Crée 21 groupes Ansible
4. Configure IP privée, clé SSH, variables

**Commande:**
```bash
python3 scripts/generate_inventory.py > ansible/inventory/hosts.yml
```

**Validation:**
```bash
grep -c "ansible_host:" ansible/inventory/hosts.yml
# Doit afficher: 49
```

### Étape 4 : Génération rebuild_order

**Script:** `scripts/generate_rebuild_order.py`

**Actions:**
1. Lit `servers/servers_v3.tsv`
2. Exclut `install-01` et `install-v3`
3. Génère `servers/rebuild_order_v3.json`
4. Crée 10 batches avec volumes

**Commande:**
```bash
python3 scripts/generate_rebuild_order.py
```

**Validation:**
```bash
jq '.metadata.total_servers' servers/rebuild_order_v3.json
# Doit afficher: 47
```

### Étape 5 : Vérification cohérence

**Script:** `scripts/verify-inventory-rebuildorder.py`

**Actions:**
1. Charge `ansible/inventory/hosts.yml`
2. Charge `servers/rebuild_order_v3.json`
3. Vérifie 49 serveurs dans inventaire
4. Vérifie 47 serveurs dans rebuild_order
5. Vérifie 10 batches
6. Vérifie volumes valides
7. Vérifie exclusions

**Commande:**
```bash
python3 scripts/verify-inventory-rebuildorder.py
```

**Validation:**
```
✓✓✓ ALL VERIFICATIONS PASSED ✓✓✓
```

### Étape 6 : Exécution playbook

**Script:** `ansible/playbooks/reset_hetzner.yml`

**Actions (pour chaque batch):**
1. Get server info (parallèle, throttle: 10)
2. Build volumes list
3. Detach volumes (parallèle, throttle: 10)
4. Wait for detachment
5. Delete volumes (parallèle, throttle: 10)
6. Rebuild servers (parallèle, throttle: 10)
7. Wait for running status (parallèle, throttle: 10)
8. Wait for SSH port 22 (parallèle, throttle: 10)
9. Log batch completion

**Commande:**
```bash
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/reset_hetzner.yml
```

**Validation:**
- Tous les batches complétés
- Logs disponibles dans `/opt/keybuzz/logs/phase1/`

### Étape 7 : Génération rapport

**Script:** `scripts/phase1-report.sh`

**Actions:**
1. Récupère tous les serveurs via API (pagination)
2. Génère `/opt/keybuzz/reports/phase1/phase1-final.json`
3. Génère `/opt/keybuzz/reports/phase1/phase1-final.md`
4. Vérifie 47 serveurs "running"
5. Vérifie bastions intacts

**Commande:**
```bash
bash scripts/phase1-report.sh
```

**Validation:**
```bash
ls -lh /opt/keybuzz/reports/phase1/
# Doit contenir: phase1-final.json, phase1-final.md
```

### Orchestration complète

**Script:** `scripts/execute-phase1.sh`

Exécute automatiquement toutes les étapes ci-dessus dans l'ordre.

**Commande:**
```bash
export HETZNER_API_TOKEN="your-token"
bash scripts/execute-phase1.sh
```

---

## Fichiers produits

### Logs

#### Logs par batch

**Localisation:** `/opt/keybuzz/logs/phase1/batch-{N}-complete.log`

**Contenu:**
```
Batch {N} - COMPLETED
========================================
Timestamp: 2024-11-30T...
Servers: ["server1", "server2", ...]
Status: All servers rebuilt and SSH port 22 open
========================================
```

**Exemples:**
- `/opt/keybuzz/logs/phase1/batch-1-complete.log`
- `/opt/keybuzz/logs/phase1/batch-2-complete.log`
- ...
- `/opt/keybuzz/logs/phase1/batch-10-complete.log`

#### Logs du playbook

**Localisation:** Sortie standard de `ansible-playbook`

**Recommandation:** Rediriger vers un fichier:
```bash
ansible-playbook ... | tee /opt/keybuzz/logs/phase1/playbook-$(date +%Y%m%d_%H%M%S).log
```

### Rapports

#### Rapport JSON

**Localisation:** `/opt/keybuzz/reports/phase1/phase1-final.json`

**Structure:**
```json
{
  "timestamp": "2024-11-30T...",
  "summary": {
    "total_servers": 47,
    "running_servers": 47,
    "expected_rebuild": 47,
    "bastions": {
      "install_01": {...},
      "install_v3": {...}
    }
  },
  "servers": [...],
  "postgres_servers": [...]
}
```

**Utilisation:**
- Analyse programmatique
- Intégration avec outils de monitoring
- Référence pour phases suivantes

#### Rapport Markdown

**Localisation:** `/opt/keybuzz/reports/phase1/phase1-final.md`

**Contenu:**
- Summary avec statistiques
- Liste des serveurs PostgreSQL
- Statut des bastions
- Tableau de tous les serveurs rebuildés
- Informations sur rebuild_order
- Liens vers logs

**Utilisation:**
- Documentation pour l'équipe
- Rapport pour Linear
- Référence opérationnelle

---

## Fiche de validation pré-dry-run (PH1-08)

### Checklist de validation

Avant de lancer le dry-run (PH1-08) ou l'exécution réelle (PH1-09), vérifier que **tous** les points suivants sont validés :

- [ ] **Token présent et fonctionnel**
  ```bash
  source /opt/keybuzz/credentials/hcloud.env
  export HETZNER_API_TOKEN
  hcloud server list | head -5
  # Doit fonctionner sans erreur
  ```

- [ ] **servers_v3.tsv validé**
  ```bash
  wc -l servers/servers_v3.tsv
  # Doit afficher: 50 (1 header + 49 serveurs)
  grep -c "^prod" servers/servers_v3.tsv
  # Doit afficher: 49
  ```

- [ ] **rebuild_order_v3.json validé**
  ```bash
  jq '.metadata.total_servers' servers/rebuild_order_v3.json
  # Doit afficher: 47
  jq '.metadata.total_batches' servers/rebuild_order_v3.json
  # Doit afficher: 10
  jq '.metadata.excluded_servers' servers/rebuild_order_v3.json
  # Doit afficher: ["install-01", "install-v3"]
  ```

- [ ] **reset_hetzner.yml validé**
  ```bash
  ansible-playbook --syntax-check ansible/playbooks/reset_hetzner.yml
  # Doit afficher: playbook: ansible/playbooks/reset_hetzner.yml
  grep -c "throttle" ansible/playbooks/reset_hetzner.yml
  # Doit afficher: 8 (opérations parallèles)
  ```

- [ ] **Inventaire validé**
  ```bash
  grep -c "ansible_host:" ansible/inventory/hosts.yml
  # Doit afficher: 49
  grep "ansible_ssh_private_key_file" ansible/inventory/hosts.yml
  # Doit contenir: /root/.ssh/id_rsa_keybuzz_v3
  ```

- [ ] **Scripts PH1 validés**
  ```bash
  # Vérifier que tous les scripts existent et sont exécutables
  ls -lh scripts/setup-hetzner-token.sh
  ls -lh scripts/rename-postgres-api.py
  ls -lh scripts/generate_inventory.py
  ls -lh scripts/generate_rebuild_order.py
  ls -lh scripts/verify-inventory-rebuildorder.py
  ls -lh scripts/execute-phase1.sh
  ls -lh scripts/phase1-report.sh
  # Tous doivent être présents et exécutables
  ```

- [ ] **47 serveurs identifiés**
  ```bash
  python3 scripts/verify-inventory-rebuildorder.py
  # Doit afficher: ✓✓✓ ALL VERIFICATIONS PASSED ✓✓✓
  ```

- [ ] **2 bastions exclus**
  ```bash
  jq '.metadata.excluded_servers' servers/rebuild_order_v3.json
  # Doit afficher: ["install-01", "install-v3"]
  grep -c "install-01\|install-v3" servers/rebuild_order_v3.json
  # Ne doit pas apparaître dans la liste des serveurs à rebuild
  ```

- [ ] **logs directory OK**
  ```bash
  mkdir -p /opt/keybuzz/logs/phase1
  mkdir -p /opt/keybuzz/reports/phase1
  ls -ld /opt/keybuzz/logs/phase1
  ls -ld /opt/keybuzz/reports/phase1
  # Doivent exister avec permissions 755
  ```

- [ ] **ready for PH1-08 (dry-run)**
  - Toutes les validations ci-dessus passées
  - Token Hetzner configuré et testé
  - Tous les fichiers en place
  - Scripts validés et testés

### Commandes de validation rapide

```bash
# Script de validation complète
cd /opt/keybuzz/keybuzz-infra

# 1. Token
source /opt/keybuzz/credentials/hcloud.env && export HETZNER_API_TOKEN && hcloud server list | head -3

# 2. Fichiers
wc -l servers/servers_v3.tsv
jq '.metadata | {total_servers, total_batches, excluded_servers}' servers/rebuild_order_v3.json

# 3. Scripts
python3 scripts/verify-inventory-rebuildorder.py

# 4. Playbook
ansible-playbook --syntax-check ansible/playbooks/reset_hetzner.yml

# 5. Répertoires
mkdir -p /opt/keybuzz/{logs,reports}/phase1 && ls -ld /opt/keybuzz/{logs,reports}/phase1
```

---

## Schéma d'architecture PH1-09

### Flux d'exécution

```
┌─────────────────────────────────────────────────────────────┐
│                    install-v3 (bastion)                     │
│              (46.62.171.61 - 10.0.0.251)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ HETZNER API (100%)
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │   reset_hetzner.yml (Ansible)        │
        │   + rebuild_order_v3.json            │
        └──────────────┬───────────────────────┘
                       │
                       │ Loop: 10 batches (séquentiel)
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
┌───────────────┐           ┌──────────────────┐
│  BATCH 1      │           │    BATCH 2-10    │
│  (5 serveurs) │           │  (5 serveurs/ch) │
└───────┬───────┘           └────────┬─────────┘
        │                            │
        │ Pour chaque batch:         │
        │                            │
        ├─→ [PARALLEL] Get server info (throttle: 10)
        │                            │
        ├─→ [PARALLEL] Detach volumes (throttle: 10)
        │     │
        │     ├─→ k8s-master-01: detach volume
        │     ├─→ k8s-master-02: detach volume
        │     ├─→ k8s-master-03: detach volume
        │     ├─→ k8s-worker-01: detach volume
        │     └─→ k8s-worker-02: detach volume
        │
        ├─→ Wait 5s (propagation)
        │
        ├─→ [PARALLEL] Delete volumes (throttle: 10)
        │     │
        │     ├─→ k8s-master-01: delete volume
        │     ├─→ k8s-master-02: delete volume
        │     ├─→ k8s-master-03: delete volume
        │     ├─→ k8s-worker-01: delete volume
        │     └─→ k8s-worker-02: delete volume
        │
        ├─→ [PARALLEL] Rebuild servers (throttle: 10)
        │     │
        │     ├─→ k8s-master-01: rebuild Ubuntu 24.04
        │     ├─→ k8s-master-02: rebuild Ubuntu 24.04
        │     ├─→ k8s-master-03: rebuild Ubuntu 24.04
        │     ├─→ k8s-worker-01: rebuild Ubuntu 24.04
        │     └─→ k8s-worker-02: rebuild Ubuntu 24.04
        │
        ├─→ [PARALLEL] Poll until "running" (throttle: 10)
        │     │
        │     ├─→ k8s-master-01: wait for running
        │     ├─→ k8s-master-02: wait for running
        │     ├─→ k8s-master-03: wait for running
        │     ├─→ k8s-worker-01: wait for running
        │     └─→ k8s-worker-02: wait for running
        │
        └─→ [PARALLEL] Wait for SSH port 22 (throttle: 10)
              │
              ├─→ k8s-master-01: check port 22
              ├─→ k8s-master-02: check port 22
              ├─→ k8s-master-03: check port 22
              ├─→ k8s-worker-01: check port 22
              └─→ k8s-worker-02: check port 22

                           │
                           ▼
        ┌──────────────────────────────────────┐
        │   Log batch completion               │
        │   → /opt/keybuzz/logs/phase1/        │
        └──────────────────────────────────────┘

                           │
                           ▼
        ┌──────────────────────────────────────┐
        │   Generate Reports                   │
        │   → phase1-final.json                │
        │   → phase1-final.md                  │
        └──────────────────────────────────────┘
```

### Détails techniques

#### Parallélisation

```
Batch N (5 serveurs)
│
├─→ Opération 1: [5 serveurs en parallèle] throttle: 10
│   ├─→ server1 ─┐
│   ├─→ server2 ─┤
│   ├─→ server3 ─┼─→ API Hetzner (simultané)
│   ├─→ server4 ─┤
│   └─→ server5 ─┘
│
├─→ Wait (si nécessaire)
│
└─→ Opération 2: [5 serveurs en parallèle] throttle: 10
    ...
```

#### Séquence temporelle

```
T0:   Début batch 1
T0+2: Get info (parallèle, ~2s)
T0+5: Detach volumes (parallèle, ~3s)
T0+10: Wait propagation (5s)
T0+12: Delete volumes (parallèle, ~2s)
T0+72: Rebuild servers (parallèle, ~60s)
T0+672: Wait running (parallèle, ~600s max)
T0+792: Wait SSH port (parallèle, ~120s max)
T0+792: Batch 1 complet

T0+792: Début batch 2
...
```

**Temps total estimé:** ~60-90 minutes pour 47 serveurs

---

## Risques et mitigations

### 1. Rate limits Hetzner

**Risque:** L'API Hetzner a une limite de ~3600 requêtes/heure

**Mitigation:**
- ✅ **Throttle: 10** limite le nombre de requêtes simultanées
- ✅ **Batching** étale les requêtes dans le temps
- ✅ **Retry logic** dans Ansible (retries: 60, delay: 10s)
- ✅ **Estimation:** ~500-1000 requêtes pour 47 serveurs (marge 3-6x)

**Action si limite atteinte:**
- Vérifier les logs pour identifier la source
- Augmenter les délais entre batches si nécessaire
- Contacter Hetzner support si problème persistant

### 2. Serveurs "stuck in rebuild"

**Risque:** Un serveur peut rester bloqué dans un état intermédiaire

**Mitigation:**
- ✅ **Polling exhaustif:** retries: 60, delay: 10s (max 10 minutes)
- ✅ **Vérification manuelle possible:** `hcloud server describe <name>`
- ✅ **Action manuelle:** Rebuild manuel via console Hetzner si nécessaire

**Action si stuck:**
```bash
# Vérifier le statut
hcloud server describe <server-name> -o json | jq '.status'

# Rebuild manuel si nécessaire
hcloud server rebuild <server-id> --image ubuntu-24.04
```

### 3. Retry global

**Risque:** Échec temporaire d'une opération

**Mitigation:**
- ✅ **Ansible retry logic:** retries + delay sur toutes les opérations critiques
- ✅ **Idempotence:** Ré-exécution possible sans problème
- ✅ **Failed when false:** Sur suppression volumes (déjà supprimés = OK)

**Action si échec:**
- Vérifier les logs pour identifier l'erreur
- Ré-exécuter le batch spécifique si nécessaire
- Le playbook peut être ré-exécuté (idempotent)

### 4. Idempotence

**Risque:** Ré-exécution accidentelle du playbook

**Mitigation:**
- ✅ **Tous les scripts sont idempotents**
- ✅ **Rebuild serveur:** Idempotent (même état = pas de changement)
- ✅ **Suppression volumes:** `failed_when: false` (déjà supprimés = OK)

**Action si ré-exécution:**
- Le playbook peut être ré-exécuté sans problème
- Les serveurs déjà rebuildés seront vérifiés mais non modifiés
- Les volumes déjà supprimés seront ignorés

### 5. Logs batch

**Risque:** Perte de traçabilité si logs non générés

**Mitigation:**
- ✅ **Log par batch:** `/opt/keybuzz/logs/phase1/batch-{N}-complete.log`
- ✅ **Logs automatiques:** Générés par le playbook
- ✅ **Rapports:** JSON et Markdown générés automatiquement

**Action si logs manquants:**
- Vérifier les permissions sur `/opt/keybuzz/logs/phase1/`
- Vérifier que le playbook a bien créé les logs
- Ré-exécuter `phase1-report.sh` pour régénérer les rapports

### 6. Exclusions bastions

**Risque:** Accidental rebuild de `install-01` ou `install-v3`

**Mitigation:**
- ✅ **Exclusion dans `rebuild_order_v3.json`:** `"excluded_servers": ["install-01", "install-v3"]`
- ✅ **Vérifications dans scripts:** Tous les scripts excluent ces serveurs
- ✅ **Playbook utilise rebuild_order:** Ne lit que les serveurs listés

**Action de vérification:**
```bash
# Vérifier que les bastions ne sont pas dans rebuild_order
jq '.servers[] | select(.hostname == "install-01" or .hostname == "install-v3")' servers/rebuild_order_v3.json
# Ne doit retourner aucun résultat
```

### 7. Pas de création automatique

**Risque:** Création accidentelle de nouveaux serveurs

**Mitigation:**
- ✅ **Playbook utilise uniquement `rebuild`:** Pas de `create`
- ✅ **Liste fermée:** Seulement les serveurs dans `rebuild_order_v3.json`
- ✅ **Vérification avant rebuild:** Get server info avant opération

**Action préventive:**
- Vérifier que `rebuild_order_v3.json` contient uniquement les serveurs existants
- Le playbook échouera si un serveur n'existe pas (sécurité)

### 8. Volume data loss

**Risque:** Perte de données lors de la suppression des volumes

**Mitigation:**
- ⚠️ **CRITIQUE:** Les volumes sont **définitivement supprimés** avant rebuild
- ✅ **Backup recommandé:** Sauvegarder les données critiques avant Phase 1
- ✅ **Documentation:** Ce runbook documente clairement ce risque

**Action préventive:**
- **SAUVEGARDER TOUTES LES DONNÉES CRITIQUES** avant Phase 1
- Vérifier que les données importantes sont sauvegardées
- La Phase 1 est un **point de non-retour** pour les volumes

---

## Notes SRE

### Monitoring pendant l'exécution

**Commandes utiles:**

```bash
# Vérifier le statut des serveurs en temps réel
watch -n 10 'hcloud server list | grep -E "STATUS|running|rebuilding"'

# Compter les serveurs en cours de rebuild
hcloud server list --output json | jq '[.[] | select(.status == "rebuilding")] | length'

# Vérifier les logs en temps réel
tail -f /opt/keybuzz/logs/phase1/batch-*-complete.log
```

### Rollback

**En cas de problème majeur:**

1. **Arrêter le playbook** (Ctrl+C)
2. **Vérifier l'état** des serveurs déjà rebuildés
3. **Décider** de continuer ou rollback
4. **Rollback manuel** si nécessaire:
   - Rebuild individuel des serveurs problématiques
   - Restauration depuis backup si disponible

**Note:** Il n'y a **pas de rollback automatique**. Les volumes supprimés sont perdus.

### Temps d'exécution

**Estimations:**
- **Batch 1-9:** ~12 minutes chacun (5 serveurs)
- **Batch 10:** ~8 minutes (2 serveurs)
- **Total:** ~60-90 minutes pour 47 serveurs

**Facteurs influençant:**
- Vitesse de l'API Hetzner
- Taille des volumes à supprimer
- Charge système Hetzner

### Points de validation

**Après chaque batch:**
1. Vérifier les logs du batch
2. Vérifier que tous les serveurs sont "running"
3. Vérifier que les bastions sont intacts

**Après Phase 1 complète:**
1. Générer le rapport final
2. Vérifier 47 serveurs "running"
3. Vérifier bastions intacts
4. Documenter dans Linear

---

## Conclusion

La Phase 1 est une étape **critique** qui prépare l'infrastructure KeyBuzz v3 pour les phases suivantes.

**Avant de lancer:**
- ✅ Valider tous les prérequis (checklist PH1-08)
- ✅ Sauvegarder les données critiques
- ✅ Vérifier que tous les scripts sont à jour
- ✅ Avoir le token Hetzner prêt

**Pendant l'exécution:**
- ✅ Monitorer les logs
- ✅ Vérifier le statut des serveurs
- ✅ Documenter tout problème

**Après l'exécution:**
- ✅ Générer le rapport final
- ✅ Valider que 47 serveurs sont "running"
- ✅ Documenter dans Linear
- ✅ Proceed to Phase 2 (SSH mesh)

**Le système est maintenant prêt pour PH1-08 (dry-run) et PH1-09 (exécution réelle).**

---

**Document généré le:** 2024-11-30  
**Dernière mise à jour:** 2024-11-30  
**Version:** 1.0


#  HAPI FHIR Observabilité — Projet DevOps ENSIAS 2026

> **Étude expérimentale de l'impact des niveaux d'observabilité sur la détection des incidents dans un système de santé (HAPI FHIR).**  
> *Projet réalisé dans le cadre de la Fiche 3 : Observabilité & DevOps (Health) — ENSIAS 2026.*

---

##  Équipe & Contributeurs

| Membre | Rôle & Contributions |
| :--- | :--- |
| **Hajar BRAIDI** | Infrastructure Docker, Automatisation des tests & Scripts PowerShell |
| **Aya TAKI** | Observabilité (Prometheus, Loki, Tempo) & Dashboards Grafana |
| **Soukaina SABBAR** | Chaos Engineering, Simulation des incidents & Mesures |
| **Khadija AIT OUFKIR** | Analyse de données, Graphiques Python & Rédaction du rapport |

* **Enseignante encadrante :** Prof. Karima MOUMANE

---

##  Description du Projet

Ce projet mesure de manière expérimentale l'impact de l'observabilité sur le **MTTD** (*Mean Time To Detect*) et le **MTTR** (*Mean Time To Resolve*) lors d'incidents appliqués à un serveur de santé **HAPI FHIR** sous la base de données PostgreSQL.

###  Les 4 Niveaux d'Observabilité Étudiés

| Niveau | Nom | Stack Déployée |
| :---: | :--- | :--- |
| **N0** | Sans monitoring | HAPI FHIR + PostgreSQL uniquement |
| **N1** | Logs | N0 + Loki + Promtail + Grafana |
| **N2** | Logs + Métriques | N1 + Prometheus |
| **N3** | Observabilité complète | N2 + Tempo (Traces distribuées OTLP) |

###  Incidents Simulés

1. **Mass HTTP 500 :** 100 requêtes FHIR invalides envoyées en rafale.
2. **High Latency :** Conteneur HAPI gelé temporairement via `docker pause` (SIGSTOP).
3. **Disk Full :** Saturation du volume PostgreSQL avec un fichier temporaire de 500 Mo.

---

##  Architecture System

```text
+-------------------------------------------------------------------------+
|                              Docker Compose                             |
|                                                                         |
|  +--------------------+      OTLP       +----------------------------+  |
|  |     HAPI FHIR      |---------------->| Tempo (:3200 / OTLP :4317) |  |
|  |      (:8080)       |                 +----------------------------+  |
|  +--------------------+                               |                 |
|       |          | Metrics (/actuator/prometheus)     |                 |
|  Logs |          v                                    |                 |
|       |    +------------+                             |                 |
|       |    | Prometheus |-------------------+         |                 |
|       |    |  (:9091)   |                   |         |                 |
|       v    +------------+                   v         v                 |
|  +----------+    |                    +----------------------------+    |
|  | Promtail |----> Loki (:3100) ----->|      Grafana (:3000)       |    |
|  +----------+                         +----------------------------+    |
|                                                       ^                 |
|  +--------------------+                               |                 |
|  | PostgreSQL (:5432) |-------------------------------+                 |
|  +--------------------+                                                 |
+-------------------------------------------------------------------------+

```

---

##  Lancement Rapide

### Prerequisites

* Docker Desktop & Docker Compose
* Git
* Python 3.x (avec `matplotlib` et `numpy`)
* PowerShell (sous Windows)

### Installation

```bash
git clone [https://github.com/HajarBraidi/Devops-hapi-fhir-jpaserver.git](https://github.com/HajarBraidi/Devops-hapi-fhir-jpaserver.git)
cd Devops-hapi-fhir-jpaserver

```

### Démarrage par niveau

```bash
# Niveau 0 — Sans monitoring
docker compose up hapi-fhir-jpaserver-start hapi-fhir-postgres -d

# Niveau 1 — Logs uniquement
docker compose --profile logs up -d

# Niveau 2 — Logs + Métriques
docker compose --profile logs --profile metrics up -d

# Niveau 3 — Observabilité complète
docker compose --profile logs --profile metrics --profile traces up -d

```

---

##  Accès aux Interfaces

| Service | URL | Identifiants | Niveau requis |
| --- | --- | --- | --- |
| **HAPI FHIR** | `http://localhost:8080/fhir/metadata` | — | N0+ |
| **Grafana** | `http://localhost:3000` | `admin` / `admin` | N1+ |
| **Prometheus** | `http://localhost:9091` | — | N2+ |
| **Loki** | `http://localhost:3100` | — | N1+ |
| **Tempo** | `http://localhost:3200` | — | N3 |

---

##  Execution des Expériences & Automatisation

Le projet intègre un script PowerShell d'automatisation (`experiments/run_experiments.ps1`) qui exécute la matrice complète des 24 tests ($4 \text{ niveaux} \times 6 \text{ itérations/incidents}$) et enregistre les horodatages précis ($T_0, T_1, T_2$).

```powershell
# Exécution du script de benchmark automatisé
.\experiments\run_experiments.ps1

```

### Calculs d'Observabilité ($MTTD$ & $MTTR$)

$$\text{MTTD (Mean Time To Detect)} = T_1 - T_0$$

$$\text{MTTR (Mean Time To Resolve)} = T_2 - T_0$$

* $T_0$ : Instant exact de l'injection de l'incident.
* $T_1$ : Horodatage de la première détection (échec HTTP / alerte log/métrique).
* $T_2$ : Horodatage du retour à l'état nominal (*Health check UP*).

---

##  Résultats Expérimentaux (Synthese MTTD)

Voici les données moyennes mesurées et extraites lors de la campagne d'essais (`results.csv`) :

| Incident | N0 (Sans monitoring) | N1 (Logs) | N2 (Logs + Métriques) | N3 (Observabilité complète) |
| --- | --- | --- | --- | --- |
| **Mass HTTP 500** | 0.706 s | 0.641 s | 1.321 s | 0.741 s |
| **High Latency** | 0.342 s | 0.187 s | 0.460 s | 0.301 s |
| **Disk Full** | 12.239 s | 7.478 s | 7.160 s | 5.367 s |
| **MTTD Moyen Global** | **4.429 s** | **2.769 s** | **2.980 s** | **2.136 s** |

### Génération des graphiques d'analyse

```bash
python experiments/analyze_results.py results.csv

```

---

##  Structure du Projet

```text
Devops-hapi-fhir-jpaserver/
├── docker-compose.yml           # Orchestration avec profils (logs, metrics, traces)
├── config/
│   ├── grafana-datasources.yml  # Data sources automatisées Grafana
│   ├── prometheus.yml           # Configuration de scrape Actuator
│   ├── loki-config.yml          # Configuration de rétention Loki
│   ├── promtail-config.yml      # Collecte des logs Docker
│   └── tempo-config.yml         # Réception des traces OTLP/gRPC
├── experiments/
│   ├── run_experiments.ps1     # Automation des expériences N0 -> N3
│   ├── analyze_results.py      # Génération des graphiques d'analyse
│   └── dashboard.html          # Visualisation web des résultats
└── results.csv                 # Dataset des mesures brutes (T0, T1, T2, MTTD, MTTR)

```

---

##  Références & Documentation

* **HAPI FHIR JPA Server :** [hapifhir.io](https://hapifhir.io/)
* **Prometheus Documentation :** [prometheus.io/docs](https://prometheus.io/docs/)
* **Grafana Stack (Loki / Tempo) :** [grafana.com/docs](https://grafana.com/docs/)

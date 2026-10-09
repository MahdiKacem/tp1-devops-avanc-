# Rapport du TP GitOps avec Argo CD

**Date de réalisation :** 9 octobre 2026  
**Sujet :** Étape 1 — Structure du dépôt Git pour une démarche GitOps  
**Application choisie :** NGINX  
**Outil de templating :** Kustomize  
**État d'avancement :** Étape 1 terminée, dépôt prêt à être publié et référencé par Argo CD

## 1. Objectif du TP

L'objectif est de manipuler les principes fondamentaux de GitOps autour
d'Argo CD. La première étape consiste à créer un dépôt Git contenant des
manifests Kubernetes déclaratifs et structurés.

Le choix s'est porté sur une application NGINX minimale. Ce choix permet de
se concentrer sur le pipeline GitOps et la structure des manifests plutôt que
sur la complexité fonctionnelle de l'application.

## 2. Prérequis et environnement

### Environnement utilisé

- Système : Windows
- Répertoire de travail :
  `C:\Users\mahdi\OneDrive\Bureau\Kraya\DevOps Avancé\TP1`
- Contrôle de version : Git
- Outil Kubernetes : `kubectl`
- Version du client Kubernetes relevée : `v1.36.1`
- Repository Git local : branche `master`

### Vérification initiale

Le dossier de travail était vide avant la réalisation de cette étape. Aucun
fichier existant n'a donc été modifié ou supprimé.

## 3. Choix de l'architecture déclarative

Kustomize a été retenu plutôt que des fichiers YAML indépendants ou un chart
Helm.

### Raisons du choix

- Kustomize est intégré à `kubectl`.
- Les ressources communes peuvent être placées dans une `base`.
- Les différences entre environnements peuvent être décrites dans des
  `overlays`.
- Les manifests restent lisibles et ne nécessitent pas de moteur de
  templating supplémentaire.
- Argo CD sait directement synchroniser un chemin Kustomize dans un dépôt Git.

## 4. Structure du dépôt

La structure obtenue est la suivante :

```text
TP1/
├── .gitignore
├── README.md
├── RAPPORT.md
├── base/
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

### Rôle des fichiers

| Fichier ou dossier | Rôle |
|---|---|
| `base/kustomization.yaml` | Déclare les ressources communes |
| `base/namespace.yaml` | Crée le namespace `gitops-demo` |
| `base/deployment.yaml` | Déploie le conteneur `nginx:1.27-alpine` |
| `base/service.yaml` | Expose NGINX avec un service `ClusterIP` |
| `overlays/dev/` | Configuration de développement |
| `overlays/prod/` | Configuration de production |
| `replica-patch.yaml` | Modifie le nombre de répliques par environnement |
| `README.md` | Explique l'utilisation et la validation locale |
| `RAPPORT.md` | Présente les étapes, résultats, logs et analyses du TP |

## 5. Ressources Kubernetes créées

### Namespace

Un namespace dédié nommé `gitops-demo` est défini afin d'isoler les ressources
de l'application.

### Deployment

Le Deployment `gitops-demo` utilise :

- l'image `nginx:1.27-alpine` ;
- le port HTTP `80` ;
- un label `app.kubernetes.io/name: gitops-demo` ;
- une `readinessProbe` HTTP sur `/` ;
- une `livenessProbe` HTTP sur `/`.

Les probes permettent à Kubernetes de vérifier que le conteneur est prêt à
recevoir du trafic et qu'il reste fonctionnel.

### Service

Le Service `gitops-demo` :

- est de type `ClusterIP` ;
- écoute sur le port `80` ;
- cible le port nommé `http` du Deployment ;
- sélectionne les pods grâce au label de l'application.

## 6. Gestion des environnements

Les deux overlays réutilisent la base commune :

```yaml
resources:
  - ../../base
```

Chaque overlay applique ensuite un suffixe de nom et un patch spécifique.

| Environnement | Suffixe | Répliques | Image |
|---|---:|---:|---|
| Dev | `-dev` | 1 | `nginx:1.27-alpine` |
| Prod | `-prod` | 2 | `nginx:1.27-alpine` |

Cette séparation permet de modifier la capacité de production sans dupliquer
les manifests communs.

## 7. Déroulement des étapes

### Étape 1 — Inspection du dossier

Le contenu du dossier a été inspecté avant toute modification. Le dossier ne
contenait aucun fichier.

### Étape 2 — Création des manifests

Les fichiers suivants ont été créés :

- `.gitignore`
- `README.md`
- les ressources de `base/`
- les overlays `dev` et `prod`

### Étape 3 — Initialisation du dépôt Git

Commande exécutée :

```powershell
git init
```

Résultat :

```text
Initialized empty Git repository in
C:/Users/mahdi/OneDrive/Bureau/Kraya/DevOps Avancé/TP1/.git/
```

### Étape 4 — Première validation Kustomize

Commandes exécutées :

```powershell
kubectl kustomize overlays\dev
kubectl kustomize overlays\prod
```

### Étape 5 — Erreur rencontrée

La première validation a échoué pour les deux overlays avec le message
suivant :

```text
error: no resource matches strategic merge patch
"Deployment.v1.apps/gitops-demo.[noNs]": no matches for Id
Deployment.v1.apps/gitops-demo.[noNs]; failed to find unique target for patch
Deployment.v1.apps/gitops-demo.[noNs]
```

### Analyse de l'erreur

Le patch ciblait une `Deployment` nommée `gitops-demo`, mais ne précisait pas
son namespace. La ressource réelle de la base appartient au namespace
`gitops-demo`. Kustomize ne pouvait donc pas identifier de manière unique la
cible du patch.

### Correction appliquée

Le champ suivant a été ajouté dans les deux patches :

```yaml
metadata:
  name: gitops-demo
  namespace: gitops-demo
```

Cette correction rend l'identité complète de la ressource cohérente avec la
ressource déclarée dans la base.

### Étape 6 — Validation après correction

Commandes exécutées :

```powershell
kubectl kustomize overlays\dev | Out-Null
kubectl kustomize overlays\prod | Out-Null
```

Logs obtenus :

```text
dev: OK
prod: OK
```

Une vérification ciblée du rendu a ensuite confirmé les différences entre les
environnements :

```text
name: gitops-demo-dev
replicas: 1

name: gitops-demo-prod
replicas: 2
```

## 8. Versionnement Git

Les manifests ont été ajoutés puis enregistrés dans Git avec la commande :

```powershell
git add .gitignore README.md base overlays
git commit -m "feat: add kustomize gitops manifests"
```

Commit créé :

```text
c95b28f (HEAD -> master) feat: add kustomize gitops manifests
```

Le dépôt était propre après ce commit.

## 9. Résultats obtenus

Les objectifs de l'étape 1 sont atteints :

- un dépôt Git local a été initialisé ;
- une application Kubernetes fonctionnelle a été décrite ;
- les manifests utilisent une approche déclarative Kustomize ;
- une base commune et deux environnements ont été créés ;
- les overlays ont été rendus avec succès par Kustomize ;
- une différence de configuration entre dev et prod est démontrée ;
- la structure est adaptée à une future synchronisation Argo CD.

## 10. Analyse GitOps

La structure mise en place suit le principe GitOps suivant :

1. La configuration désirée est stockée dans Git.
2. Les manifests sont déclaratifs et reproductibles.
3. Les différences d'environnement sont explicites et traçables.
4. Un changement de configuration pourra être effectué par commit.
5. Argo CD pourra comparer l'état Git à l'état réel du cluster.
6. Argo CD pourra ensuite appliquer automatiquement les changements autorisés.

L'utilisation d'une base et d'overlays évite la duplication de fichiers et
facilite les évolutions futures, par exemple :

- changer la version de l'image ;
- augmenter les ressources CPU ou mémoire en production ;
- ajouter une Ingress ;
- ajouter des ConfigMaps ou Secrets ;
- créer une `Application` Argo CD différente pour chaque environnement.

## 11. Limites et suite prévue

Cette étape porte uniquement sur la structure du dépôt Git. Les éléments
suivants ne sont pas encore réalisés :

- publication du dépôt sur GitHub ou GitLab ;
- installation d'Argo CD dans un cluster Kubernetes ;
- création de la ressource Argo CD `Application` ;
- synchronisation automatique ;
- démonstration d'une modification détectée et appliquée par Argo CD ;
- observation de l'état `Synced` et `Healthy`.

La suite logique consiste à publier ce dépôt, puis à configurer Argo CD avec
le chemin `overlays/dev` ou `overlays/prod` comme source de l'application.

## 12. Conclusion

La première étape du TP est validée. Le dépôt possède une structure claire,
déclarative et adaptée à GitOps. L'erreur Kustomize rencontrée lors de la
validation a été identifiée, expliquée et corrigée. Les deux configurations
d'environnement sont maintenant valides et prêtes à être consommées par
Argo CD.

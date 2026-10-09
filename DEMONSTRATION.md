# Démonstration Argo CD — résultats et logs

**Date :** 9 octobre 2026  
**Cluster :** Kind `gitops`  
**Application Argo CD :** `gitops-demo-dev`  
**Dépôt :** `https://github.com/MahdiKacem/tp1-devops-avanc-.git`  
**Chemin synchronisé :** `overlays/dev`

Ce document sert de support pour la démonstration depuis le tableau de bord
Argo CD et depuis un terminal PowerShell.

## 1. Préparation de l'environnement

Dans chaque terminal PowerShell :

```powershell
$KUBECONFIG = "$env:TEMP\tp1-kind-gitops.yaml"
```

Le fichier kubeconfig est généré depuis le contexte Kind disponible dans WSL.

### Ouverture de l'interface Argo CD

Dans un terminal dédié :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n argocd port-forward svc/argocd-server 8080:443
```

Ouvrir ensuite :

```text
https://localhost:8080
```

Connexion :

```text
Utilisateur : admin
Mot de passe : valeur du secret argocd-initial-admin-secret
```

Pour récupérer le mot de passe :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n argocd get secret argocd-initial-admin-secret `
  -o jsonpath="{.data.password}" |
  ForEach-Object {
    [System.Text.Encoding]::UTF8.GetString(
      [System.Convert]::FromBase64String($_)
    )
  }
```

## 2. État de référence vérifié

Commandes exécutées :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n argocd get application gitops-demo-dev -o wide

kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get deployment,pods,service
```

Logs observés :

```text
NAME              SYNC STATUS   HEALTH STATUS   REVISION                                   PROJECT
gitops-demo-dev   Synced        Healthy         05b6fbe9ebcbb2c0ce108d9ec045cb52cac5458e   default
```

```text
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gitops-demo-dev   1/1     1            1            ...

NAME                                   READY   STATUS    RESTARTS   AGE
pod/gitops-demo-dev-...                1/1     Running   ...        ...

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/gitops-demo-dev   ClusterIP   ...             <none>        80/TCP
```

### Analyse

L'état initial est conforme :

- **Sync Status `Synced`** : les ressources du cluster correspondent à Git ;
- **Health Status `Healthy`** : les ressources Kubernetes sont opérationnelles ;
- le Deployment possède une seule réplique, conformément à l'overlay `dev`.

Dans l'interface, la carte `gitops-demo-dev` doit donc afficher
`Synced` et `Healthy`.

## 3. Scénario 1 — Déploiement continu automatisé

### Modification à effectuer dans Git

Dans [overlays/dev/replica-patch.yaml](overlays/dev/replica-patch.yaml),
remplacer :

```yaml
spec:
  replicas: 1
```

par :

```yaml
spec:
  replicas: 2
```

Valider et publier :

```powershell
kubectl kustomize overlays\dev | Select-String "replicas:"
git add overlays\dev\replica-patch.yaml
git commit -m "demo: scale dev application to two replicas"
git push origin main
```

### Observation dans le tableau de bord

Dans Argo CD :

1. Ouvrir `gitops-demo-dev`.
2. Cliquer sur **Refresh** si nécessaire.
3. Observer la détection du nouveau commit.
4. Observer la synchronisation automatique.
5. Vérifier le retour à `Synced` et `Healthy`.
6. Ouvrir l'arbre des ressources et vérifier les deux pods.

### Vérification terminal

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get deployment,pods
```

Résultat attendu :

```text
deployment.apps/gitops-demo-dev   2/2   2   2
pod/gitops-demo-dev-...            1/1   Running
pod/gitops-demo-dev-...            1/1   Running
```

### Conclusion à présenter

La modification a été faite uniquement dans Git. Argo CD l'a détectée et a
déployé automatiquement la deuxième réplique, sans `kubectl apply`.

## 4. Scénario 2 — Drift et Self-Healing

Ce scénario ne modifie pas Git. Il modifie directement l'état réel du cluster.

### Provoquer un drift

Si le scénario 1 a été exécuté, la valeur désirée est `2` :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo scale deployment/gitops-demo-dev --replicas=4
```

Vérifier le drift :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get deployment gitops-demo-dev
```

Dans Argo CD, cliquer sur **Refresh** et montrer temporairement
`OutOfSync`.

### Résultat observé précédemment

Une première vérification a utilisé une dérive vers trois répliques :

```text
deployment.apps/gitops-demo-dev scaled

NAME              DESIRED   READY
gitops-demo-dev   1         1

NAME              SYNC STATUS   HEALTH STATUS
gitops-demo-dev   Synced        Healthy
```

La valeur `DESIRED` était déjà revenue à `1` lorsque la commande de contrôle
a été exécutée. Cela confirme la réconciliation automatique par
`selfHeal: true`.

### Variante plus visuelle : supprimer le Deployment

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo delete deployment/gitops-demo-dev
```

Surveiller la recréation :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get deployment,pods --watch
```

Résultat attendu après réconciliation :

```text
deployment.apps/gitops-demo-dev   ...   2/2   2   2
```

Dans Argo CD, l'état revient à :

```text
Sync Status   : Synced
Health Status : Healthy
```

### Analyse

Le cluster a été modifié hors Git. Argo CD a comparé l'état réel à l'état
désiré, détecté le drift, puis réappliqué automatiquement le Deployment.

## 5. Scénario 3 — Sync versus Health

### Introduire une erreur applicative

Dans [base/deployment.yaml](base/deployment.yaml), remplacer temporairement :

```yaml
image: nginx:1.27-alpine
```

par :

```yaml
image: nginx:version-inexistante
```

Valider et publier :

```powershell
kubectl kustomize overlays\dev | Select-String "image:"
git add base\deployment.yaml
git commit -m "demo: introduce invalid container image"
git push origin main
```

### Résultat attendu dans Argo CD

```text
Sync Status   : Synced
Health Status : Degraded
```

### Vérification terminal

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get pods

kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo describe pod -l app.kubernetes.io/name=gitops-demo

kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get events --sort-by=.lastTimestamp
```

Événements attendus :

```text
ErrImagePull
ImagePullBackOff
```

### Analyse

`Synced` et `Healthy` sont deux informations différentes :

- `Synced` indique que le cluster correspond à la déclaration Git ;
- `Degraded` indique que la ressource déployée ne fonctionne pas ;
- le Deployment est correctement appliqué, mais le pod ne peut pas télécharger
  l'image inexistante.

## 6. Scénario 4 — Historique et rollback

### Ouvrir l'historique

Dans le tableau de bord :

1. Ouvrir `gitops-demo-dev`.
2. Ouvrir **History and Rollback**.
3. Repérer la révision stable avant l'image invalide.
4. Vérifier le commit associé.

### Effectuer le rollback

Si l'Automated Sync empêche le maintien du rollback, le désactiver
temporairement dans les paramètres de l'application, puis :

1. Sélectionner la dernière révision saine.
2. Cliquer sur **Rollback**.
3. Confirmer.
4. Vérifier le retour à l'image `nginx:1.27-alpine`.
5. Vérifier le retour des pods à `Running`.

Commandes de contrôle :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get pods

kubectl --kubeconfig $KUBECONFIG `
  -n argocd get application gitops-demo-dev
```

Résultat attendu :

```text
gitops-demo-dev   Synced   Healthy
```

### Correction Git obligatoire

Un rollback de l'état du cluster ne remplace pas la correction dans Git.
Restaurer l'image valide dans [base/deployment.yaml](base/deployment.yaml) :

```yaml
image: nginx:1.27-alpine
```

Puis publier :

```powershell
git add base\deployment.yaml
git commit -m "fix: restore valid nginx image"
git push origin main
```

Réactiver ensuite l'Automated Sync.

## 7. Résumé des preuves à montrer

| Fonctionnalité | Action | Preuve terminal | Preuve UI |
|---|---|---|---|
| Continuous Deployment | Modifier le nombre de répliques dans Git | `get deployment` affiche la nouvelle valeur | Synchronisation automatique |
| Drift | Modifier les répliques avec `kubectl scale` | La valeur revient à celle de Git | `OutOfSync`, puis `Synced` |
| Self-Healing | Supprimer le Deployment | Le Deployment est recréé | Ressource `Missing`, puis disponible |
| Erreur applicative | Utiliser une image inexistante | `ErrImagePull`, `ImagePullBackOff` | `Synced`, mais `Degraded` |
| Rollback | Restaurer une révision stable | Pods `Running` | Historique et rollback visibles |

## 8. État final de référence

Après la démonstration, le dépôt et le cluster doivent revenir à une
configuration saine :

```text
Image       : nginx:1.27-alpine
Répliques   : valeur de l'overlay dev
Sync Status : Synced
Health      : Healthy
```

Vérification finale :

```powershell
kubectl --kubeconfig $KUBECONFIG `
  -n argocd get application gitops-demo-dev

kubectl --kubeconfig $KUBECONFIG `
  -n gitops-demo get deployment,pods
```

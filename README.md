# TP GitOps avec Argo CD

Ce dépôt contient les manifests Kubernetes déclaratifs de l'application
`gitops-demo`, une application NGINX volontairement simple. La structure
utilise [Kustomize](https://kustomize.io/) afin de séparer les ressources
communes des configurations propres à chaque environnement.

## Structure

```text
.
├── base/
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── service.yaml
├── argocd/
│   └── application-dev.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

- `base/` contient les ressources communes à tous les environnements.
- `overlays/dev/` et `overlays/prod/` composent la base et appliquent leurs
  propres paramètres.
- Argo CD pourra ensuite référencer directement
  `overlays/dev` ou `overlays/prod` comme source Git.
- `argocd/application-dev.yaml` déclare l'Application Argo CD de
  l'environnement de développement, avec synchronisation automatique et
  self-healing.

## Validation locale

Avec `kubectl` installé, les manifests peuvent être rendus sans contacter un
cluster :

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
```

Pour appliquer l'environnement de développement :

```bash
kubectl apply -k overlays/dev
```

## GitOps : quatre principes

<div class="left">

1. **Déclaratif** : tout le système est décrit par des fichiers, pas par des commandes.
2. **Versionné** : l'état désiré vit dans git, avec son historique.
3. **Tiré automatiquement** : un agent dans le cluster va chercher l'état désiré.
4. **Réconcilié en continu** : l'agent corrige tout écart, sans qu'on le lui demande.

</div>

<p class="lead">C'est la boucle de réconciliation, étendue jusqu'à git.</p>

| Avant | Avec GitOps |
|---|---|
| « Qu'est-ce qui tourne en prod ? » : on va voir | on lit git |
| Déployer : lancer une commande | merger une pull request |
| Revenir en arrière : une procédure | `git revert` |
| Qui a changé quoi : on cherche | `git log` |

---

## Push ou pull

<div class="cols">
<div>

#### Push : le pipeline déploie

```text
git ──► pipeline ──kubectl──► cluster
```

- le pipeline détient les clés du cluster ;
- le cluster doit être joignable depuis le pipeline ;
- une modification manuelle n'est jamais détectée.

</div>
<div>

#### Pull : le cluster se déploie

```text
git ◄──lit── ArgoCD (dans le cluster)
```

- aucune clé du cluster ne sort du cluster ;
- le cluster peut être derrière un firewall ;
- tout écart est détecté et corrigé.

</div>
</div>

<div class="valid">
C'est pour cela que votre cluster, sur votre portable, peut être piloté depuis GitHub : c'est lui qui va lire le dépôt, jamais l'inverse.
</div>

---

## Kustomize : une base, des variantes

Deux environnements, presque identiques. On ne copie pas les fichiers : on décrit la différence.

```text
deploy/
├── base/                    ce qui est commun
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml       ce qui change en dev
    └── prod/kustomization.yaml      ce qui change en prod
```

```yaml
# deploy/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

<p class="small">Kustomize est intégré à kubectl : <code>kubectl kustomize deploy/overlays/dev</code> affiche le résultat, <code>kubectl apply -k</code> l'applique.</p>

---

## Un overlay

```yaml
# deploy/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod              # tout part dans ce namespace
resources:
  - ../../base
images:
  - name: api                # l'image nommée « api » dans la base...
    newName: api
    newTag: initial          # ...devient api:initial
replicas:
  - name: api
    count: 2                 # deux replicas en prod, un en dev
```

```bash
$ kubectl kustomize deploy/overlays/prod | grep -E "namespace:|replicas:|image:"
  namespace: prod
  namespace: prod
  replicas: 2
      - image: api:initial
```

<p class="small">Helm ou Kustomize ? Helm empaquette un logiciel pour le distribuer (ArgoCD, Prometheus). Kustomize décline <strong>vos</strong> manifests par environnement. Les deux coexistent très bien.</p>

---

## L'objet Application

```yaml
# deploy/argocd/api-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-dev
  namespace: argocd
spec:
  project: default
  source:                                   # d'où : un dossier d'un dépôt git
    repoURL: https://github.com/<vous>/<votre-depot>.git
    targetRevision: main
    path: deploy/overlays/dev
  destination:                              # vers où : un namespace d'un cluster
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true                           # supprime ce qui a disparu de git
      selfHeal: true                        # annule les modifications manuelles
    syncOptions:
      - CreateNamespace=true
```

---

## Synchronisé, en bonne santé

```bash
$ kubectl get applications -n argocd
NAME       SYNC STATUS   HEALTH STATUS
api-dev    Synced        Healthy
api-prod   Synced        Healthy
```

| | Valeurs | Question posée |
|---|---|---|
| **Sync** | `Synced`, `OutOfSync` | le cluster correspond-il à git ? |
| **Health** | `Healthy`, `Progressing`, `Degraded`, `Missing` | ce qui est déployé fonctionne-t-il ? |

Les deux sont indépendants : une application peut être `Synced` et `Degraded` (git est appliqué à la lettre, mais l'image n'existe pas).

| Option | Sans elle |
|---|---|
| `automated` | ArgoCD montre l'écart et attend qu'on clique sur *Sync* |
| `prune` | un fichier supprimé de git laisse son objet dans le cluster |
| `selfHeal` | une modification manuelle reste en place jusqu'au prochain commit |

---

## App of apps : ArgoCD se configure par git

Qui crée les objets `Application` ? Une application, qui surveille le dossier où ils sont rangés.

```text
   Terraform ──crée──► Application « racine » ──lit──► deploy/argocd/
                                                          ├── api-dev.yaml
                                                          ├── api-prod.yaml
                                                          └── (tout ce que vous ajouterez)
```

```hcl
resource "helm_release" "racine" {
  name       = "racine"
  namespace  = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argocd-apps"
  version    = "2.0.6"
  depends_on = [helm_release.argocd]
  values = [yamlencode({ applications = { racine = {
    namespace   = "argocd"
    project     = "default"
    source      = { repoURL = var.depot, targetRevision = "main", path = "deploy/argocd" }
    destination = { server = "https://kubernetes.default.svc", namespace = "argocd" }
    syncPolicy  = { automated = { prune = true, selfHeal = true } }
  } } })]
}
```

Ajouter un composant à la plateforme, c'est **commiter un fichier** dans `deploy/argocd/`.

---

## Et la CI ?

GitOps ne remplace pas le pipeline : il lui retire le déploiement.

```text
  commit dans app/ ──► pipeline ──► construit l'image ──► la pousse sur la registry
                          │
                          └──► écrit le nouveau tag dans deploy/overlays/dev
                                           │
                                           ▼
                                  ArgoCD voit le commit et déploie
```

<div class="left">

- Le pipeline **ne touche jamais au cluster**. Il ne connaît que git et la registry.
- Chaque image a un tag **unique** (les premiers caractères du commit), jamais `latest` : sinon git ne dit plus ce qui tourne.
- La promotion en prod est un **commit** : on recopie le tag de `dev` dans `prod`.

</div>

---

## Le pipeline complet

```yaml
# .github/workflows/pipeline.yml
name: pipeline
on:
  push:
    branches: [main]
    paths: ["app/**"]              # seulement quand l'application change
permissions:
  contents: write                  # pousser le commit du tag
  packages: write                  # pousser l'image dans ghcr.io
jobs:
  construire-et-deployer:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Calculer le tag et le nom de l'image
        run: |
          echo "TAG=${GITHUB_SHA::8}" >> "$GITHUB_ENV"
          echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}" >> "$GITHUB_ENV"
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}" }
      - name: Construire et pousser l'image
        run: |
          docker build --build-arg VERSION=$TAG -t $IMAGE:$TAG app
          docker push $IMAGE:$TAG
      - name: Écrire le nouveau tag dans git
        run: |
          cd deploy/overlays/dev
          kustomize edit set image api=$IMAGE:$TAG
          git config user.name pipeline-ci
          git config user.email pipeline-ci@users.noreply.github.com
          git commit -am "Déploie l'image $TAG en dev"
          git push
```

<p class="small"><code>${GITHUB_REPOSITORY,,}</code> met le nom du dépôt en minuscules : un nom d'image n'accepte pas les majuscules.</p>

---

## La boucle infinie, et comment l'éviter

Le pipeline pousse un commit. Ce commit déclenche le pipeline. Qui pousse un commit...

<div class="left">

Trois parades, la première suffit ici :

1. **Filtrer par chemin** : le pipeline ne se déclenche que sur `app/**`, et son commit ne touche que `deploy/`.
2. Mettre `[skip ci]` dans le message du commit du pipeline.
3. Séparer en **deux dépôts** : le code d'un côté, les manifests de l'autre. C'est la pratique courante en entreprise, parce que les droits ne sont pas les mêmes.

</div>

<div class="memo">
Sur GitHub, un push fait avec le token du pipeline (<code>GITHUB_TOKEN</code>) ne déclenche de toute façon aucun autre pipeline. Le filtre par chemin reste la bonne habitude : il marche partout.
</div>

---

## Revenir en arrière

Une mauvaise version est en ligne.

```bash
git log --oneline -- deploy/overlays/dev
a7d3e21 Déploie l'image 9c1f2ab4 en dev      # la mauvaise
4b8e0c9 Déploie l'image f8d82fac en dev      # la bonne

git revert a7d3e21
git push
```

ArgoCD voit le nouveau commit et redéploie l'image précédente. Rien d'autre à faire.

<div class="left">

- Pas de procédure de rollback à écrire, à tester, à tenir à jour.
- Le rollback est **tracé** comme n'importe quel changement.
- Ne corrigez jamais « vite fait » au `kubectl` : avec `selfHeal`, votre correction disparaît en quelques secondes.

</div>

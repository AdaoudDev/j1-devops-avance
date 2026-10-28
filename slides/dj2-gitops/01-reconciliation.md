## État désiré, état observé

Vous ne dites jamais à Kubernetes **quoi faire**. Vous lui dites **ce que vous voulez**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2          # état désiré : « je veux 2 pods »
```

```bash
$ kubectl get deployment api
NAME   READY   UP-TO-DATE   AVAILABLE
api    1/2     2            1            # état observé : il en manque un
```

<p class="lead">Tout Kubernetes est construit sur cet écart, et sur ce qui le réduit.</p>

---

## La boucle de réconciliation

```text
            ┌──────────────────────────────┐
            │  1. observer l'état réel      │
            │  2. comparer à l'état désiré  │◄───┐
            │  3. agir pour réduire l'écart │    │  sans fin
            └──────────────┬───────────────┘    │
                           └────────────────────┘
```

Un **contrôleur** est un programme qui fait tourner cette boucle pour un type d'objet.

| Contrôleur | Ce qu'il surveille | Ce qu'il fait |
|---|---|---|
| Deployment | les Deployments | crée et remplace des ReplicaSets |
| ReplicaSet | les ReplicaSets | crée ou supprime des pods |
| Scheduler | les pods sans node | leur choisit un node |
| kubelet | les pods de son node | lance les conteneurs |

Vous supprimez un pod ? Personne ne « relance » rien : le contrôleur constate qu'il en manque un.

---

## Le control plane

```text
   kubectl ──► API server ◄──► etcd        (la seule base de données)
                   ▲
       ┌───────────┼─────────────┐
   scheduler   controller      kubelet
               manager         (sur chaque node)
```

<div class="left">

- **API server** : la seule porte d'entrée. Tout le monde lui parle, personne ne se parle directement.
- **etcd** : stocke l'état désiré et l'état observé de tous les objets.
- **Les contrôleurs** : observent l'API, agissent, écrivent le résultat dans l'API.

</div>

<div class="valid">
Conséquence : pour étendre Kubernetes, il suffit d'ajouter un <strong>type d'objet</strong> dans l'API et un <strong>contrôleur</strong> qui le surveille.
</div>

---

## CRD : un nouveau type d'objet

Une *Custom Resource Definition* ajoute un type à l'API de votre cluster.

```bash
$ kubectl get crd | grep argoproj
applications.argoproj.io      2026-10-13T08:57:08Z
applicationsets.argoproj.io   2026-10-13T08:57:08Z
appprojects.argoproj.io       2026-10-13T08:57:08Z
```

En installant ArgoCD, vous avez appris un mot nouveau à votre cluster : `Application`.

```bash
kubectl get applications -n argocd      # se manipule comme un pod ou un service
kubectl describe application api-dev -n argocd
```

<p class="small">Une CRD seule ne fait rien : c'est une ligne de plus dans un dictionnaire. Il lui faut un contrôleur.</p>

---

## Operator : un contrôleur pour une CRD

<p class="lead">Operator = CRD + contrôleur qui porte le savoir-faire d'un exploitant.</p>

| Operator | L'objet que vous écrivez | Ce qu'il fait pour vous |
|---|---|---|
| ArgoCD | `Application` | synchronise un dossier git avec le cluster |
| Sealed Secrets | `SealedSecret` | déchiffre et crée le `Secret` |
| Prometheus Operator | `ServiceMonitor`, `PrometheusRule` | configure la collecte et les alertes |
| CloudNativePG | `Cluster` | installe PostgreSQL, gère réplication et sauvegardes |
| cert-manager | `Certificate` | obtient et renouvelle les certificats TLS |

Vous allez en utiliser trois dans ce cours. **Vous ne configurerez jamais l'outil directement** : vous écrirez un objet, et son contrôleur fera le reste.

---

## Quand adopter un operator

<div class="cols">
<div>

#### Ce que ça apporte

- l'exploitation devient déclarative ;
- le savoir-faire est dans le code, pas dans la tête d'une personne ;
- tout passe par l'API : mêmes outils, mêmes droits, même audit.

</div>
<div>

#### Ce que ça coûte

- un composant de plus à mettre à jour et à surveiller ;
- des droits étendus sur le cluster ;
- quand il se trompe, il se trompe vite et partout ;
- la qualité varie beaucoup d'un operator à l'autre.

</div>
</div>

<div class="memo">
Avant d'installer un operator : qui le maintient, à quelle fréquence, et que se passe-t-il pour mes données si je le désinstalle ?
</div>

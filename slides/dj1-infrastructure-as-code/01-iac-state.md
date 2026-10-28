## Le problème : des serveurs faits à la main

> « Le serveur de prod a été installé il y a trois ans par quelqu'un qui est parti.
> Personne ne sait exactement ce qu'il y a dessus. On n'y touche pas. »

<div class="left">

- Impossible de **recréer** l'environnement à l'identique.
- Impossible de **relire** un changement avant qu'il soit fait.
- Impossible de savoir **qui a changé quoi**, et quand.
- La recette et la prod **divergent** un peu plus chaque semaine.

</div>

<p class="lead">L'infrastructure as code répond aux quatre : l'infrastructure devient un texte, versionné dans git.</p>

---

## Impératif contre déclaratif

<div class="cols">
<div>

#### Impératif : les étapes

```bash
kind create cluster --name plateforme
kubectl create namespace argocd
helm install argocd argo/argo-cd \
  --namespace argocd
```

Relancé une seconde fois : **erreur**, le cluster existe déjà.

</div>
<div>

#### Déclaratif : le résultat voulu

```hcl
resource "kind_cluster" "this" {
  name = "plateforme"
}

resource "helm_release" "argocd" {
  name      = "argocd"
  namespace = "argocd"
  chart     = "argo-cd"
}
```

Relancé une seconde fois : **rien à faire**.

</div>
</div>

On décrit l'état final. C'est l'outil qui calcule les étapes pour y arriver : c'est l'**idempotence**.

---

## Le cycle de travail

```text
   écrire          terraform plan         terraform apply
  le code   ──►   « voici ce que je  ──►  « c'est fait »
                    vais changer »
      ▲                                         │
      └──────────── git commit ◄────────────────┘
```

| Commande | Ce qu'elle fait |
|---|---|
| `terraform init` | télécharge les providers et les modules |
| `terraform plan` | compare le code à l'existant, **ne change rien** |
| `terraform apply` | applique le plan, après confirmation |
| `terraform destroy` | supprime tout ce que le code décrit |

<div class="memo">
Le <code>plan</code> est ce qu'on relit avant d'agir. Un <code>apply</code> sans avoir lu le plan, c'est un <code>rm -rf</code> les yeux fermés.
</div>

---

## Lire un plan

```text
  # module.argocd.helm_release.argocd will be created
  + resource "helm_release" "argocd" {
      + chart     = "argo-cd"
      + name      = "argocd"
      + namespace = "argocd"
      + version   = "10.10.3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

| Symbole | Sens |
|---|---|
| `+` | création |
| `~` | modification sur place |
| `-` | suppression |
| `-/+` | **destruction puis recréation** : à lire deux fois |

---

## Le state : la mémoire de Terraform

Le code dit ce que vous **voulez**. Le state dit ce que Terraform **a créé**.

```bash
$ terraform state list
module.argocd.helm_release.argocd
module.cluster.kind_cluster.this
```

<div class="left">

- C'est un fichier JSON : `terraform.tfstate`.
- Sans lui, Terraform ne sait plus que le cluster est le sien : il essaie de le recréer.
- Il contient **tout** ce que les ressources exposent, y compris les **secrets en clair** (ici, la clé d'accès au cluster).

</div>

<div class="memo">
Le state ne va <strong>jamais</strong> dans git. Première ligne du <code>.gitignore</code> : <code>*.tfstate</code>.
</div>

---

## Où ranger le state

| Stockage | Pour qui | Limite |
|---|---|---|
| Fichier local | un apprentissage, une machine | perdu avec le disque, impossible à partager |
| Backend distant (S3, GCS, Azure Blob, GitLab) | une équipe | à créer avant tout le reste |
| Service géré (HCP Terraform, Spacelift, Scalr) | une organisation | payant |

Un backend distant apporte deux choses :

- le **verrouillage** : deux `apply` simultanés ne peuvent pas corrompre le state ;
- le **chiffrement** et le contrôle d'accès, puisque le state contient des secrets.

<p class="small">Dans ce cours : state local. C'est suffisant pour une plateforme qui vit sur votre machine, et c'est un choix qu'il faudra savoir justifier.</p>

---

## Le drift

Quelqu'un modifie l'infrastructure **à la main**, sans passer par le code.

```text
   code  ──────►  « 2 replicas »
   réel  ──────►  « 5 replicas »   (un kubectl scale pendant un incident)
```

<div class="left">

- Le prochain `plan` montre l'écart et propose de **revenir à ce que dit le code**.
- Si l'écart était voulu, il faut le reporter dans le code. Sinon il sera écrasé.
- Terraform ne détecte le drift que **quand on le lance**.

</div>

<p class="lead">Cet après-midi, ArgoCD fera la même chose en continu, sans qu'on le lui demande.</p>

---

## Immutabilité : remplacer plutôt que modifier

<div class="cols">
<div>

#### Mutable

Un serveur qu'on met à jour sur place, pendant des années.

- chaque serveur finit unique ;
- le rollback est une opération à part ;
- « ça marchait avant la mise à jour ».

</div>
<div>

#### Immuable

On ne modifie pas : on construit une nouvelle version, on bascule, on jette l'ancienne.

- tous les exemplaires sont identiques ;
- revenir en arrière, c'est redéployer la version précédente ;
- ce qui tourne est ce qui a été testé.

</div>
</div>

Vous le pratiquez déjà : une **image de conteneur** ne se modifie pas, on en construit une autre.

---

## Et avec un vrai fournisseur cloud ?

Ici, le provider `kind` crée un cluster dans Docker. En entreprise, seul le provider change.

| | Ce cours | En entreprise |
|---|---|---|
| Provider | `kind` | `aws`, `azurerm`, `google`, `scaleway`, `ovh` |
| Cluster | un conteneur Docker | EKS, AKS, GKE, Kapsule |
| Autour du cluster | rien | réseau, DNS, identités, stockage, bases de données |
| Durée d'un `apply` | 2 minutes | 15 à 30 minutes |
| Coût d'une erreur | nul | une facture, ou une panne |

Le cycle `plan` puis `apply`, le state, les modules et la relecture dans git sont **strictement les mêmes**.

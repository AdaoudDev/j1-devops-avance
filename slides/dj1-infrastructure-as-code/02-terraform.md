## Anatomie d'un projet Terraform

```hcl
terraform {                       # les outils dont le projet a besoin
  required_version = ">= 1.6"
  required_providers {
    kind = { source = "tehcyx/kind", version = "~> 0.11" }
  }
}

resource "kind_cluster" "this" {  # une ressource : type, puis nom local
  name           = "plateforme"
  wait_for_ready = true
}
```

<div class="left">

- Un **provider** est un greffon qui sait parler à une API (Docker, AWS, Kubernetes, GitHub...).
- Une **resource** est un objet que Terraform crée, modifie et supprime.
- `~> 0.11` : toute version 0.x à partir de 0.11. Les versions exactes sont figées dans `.terraform.lock.hcl`, **qui se commite**.

</div>

<p class="small">OpenTofu est le fork libre de Terraform : mêmes fichiers, commande <code>tofu</code> à la place de <code>terraform</code>.</p>

---

## Un provider branché sur un autre

Helm doit se connecter au cluster que `kind` vient de créer.

```hcl
provider "helm" {
  kubernetes = {
    host                   = kind_cluster.this.endpoint
    client_certificate     = kind_cluster.this.client_certificate
    client_key             = kind_cluster.this.client_key
    cluster_ca_certificate = kind_cluster.this.cluster_ca_certificate
  }
}
```

`kind_cluster.this.endpoint` est une **référence** : `type.nom.attribut`.

<div class="valid">
Grâce aux références, Terraform connaît l'ordre : le cluster d'abord, ce qui s'y installe ensuite. Vous n'écrivez jamais cet ordre vous-même.
</div>

---

## Installer ArgoCD : une release Helm

```hcl
resource "helm_release" "argocd" {
  name             = "argocd"
  namespace        = "argocd"
  create_namespace = true
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  version          = "10.10.3"
  timeout          = 600
}
```

```bash
$ terraform apply
...
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

$ kubectl --context kind-plateforme -n argocd get deploy argocd-server
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
argocd-server   1/1     1            1           21s
```

<p class="small">Toujours figer la <code>version</code> d'un chart : sans elle, deux <code>apply</code> à un mois d'écart n'installent pas la même chose.</p>

---

## Variables : sortir les valeurs du code

<div class="cols">
<div>

#### variables.tf : la déclaration

```hcl
variable "nom_cluster" {
  description = "Nom du cluster kind"
  type        = string
}

variable "version_argocd" {
  description = "Version du chart argo-cd"
  type        = string
}
```

</div>
<div>

#### plateforme.tfvars : les valeurs

```hcl
nom_cluster    = "plateforme"
version_argocd = "10.10.3"
```

#### l'usage

```hcl
name = var.nom_cluster
```

</div>
</div>

```bash
terraform apply -var-file=plateforme.tfvars
```

<div class="memo">
Un <code>.tfvars</code> sans secret <strong>se commite</strong>. Beaucoup de modèles de <code>.gitignore</code> l'excluent par défaut : vérifiez le vôtre.
</div>

---

## Modules : pourquoi

Un module est un **dossier de fichiers Terraform** qu'on appelle comme une fonction.

<div class="left">

- **Réutiliser** : le même module `cluster` pour trois environnements.
- **Cacher la complexité** : celui qui appelle donne un nom, pas quinze paramètres.
- **Donner un contrat** : des variables en entrée, des `output` en sortie.

</div>

```text
infra/
├── main.tf               appelle les modules
├── variables.tf
├── versions.tf           providers
├── plateforme.tfvars
└── modules/
    ├── cluster/          main.tf, variables.tf, outputs.tf
    └── argocd/           main.tf, variables.tf
```

---

## Écrire un module

<div class="cols">
<div>

#### modules/cluster/main.tf

```hcl
terraform {
  required_providers {
    kind = { source = "tehcyx/kind" }
  }
}

resource "kind_cluster" "this" {
  name           = var.nom
  wait_for_ready = true
}
```

#### modules/cluster/variables.tf

```hcl
variable "nom" {
  type = string
}
```

</div>
<div>

#### modules/cluster/outputs.tf

```hcl
output "endpoint" {
  value = kind_cluster.this.endpoint
}

output "client_key" {
  value     = kind_cluster.this.client_key
  sensitive = true
}
```

Un `output` est ce que le module **expose** à celui qui l'appelle. `sensitive` masque la valeur dans les affichages.

</div>
</div>

---

## Appeler un module

```hcl
module "cluster" {
  source = "./modules/cluster"
  nom    = var.nom_cluster
}

module "argocd" {
  source        = "./modules/argocd"
  version_chart = var.version_argocd
  depends_on    = [module.cluster]
}
```

Les sorties d'un module se lisent avec `module.<nom>.<output>` :

```hcl
provider "helm" {
  kubernetes = {
    host = module.cluster.endpoint
    # ...
  }
}
```

<p class="small">Après avoir ajouté ou déplacé un module : <code>terraform init</code> à nouveau.</p>

---

## Plusieurs environnements

| Approche | Principe | Quand |
|---|---|---|
| Un `.tfvars` par environnement | même code, `-var-file=prod.tfvars` | environnements identiques à la taille près |
| Workspaces | même code, un state par workspace | idem, mais on oublie vite dans lequel on est |
| Un dossier par environnement | `envs/dev/`, `envs/prod/`, modules partagés | environnements qui diffèrent vraiment, le plus lisible |

Dans tous les cas : **un state par environnement**. Une erreur en recette ne doit pas pouvoir toucher la prod.

<p class="small">Dans ce cours : un seul cluster, et les environnements <code>dev</code> et <code>prod</code> sont deux namespaces. La séparation se fera cet après-midi, côté Kubernetes.</p>

---

## Ce qu'on commite, ce qu'on ignore

<div class="cols">
<div>

#### Dans git

```text
*.tf
*.tfvars              (sans secret)
.terraform.lock.hcl
```

</div>
<div>

#### Dans .gitignore

```text
.terraform/
*.tfstate
*.tfstate.*
*-config
```

</div>
</div>

<div class="left">

- `.terraform/` : les providers téléchargés, plusieurs centaines de Mo.
- `*.tfstate` : le state, avec ses secrets.
- `*-config` : le provider `kind` écrit le fichier d'accès au cluster à côté du code.

</div>

<div class="memo">
Un secret commité une fois reste dans l'historique pour toujours. Écrivez le <code>.gitignore</code> <strong>avant</strong> le premier <code>terraform apply</code>.
</div>

# DevOps avancé : première journée

Support de la **première journée** : les slides des deux demi-journées, les énoncés TP0 à TP4,
le mémo des commandes et la grille d'évaluation.

## Avant de venir : ta machine

Tout le cours se fait **en local, sur ta machine**. Aucun compte cloud n'est nécessaire.

**Il te faut environ 8 Go de mémoire libres et 10 Go de disque.** Installe et vérifie ces
outils **avant** la séance :

| Outil | Rôle | Installation |
|---|---|---|
| Docker (Desktop ou Engine) | moteur de conteneurs | <https://docs.docker.com/get-docker/> |
| kind | cluster Kubernetes local | <https://kind.sigs.k8s.io/docs/user/quick-start/#installation> |
| kubectl | accès au cluster | <https://kubernetes.io/docs/tasks/tools/> |
| Helm | packages Kubernetes | <https://helm.sh/docs/intro/install/> |
| Terraform (ou OpenTofu) | infrastructure as code | <https://developer.hashicorp.com/terraform/install> |
| kubeseal 0.40.0 | chiffrement des secrets | <https://github.com/bitnami/sealed-secrets/releases> |
| git | | |
| Un shell bash | script de vérification | sous Windows : WSL2 recommandé, Git Bash accepté |

Il te faut aussi un compte **GitHub** : tu y créeras un dépôt public.

Vérification rapide :

```bash
docker version --format '{{.Server.Version}}' && kind version && kubectl version --client \
  && helm version --short && terraform version && git --version
```

## Ce que tu vas construire

Une plateforme de déploiement complète, dans **ton** dépôt git, sur **ta** machine. Chaque TP
ajoute une couche à la même plateforme. Il n'y a pas de projet à part : le projet, c'est ton
dépôt à la fin du cours. Le travail est **individuel**.

| Demi-journée | Théorie | TP |
|---|---|---|
| 1 : L'infrastructure est du code | une heure | TP0 (30′) remise à niveau · TP1 (~1h30) la plateforme en code |
| 2 : Git est la source de vérité | une heure | TP2 (~1h30) GitOps · TP3 (~1h30) du commit au cluster · TP4 (~30′) un secret dans git |

## Lancer les slides

Les slides sont en Reveal.js et chargent des fichiers Markdown : il faut un serveur HTTP local
(`file://` ne marche pas).

```bash
cd slides
python -m http.server 8000
```

Puis ouvre <http://localhost:8000>. Alternatives : `npx serve`, ou l'extension **Live Server**
de VS Code sur `slides/index.html`.

> Les slides servent aussi de **documentation de référence** pendant les TP : elles sont
> volontairement détaillées. Navigation : flèches ou espace, vue d'ensemble avec `O`,
> recherche avec `Ctrl+Maj+F`.

## Le dépôt de support

<https://github.com/DeepSnowNeeL/devops-avance-support> contient l'application que tu
déploies et le script qui vérifie ta plateforme :

```bash
# depuis la racine de ton dépôt, tout commité et poussé
curl -fsSL https://raw.githubusercontent.com/DeepSnowNeeL/devops-avance-support/main/verifier.sh | bash
```

## Assistants IA

Tu peux te servir d'un assistant IA comme d'un collègue, pas pour construire ta plateforme à ta
place : tu dois pouvoir expliquer chaque fichier de ton dépôt. Au TP0, tu copies à la racine de
ton dépôt les fichiers `AGENTS.md` et `CLAUDE.md` du dépôt de support : ils donnent ces règles
aux assistants de code, qui les lisent automatiquement.

## Évaluation

`note finale = plateforme × 2/3 + quiz × 1/3`

La plateforme est notée sur une **grille de dix cases**, vérifiée par le script. Lis
`evaluation/grille.html` dès maintenant : c'est la cible à viser. **L'historique de commits fait
partie de la preuve** : le script refuse un dépôt non commité ou non poussé.

Des **noms sont imposés** (cluster `plateforme`, namespaces `dev` et `prod`, déploiement
`api`...) parce que le script s'appuie dessus. Ils sont dans `tp/memo-j1.html`.

## Arborescence

```
j1-devops-avance/
├── README.md
├── slides/        deux decks, un par demi-journée
├── tp/            énoncés TP0 à TP4, mémo des commandes
└── evaluation/    grille des dix cases
```

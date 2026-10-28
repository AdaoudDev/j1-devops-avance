# DevOps avancé

## 1 : L'infrastructure est du code

<p class="small">Une heure de théorie · puis TP en autonomie</p>

---

## Ce que vous allez construire

<p class="lead">Une plateforme de déploiement complète, sur votre machine, en deux jours.</p>

```text
  git push ──► pipeline ──► image ──► tag écrit dans git
                                            │
   ┌────────────────────────────────────────▼──────────┐
   │ cluster Kubernetes local (kind), créé par Terraform│
   │                                                    │
   │   ArgoCD ──► lit git ──► déploie en dev et en prod │
   │   Prometheus + Grafana ──► SLO et alerte           │
   │   Kyverno ──► refuse ce qui n'est pas conforme     │
   └────────────────────────────────────────────────────┘
```

Chaque TP ajoute une couche. **Il n'y a pas de projet à part** : le projet, c'est votre dépôt à la fin.

---

## Les règles du jeu

<div class="left">

- **Un dépôt git par étudiant**, public, sur GitHub. Le travail est individuel.
- **Une grille de dix cases**, 2 points chacune : c'est fait, ou ce n'est pas fait.
- **Un script vérifie les cases** sur votre machine. Vous le lancez quand vous voulez.
- **L'historique de commits fait partie de la preuve** : le script refuse un dépôt non commité ou non poussé.
- **L'IA comme un collègue, pas à votre place** : vous devez pouvoir expliquer chaque fichier de votre dépôt, et je peux vous le demander.
- **Des noms imposés** (cluster `plateforme`, namespaces `dev` et `prod`, déploiement `api`...) : le script s'appuie dessus.

</div>

`note finale = plateforme × 2/3 + quiz × 1/3`

<p class="small">Grille : <code>evaluation/grille.html</code>. Noms imposés : <code>tp/memo-j1.html</code>.</p>

---

## Cette demi-journée

| | |
|---|---|
| 25′ | L'infrastructure as code : déclaratif, state, drift, immutabilité |
| 25′ | Terraform en pratique : providers, variables, modules |
| 10′ | Du DevOps au Platform Engineering |
| 30′ | **TP0** : remise à niveau, un cluster à la main |
| ~1h30 | **TP1** : la plateforme en code |

## À vous

<p class="lead">À partir de maintenant, vous êtes en autonomie jusqu'à la fin de la journée.</p>

<div class="left">

| | | |
|---|---|---|
| **TP2** | ~1h30 | GitOps : l'application en `dev` et en `prod`, déployée par ArgoCD |
| **TP3** | ~1h30 | Du commit au cluster : le pipeline construit et déploie |
| **TP4** | ~30′ | Un secret dans git, scellé |

</div>

<p class="small">Énoncés : <code>tp/tp2-gitops.html</code>, <code>tp/tp3-du-commit-au-cluster.html</code>, <code>tp/tp4-secret-dans-git.html</code>. Commandes : <code>tp/memo-j1.html</code>.</p>

---

## TP2 : GitOps

<div class="left">

1. Les manifests de l'application dans `deploy/base/`.
2. Deux overlays : `dev` (un replica) et `prod` (deux replicas).
3. Deux objets `Application` dans `deploy/argocd/`, en synchronisation automatique.
4. Une application racine, déclarée par Terraform, qui surveille `deploy/argocd/`.

</div>

<div class="valid">
✅ <strong>Validation</strong> : <code>kubectl -n dev delete deployment api</code>, et le déploiement revient tout seul.
</div>

Cases de la grille : **3** (application en `dev` et `prod`) et **4** (self-heal).

---

## TP3 et TP4

<div class="left">

**TP3 : du commit au cluster.** Un pipeline construit l'image, la pousse sur la registry, et écrit son tag dans `deploy/overlays/dev`, avec l'auteur `pipeline-ci`.

</div>

<div class="valid">
✅ <strong>Validation</strong> : vous modifiez <code>app/app.py</code>, vous poussez, et la nouvelle version répond en <code>dev</code> sans que vous ayez tapé un seul <code>kubectl</code>.
</div>

<div class="left">

**TP4 : un secret dans git.** Sealed Secrets installé par ArgoCD, un SealedSecret commité, consommé par l'application.

</div>

<div class="valid">
✅ <strong>Validation</strong> : l'API répond <code>"jeton_present": true</code>, et la valeur du secret est introuvable dans votre dépôt.
</div>

Cases de la grille : **5** (déploiement par le pipeline) et **6** (SealedSecret).

---

## Le passage de vérification

Avant la fin de la journée, appelez-moi pour un premier passage.

```bash
curl -fsSL https://raw.githubusercontent.com/DeepSnowNeeL/devops-avance-support/main/verifier.sh | bash
```

<div class="left">

- Cluster démarré, terminal à la racine du dépôt, **tout commité et poussé**.
- Vous lancez le script devant moi, je relève le code. Deux minutes.
- Cases attendues à la fin de cette journée : **1 à 5**. La case 6 peut attendre la prochaine séance.
- Une case validée aujourd'hui **reste acquise**, quoi qu'il arrive à votre cluster ensuite.

</div>

<div class="memo">
D'ici la prochaine séance : terminez ce qui ne l'est pas, et relancez tout de zéro au moins une fois (<code>terraform destroy</code> puis <code>apply</code>). Une plateforme qu'on n'a jamais reconstruite n'est pas une plateforme reproductible.
</div>

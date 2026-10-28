## À vous

<p class="lead">À partir de maintenant, vous êtes en autonomie jusqu'à la fin de la demi-journée.</p>

<div class="left">

| | | |
|---|---|---|
| **TP0** | 30′ | Remise à niveau : les outils, un cluster à la main, votre dépôt |
| **TP1** | ~1h30 | La plateforme en code : le cluster et ArgoCD par Terraform |

</div>

**Appelez-moi tôt** plutôt que de rester bloqué vingt minutes.

<p class="small">Énoncés : <code>tp/tp0-remise-a-niveau.html</code> et <code>tp/tp1-plateforme-en-code.html</code>. Commandes : <code>tp/memo-j1.html</code>.</p>

---

## TP1 : ce qui est attendu

<div class="left">

1. Terraform crée le cluster kind `plateforme`.
2. Terraform y installe ArgoCD avec Helm.
3. Le code est découpé en deux modules, les valeurs sont dans un `.tfvars`.
4. Tout est commité et poussé, sauf le state.

</div>

<div class="valid">
✅ <strong>Validation</strong> : <code>terraform destroy</code> puis <code>terraform apply</code> recrée le cluster et ArgoCD, sans aucune commande manuelle entre les deux.
</div>

Cases de la grille gagnées : **1** (cluster et ArgoCD issus de Terraform) et **2** (module et variables).

---

## Vérifier où vous en êtes

Depuis la racine de votre dépôt, tout commité et poussé :

```bash
curl -fsSL https://raw.githubusercontent.com/DeepSnowNeeL/devops-avance-support/main/verifier.sh | bash
```

```text
[ OK ]   1. Cluster et ArgoCD issus de Terraform
[ OK ]   2. Module et variables Terraform
[ECHEC]  3. Application en dev et prod par ArgoCD
         l'application api-dev est « introuvable », attendu « Synced Healthy »
...
Score : 4/20
Code : 1100000000
```

<div class="left">

- Lancez-le **aussi souvent que vous voulez** : c'est votre dashboard.
- Pour la note, vous le lancez **devant moi** et je relève le code.
- Une case validée reste acquise.

</div>

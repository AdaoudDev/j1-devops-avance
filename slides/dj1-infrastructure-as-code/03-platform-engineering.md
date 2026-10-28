## Le DevOps à l'échelle : la charge cognitive

« You build it, you run it » : l'équipe qui développe exploite aussi.

À dix équipes, chacune doit maîtriser :

<div class="left">

Docker · Kubernetes · Helm · Terraform · le pipeline · les secrets · la supervision · la sécurité · le réseau · le cloud

</div>

<div class="left">

- Chaque équipe réinvente son pipeline, un peu différemment.
- Les développeurs passent plus de temps sur l'outillage que sur leur produit.
- La sécurité et la conformité se vérifient équipe par équipe.

</div>

<p class="lead">Le problème n'est plus technique : c'est trop de choses à savoir pour une seule équipe.</p>

---

## La réponse : une plateforme interne

Une équipe dédiée construit **un produit** dont les clients sont les développeurs de l'entreprise.

```text
   équipes produit          « je veux déployer mon service »
         │
         ▼   une interface simple : un fichier, un formulaire, une commande
   ┌──────────────────────────────────────────────┐
   │  plateforme interne (IDP)                     │
   │  pipeline · déploiement · secrets · métriques │
   └──────────────────────────────────────────────┘
         │
         ▼
   Kubernetes, cloud, réseau
```

<div class="left">

- **Golden path** : le chemin balisé, celui qui marche sans rien demander à personne.
- On peut en sortir, mais alors on assume le reste soi-même.
- La plateforme se pilote comme un produit : des utilisateurs, des retours, une feuille de route.

</div>

---

## Backstage, le portail développeur

| | |
|---|---|
| **À quoi ça sert** | un catalogue de tous les services (qui possède quoi, où est la doc, quel est l'état), et des modèles pour créer un nouveau service en quelques clics |
| **Quand l'adopter** | à partir de plusieurs dizaines de services et d'équipes, quand plus personne ne sait qui possède quoi |
| **Ce que ça coûte** | c'est un cadre à développer, pas un produit fini : il faut une équipe pour l'intégrer et le maintenir |

<p class="small">Projet libre créé par Spotify, hébergé par la CNCF. Alternatives : Port, Cortex, ou simplement un dépôt modèle bien tenu.</p>

<div class="memo">
Un portail n'est pas une plateforme. Sans chemin balisé derrière, c'est une jolie page qui pointe vers le même désordre.
</div>

---

## Votre plateforme miniature

Ce que vous construisez pendant ce cours **est** une plateforme interne, à l'échelle d'une machine.

| Ce qu'offre une plateforme | Votre version |
|---|---|
| Un environnement recréable | `terraform apply` |
| Un déploiement sans ticket | `git push`, ArgoCD fait le reste |
| Des secrets gérés | Sealed Secrets |
| De la visibilité | Prometheus, Grafana, un SLO |
| Des garde-fous | scan d'images, politiques d'admission |

<p class="lead">Le golden path de votre plateforme : « je modifie mon code, je pousse, c'est en ligne, mesuré et contrôlé ».</p>

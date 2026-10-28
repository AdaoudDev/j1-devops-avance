## Mettre à jour sans couper : rolling update

C'est ce que fait un Deployment **par défaut**, et ce que vous utiliserez dans ce cours.

```text
   v1 v1 v1 v1      départ
   v1 v1 v1 v2      un pod v2 démarre, un pod v1 s'arrête quand v2 est prêt
   v1 v1 v2 v2
   v1 v2 v2 v2
   v2 v2 v2 v2      arrivée
```

<div class="left">

- Aucune coupure, **à condition** d'avoir une `readinessProbe` : sans elle, Kubernetes envoie du trafic à un pod qui n'a pas fini de démarrer.
- Pendant la bascule, **les deux versions répondent en même temps** : elles doivent être compatibles (schéma de base, format des messages).
- Rollback : la même bascule dans l'autre sens.

</div>

---

## Blue-green : deux environnements, une bascule

```text
   trafic ──► [ blue  : v1 v1 v1 ]        green est déployé et testé à côté
              [ green : v2 v2 v2 ]

   trafic ──► [ green : v2 v2 v2 ]        bascule instantanée
              [ blue  : v1 v1 v1 ]        gardé un moment, pour revenir
```

| | |
|---|---|
| **À quoi ça sert** | basculer d'un coup, après avoir testé la nouvelle version dans les conditions réelles |
| **Quand l'adopter** | versions incompatibles entre elles, besoin d'un rollback immédiat |
| **Ce que ça coûte** | le double de ressources pendant la bascule, et une base de données qui doit servir les deux versions |

---

## Canary : une petite part d'abord

```text
   trafic ──► 95 % ──► v1 v1 v1 v1
          └─►  5 % ──► v2                 on observe les métriques

   si tout va bien : 5 % → 25 % → 50 % → 100 %
   sinon           : retour à 0 %, peu d'utilisateurs touchés
```

| | |
|---|---|
| **À quoi ça sert** | limiter le nombre d'utilisateurs exposés à un défaut |
| **Quand l'adopter** | trafic suffisant pour que 5 % soit significatif, et des métriques fiables pour décider |
| **Ce que ça coûte** | il faut répartir le trafic finement (ingress ou service mesh) et savoir mesurer « ça va bien » |

<p class="small">Sans métriques, un canary n'est qu'un déploiement lent. C'est le sujet de la prochaine demi-journée.</p>

---

## Progressive delivery : Argo Rollouts

Automatiser le canary : c'est la machine qui décide d'avancer ou de reculer.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout                      # remplace le Deployment
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis:                # interroge Prometheus : le taux d'erreurs est-il bon ?
            templates: [{ templateName: taux-de-succes }]
        - setWeight: 50
        - pause: { duration: 5m }
```

| | |
|---|---|
| **À quoi ça sert** | des déploiements qui s'annulent seuls si les métriques se dégradent |
| **Quand l'adopter** | déploiements fréquents, SLO déjà en place |
| **Ce que ça coûte** | un operator de plus, un objet `Rollout` à la place du `Deployment`, des analyses à écrire et à régler |

<p class="small">Alternative : Flagger. Même idée, autre operator.</p>

---

## Choisir

| | Rolling | Blue-green | Canary |
|---|---|---|---|
| Coupure | aucune | aucune | aucune |
| Deux versions en même temps | oui, brièvement | non | oui, longtemps |
| Ressources en plus | faibles | le double | faibles |
| Rollback | une bascule | instantané | instantané |
| Utilisateurs touchés par un défaut | tous, progressivement | tous, d'un coup | une petite part |
| Prérequis | readiness probe | de quoi doubler | métriques, traffic splitting |

<p class="lead">On commence toujours par un rolling update bien réglé. Le reste se mérite avec des métriques.</p>

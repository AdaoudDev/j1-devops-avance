## Le problème : un Secret n'est pas secret

Si tout est dans git, les mots de passe aussi ?

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
data:
  JETON: dmFsZXVyLWRlLWRlbW8=
```

```bash
$ echo dmFsZXVyLWRlLWRlbW8= | base64 -d
valeur-de-demo
```

<div class="left">

- `base64` est un **encodage**, pas un chiffrement. N'importe qui le décode.
- Un `Secret` commité est un mot de passe publié, **pour toujours** : l'historique git ne s'efface pas.
- Mais sans lui dans git, le cluster n'est plus entièrement décrit par git.

</div>

---

## Sealed Secrets : chiffrer pour le cluster

```text
   votre machine                         le cluster
   ─────────────                         ──────────
   Secret (en clair, jamais commité)     contrôleur sealed-secrets
        │                                  détient la clé PRIVÉE
        │  kubeseal                              │
        │  chiffre avec la clé PUBLIQUE          │
        ▼                                        ▼
   SealedSecret ──── git ──── ArgoCD ───► déchiffre ──► Secret
   (chiffré, commitable)
```

<div class="left">

- La clé publique chiffre, **seule la clé privée déchiffre**, et elle ne quitte jamais le cluster.
- Le `SealedSecret` peut être public : personne ne peut le lire, pas même vous.
- C'est un operator : vous commitez un `SealedSecret`, le contrôleur crée le `Secret`.

</div>

---

## Sceller un secret

```bash
kubectl create secret generic api-secret --namespace dev \
    --from-literal=JETON=valeur-de-demo \
    --dry-run=client -o yaml \
  | kubeseal --context kind-plateforme --format yaml \
  > deploy/overlays/dev/api-secret-scelle.yaml
```

<div class="left">

- `--dry-run=client -o yaml` : fabrique le `Secret` **sans l'envoyer** au cluster.
- `kubeseal` va chercher la clé publique dans le cluster et chiffre.
- Seul le fichier chiffré est écrit sur le disque, **dans l'overlay** : Kustomize ne lit pas hors de son dossier.

</div>

Et le déploiement le consomme comme n'importe quel secret :

```yaml
          envFrom:
            - secretRef:
                name: api-secret
```

<p class="small">Un SealedSecret est lié à son nom et à son namespace : scellé pour <code>dev</code>, il est inutilisable en <code>prod</code>.</p>

---

## Les autres approches

| Outil | Principe | Quand le préférer |
|---|---|---|
| **Sealed Secrets** | secret chiffré dans git, déchiffré par le cluster | petite équipe, peu de secrets, aucune infrastructure à côté |
| **SOPS** | fichier chiffré dans git, déchiffré au déploiement avec une clé externe (KMS, age) | secrets partagés entre plusieurs clusters ou hors Kubernetes |
| **External Secrets Operator** | git ne contient qu'une **référence**, la valeur vit dans un secret manager | l'entreprise a déjà un secret manager (Vault, AWS Secrets Manager, Azure Key Vault) |

<p class="lead">Les deux premiers mettent le secret chiffré dans git. Le troisième n'y met que son adresse.</p>

<div class="left">

Avec un secret manager : rotation automatique, audit des accès, secrets à durée de vie courte. C'est la cible en entreprise, et c'est ce qu'il faut savoir proposer.

</div>

---

## Le piège : la clé perdue

La clé privée vit **dans le cluster**. Vous détruisez le cluster : elle disparaît.

```bash
terraform destroy && terraform apply
```

<div class="left">

- Le nouveau contrôleur génère une **nouvelle** paire de clés.
- Tous vos `SealedSecret` commités sont devenus indéchiffrables.
- L'application démarre sans son secret.

</div>

<div class="memo">
En production, la clé privée du contrôleur se <strong>sauvegarde</strong> hors du cluster, et se restaure avant tout le reste. Ici, vous rescellerez votre secret après chaque reconstruction du cluster : c'est une commande.
</div>

<p class="small">C'est le point faible de Sealed Secrets, et l'argument principal pour un secret manager externe.</p>

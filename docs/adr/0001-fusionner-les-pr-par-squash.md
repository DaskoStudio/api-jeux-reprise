# 1. Fusionner les pull requests en squash

- **Date** : 2026-10-06
- **Statut** : Acceptée

## Contexte

Le projet est repris par une équipe de plusieurs membres. Chaque changement
arrive sur `main` par une pull request relue ; une PR regroupe souvent
plusieurs commits de travail (« test catalogue vide », « correction du test »,
« fix »…). GitHub propose trois façons de fusionner une PR, et le comportement
par défaut du bouton vert est *Create a merge commit*. Sans décision commune,
chacun fusionne à sa manière et l'historique de `main` devient illisible.

Nous voulons un `main` dont l'historique se lit vite : un lecteur doit pouvoir
retrouver *quelle PR* a introduit un changement, sans se noyer dans les commits
intermédiaires de chaque auteur.

## Options envisagées

1. **Merge commit** (tous les commits de la branche, plus un commit de fusion).
   Pour : conserve l'historique complet, y compris l'ordre réel des commits.
   Contre : les commits de travail (`wip`, `fix typo`, `correction du test`)
   se retrouvent sur `main` ; l'historique est bruité et difficile à relire.

2. **Squash and merge** (un seul commit par PR).
   Pour : un commit sur `main` égale une PR relue ; l'historique est lisible,
   le message du commit est le titre de la PR. Contre : le détail des commits
   intermédiaires disparaît (il reste consultable dans la PR fermée).

3. **Rebase and merge** (les commits de la branche, sans commit de fusion).
   Pour : historique linéaire, sans commit de fusion. Contre : exige des
   commits déjà propres et atomiques ; avec nos commits de travail, `main`
   hériterait du même bruit que l'option 1.

## Décision

Nous fusionnons les pull requests avec **Squash and merge**. Le titre de la PR
devient le message du commit sur `main` : il doit décrire le changement
(« Corriger les statistiques sur un catalogue vide »), pas se résumer à « fix ».
La branche est supprimée après la fusion.

## Conséquences

- **Plus facile** : l'historique de `main` est lisible — un commit = une PR
  relue. `git log` sur `main` donne la liste des changements fonctionnels.
- **Plus difficile** : le détail des commits intermédiaires n'apparaît plus sur
  `main`. Il reste accessible dans la pull request fermée, pas dans
  `git log` ; un `git bisect` est donc moins fin.
- **À revoir si** : l'équipe adopte des commits atomiques déjà propres (un
  commit = une intention vérifiée), auquel cas *Rebase and merge* donnerait un
  historique linéaire aussi lisible, sans perdre le détail.

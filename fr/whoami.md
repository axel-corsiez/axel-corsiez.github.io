---
layout: post
title: "whoami"
permalink: /fr/whoami/
lang: fr
alt_url: /whoami/
---

Chercheur en sécurité offensive. Je passe l'essentiel de mon temps à lire le code des autres,
en cherchant les endroits où un contrôle de sécurité existe mais ne couvre pas tout ce qu'il
devrait.

Ça paraît étroit, et c'est volontairement étroit. Presque aucun des onze avis publiés qui me
créditent ne relève du « développeur qui a oublié la sécurité ». C'est l'inverse : un garde-fou
a été écrit, correctement, et un chemin de code est resté en dehors. Un sanitiseur soixante
lignes au-dessus de l'endpoint qui ne l'appelle pas. Un décorateur d'authentification placé une
ligne trop haut, que le framework écarte silencieusement. Une liste blanche qui couvre
l'écrivain de configuration principal mais pas celui des plugins. Les trouver consiste à
établir quelle est l'écriture majoritaire dans une base de code, puis à lire la minorité.

## Ce sur quoi je travaille

**Recherche de vulnérabilités sur du logiciel open source.** Audit de code source, analyse de
variantes à partir des correctifs publiés, divulgation coordonnée. Onze avis publiés à ce jour,
couvrant le contournement d'authentification, la traversée de chemin non authentifiée, la SSRF,
l'autorisation inter-locataires et l'exécution de commande. Ils sont listés avec leurs articles
sur [l'accueil]({{ '/fr/' | relative_url }}).

**Bug bounty sur cibles vivantes.** Les mêmes réflexes, une visibilité différente : pas de code
source, seulement ce que la cible expose. Recon et cartographie de surface d'attaque, minage des
bundles JavaScript pour les endpoints que l'interface n'appelle jamais, et test de contrôle
d'accès mené avec deux identités contrôlées, pour qu'une faille d'autorisation soit démontrée et
non affirmée. Dans le périmètre, sur des programmes qui l'autorisent, jamais avec les données
d'un autre utilisateur.

Je débute sur les plateformes et je le dis franchement : des rapports soumis, des duplicates
jusqu'ici, aucun validé pour l'instant. Un duplicate, c'est la plateforme qui confirme que le bug
était réel et que quelqu'un l'a atteint avant moi ; c'est le premier chapitre ordinaire et un
signal de calibrage raisonnable. Les avis plus haut sont l'endroit où la même lecture a déjà
abouti, et ce qui sépare les deux, c'est la surface visée, pas la méthode.

**Outillage.** J'ai construit le pipeline qui rend tout ça possible à l'échelle plutôt qu'un
dépôt à la fois, et j'écris sur son fonctionnement comme sur ses limites.

**Sécurité des applications web.** Contrôle d'accès, falsification de requête côté serveur,
injection, authentification et gestion de session. C'est là que tombe la majorité de mes
découvertes.

## Pourquoi l'open source

Les logiciels que les gens auto-hébergent sont maintenus par des équipes de deux ou trois. Ce
sont de bons ingénieurs, ce ne sont pas des ingénieurs sécurité, ils n'ont pas de budget d'audit,
et les projets sont assez gros pour que personne n'en tienne la totalité en tête. Pendant ce
temps, ce code fait tourner l'outillage interne d'entreprises qui ne le publieraient jamais
elles-mêmes.

C'est dans cet écart que je travaille. Onze avis, dix versions correctives livrées par leurs
mainteneurs, le tout gratuit pour le projet et permanent pour chaque déploiement qui se met à
jour. C'est le travail de sécurité au meilleur effet de levier que je connaisse, et la raison
pour laquelle rien de tout ça n'est derrière un paywall ici.

## Comment je travaille

Une découverte n'en est pas une tant qu'une preuve de concept ne s'exécute pas. Pas de
conditionnel, pas de « un attaquant pourrait éventuellement ». Si ça ne se reproduit pas sur un
déploiement réel avec des données réalistes, ce n'est pas remonté ; et si ça se reproduit sans
rien accorder de conséquent, ce n'est pas remonté non plus : un bug techniquement réel mais sans
impact gaspille l'après-midi d'un mainteneur.

Les rapports partent avec la preuve de concept qui fonctionne et une remédiation concrète, parce
que l'objectif est un correctif et pas un trophée. Les avis sont publiés quand le projet est
prêt, pas quand ce serait le plus intéressant à poster.

## Divulgation

Tout ce qui est publié ici concerne du code open source analysé en local, ou des systèmes que je
suis autorisé à tester. Les découvertes vont d'abord aux mainteneurs et ne font l'objet d'un
article qu'une fois corrigées. Le travail encore en relecture apparaît sur ce site sous forme de
compteur, sans nom de projet.

## Contact

<ul class="contact">
  <li><b>github</b><a href="https://github.com/axel-corsiez" rel="me noopener">axel-corsiez</a></li>
  <li><b>linkedin</b><a href="https://www.linkedin.com/in/axel-corsiez/" rel="me noopener">axel-corsiez</a></li>
  <li><b>mail</b><a href="{{ site.email_href }}">{{ site.email_text }}</a></li>
</ul>

<p class="avail"><b>Ouvert aux opportunités.</b> Recherche de vulnérabilités, sécurité applicative, outillage offensif.</p>

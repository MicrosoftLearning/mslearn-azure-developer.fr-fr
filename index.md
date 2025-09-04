---
title: Exercices pour les développeurs Azure
permalink: index.html
layout: home
---

## Vue d’ensemble

Les exercices suivants sont conçus pour vous offrir une expérience d’apprentissage pratique dans laquelle vous explorerez les tâches courantes que les développeurs effectuent lors de la génération et du déploiement de solutions sur Microsoft Azure.

> **Note** : pour effectuer les exercices, vous aurez besoin d’un abonnement Azure et de disposer des autorisations et du quota suffisants pour approvisionner les ressources Azure nécessaires. Si vous n’avez pas de compte Azure, vous pouvez en créer un [ici](https://azure.microsoft.com/free). 

Certains exercices peuvent avoir des exigences supplémentaires ou différentes. Ceux-ci comporteront une section **Avant de commencer** spécifique à cet exercice.

## Zones de rubrique
{% assign exercises = site.pages | where_exp:"page", "page.url contains '/instructions'" %} {% assign grouped_exercises = exercises | group_by: "lab.topic" %}

<ul>
{% for group in grouped_exercises %}
<li><a href="#{{ group.name | slugify }}">{{ group.name }}</a></li>
{% endfor %}
</ul>

{% for group in grouped_exercises %}

## <a id="{{ group.name | slugify }}"></a>{{ group.name }} 

{% for activity in group.items %} [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }}

---

{% endfor %} <a href="#overview">Retour en haut</a> {% endfor %}


# Slab allocator writeup

## Sommaire
1. [Intro](#Intro)
2. [Composants](#Composants)

    2.1 [Objet kernel](#Objet-kernel)
    
    2.2 [Slab](#Slab)
    
    2.3 [Cache](#Cache)
3. [Le slab allocator](#Le-slab-allocator)
4. [Points importants et contraintes](#Points-importants-et-contraintes)
5. [Use-cases et utilité](#Use-cases-et-utilité)


## Intro

La gestion dynamique de la mémoire est un enjeu central dans les systèmes d’exploitation modernes, tant pour des raisons de performance que de sécurité. Dans le noyau Linux, cette gestion repose en grande partie sur des allocateurs spécialisés, conçus pour répondre aux contraintes spécifiques du contexte kernel : allocations fréquentes, tailles d’objets variées, fragmentation minimale et faible surcoût.

Parmi ces mécanismes, le slab allocator occupe une place clé. Introduit pour optimiser l’allocation d’objets de taille fixe, il repose sur le principe de caches d’objets préalloués, permettant de réduire les coûts liés aux appels répétés à l’allocateur général et d’améliorer la localité mémoire. Ce modèle a également des implications importantes en matière de sécurité, notamment dans le contexte des vulnérabilités de type use-after-free, heap overflow ou exploitation du kernel heap.

Ce write-up a pour objectif de détailler le fonctionnement interne du slab allocator : sa structure, ses principaux composants (caches, slabs, objets), ainsi que le cycle de vie d’une allocation et d’une libération. L’accent sera mis sur les mécanismes concrets utilisés par le noyau, afin de fournir une compréhension exploitable aussi bien pour l’analyse de performances que pour la recherche et l’exploitation de vulnérabilités.

## Composants
> Avant d’entrer dans le détail du fonctionnement global du slab allocator, il est nécessaire de présenter les différentes briques qui le composent. 

> Le modèle du slab repose sur une hiérarchie de structures clairement définies, allant de l’objet kernel individuel jusqu’aux caches qui organisent leur allocation. Comprendre le rôle et les interactions entre ces composants est essentiel pour appréhender à la fois les choix de conception de l’allocateur et les comportements observables lors des allocations, des libérations et des réutilisations d’objets en mémoire.

### <ins>Objet kernel</ins>
> Objet kernel
#### Contexte et historique:

Dans un système d’exploitation comme Linux, le noyau manipule en permanence un grand nombre d’entités internes représentant l’état du système : processus, fichiers, sockets, verrous, files d’attente, timers, etc... 

Très tôt dans l’histoire des noyaux Unix, il est apparu nécessaire de représenter ces entités sous forme de structures de données bien définies, allouées dynamiquement en mémoire kernel.

Contrairement à l’espace utilisateur, où les allocations sont souvent variées et imprévisibles, le noyau travaille majoritairement avec des objets de taille connue à l’avance, correspondant à des structures (struct) C précises. Cette caractéristique a fortement influencé la conception des allocateurs kernel, et en particulier du slab allocator, dont l’objectif principal est d’optimiser la gestion de ces objets répétitifs, typés et fréquemment utilisés.
____
> Objet kernel
#### Définition:

Un objet kernel est une instance en mémoire d’une structure interne du noyau Linux. Il représente une ressource, un état ou un mécanisme du système, et est généralement manipulé via un pointeur vers une struct spécifique.

`ajouter des exemples...`

Ces objets sont strictement internes au noyau et ne sont jamais directement exposés à l’espace utilisateur, même si ce dernier peut indirectement provoquer leur création ou leur destruction via des appels système.
____
> Objet kernel
#### Spécificités 
**1. Taille fixe**

Un point fondamental des objets kernel est leur taille fixe.
Chaque type d’objet correspond à une structure C dont la taille est déterminée à la compilation :
```c
struct file {
    struct path f_path;
    struct inode *f_inode;
    const struct file_operations *f_op;
    ...
};
```
Toutes les instances de `struct type_de_struct` auront donc exactement la même taille

**2. Typage fort**

Le noyau Linux repose sur un typage fort par structure, même si le langage C ne fournit pas de mécanisme de sécurité à l’exécution. Chaque objet est censé être utilisé uniquement comme une instance de son type d’origine. Par exemple un pointeur vers `struct file` doit toujours référencer une `struct file` et un pointeur vers `struct semaphore` ne doit jamais être interprété comme autre chose

Cette discipline est respectée par le code kernel normal, mais elle n’est pas enforcée par le matériel. Toute confusion de type est donc catastrophique et peut mener à 🥁... :

    - corruption mémoire

    - exécution de code arbitraire

    - élévation de privilèges

`à compléter...`

### Slab
`à compléter...`

### Cache
`à compléter...`

## Le slab allocator

`Explication du méchanisme + schéma homemade`

## Points importants et contraintes 

`Rappelez les contraintes de typages et continuez d'investiguer sur plus de détail`

## Use-cases et utilité

`Diger la partie poerformance avec des chiffres, parler de son importance pour les exploits`

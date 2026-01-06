# Slab allocator writeup

## Sommaire
1. [Intro](#Intro)
2. [Composants](#Composants)

    2.1 [Objet kernel](#Objet-Kernel)
    
    2.2 [Slab](#SlabSlab)
    
    2.3 [Cache](#Cache)
3. [Le slab allocator](#Le-slab-allocator)
4. [Points importants et contraintes](#Points-importants-et-contraintes)
5. [Use-cases et utilité](#Use-cases-et-utilite)


## Intro

La gestion dynamique de la mémoire est un enjeu central dans les systèmes d’exploitation modernes, tant pour des raisons de performance que de sécurité. Dans le noyau Linux, cette gestion repose en grande partie sur des allocateurs spécialisés, conçus pour répondre aux contraintes spécifiques du contexte kernel : allocations fréquentes, tailles d’objets variées, fragmentation minimale et faible surcoût.

Parmi ces mécanismes, le slab allocator occupe une place clé. Introduit pour optimiser l’allocation d’objets de taille fixe, il repose sur le principe de caches d’objets préalloués, permettant de réduire les coûts liés aux appels répétés à l’allocateur général et d’améliorer la localité mémoire. Ce modèle a également des implications importantes en matière de sécurité, notamment dans le contexte des vulnérabilités de type use-after-free, heap overflow ou exploitation du kernel heap.

Ce write-up a pour objectif de détailler le fonctionnement interne du slab allocator : sa structure, ses principaux composants (caches, slabs, objets), ainsi que le cycle de vie d’une allocation et d’une libération. L’accent sera mis sur les mécanismes concrets utilisés par le noyau, afin de fournir une compréhension exploitable aussi bien pour l’analyse de performances que pour la recherche et l’exploitation de vulnérabilités.

## Composants

### Objet kernel :
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

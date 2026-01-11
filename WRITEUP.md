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

Parmis les plus commune:

`struct task_struct` = processus / thread

`struct file` = fichier ouvert

`struct inode` = métadonnées d’un fichier

`struct socket` = struct sock → socket réseau

`struct semaphore` = → synchronisation

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

3. Alignement

Les objets kernel sont soumis à des contraintes d’alignement mémoire, imposées par l’architecture, les exigences de performance et certains champs internes (Pointeurs, listes chaînées, etc...). Dans notre cas le slab allocator garantira que chaque objet soit correctement aligné et respecte les contraintes nécessaires à son type, ce qui améliore la localité cache, les performances et **très important** la fiabilité des accès concurrents

Mais cela rend aussi l’agencement mémoire hautement prédictible, ce qui est un point clé du point de vue de la sécurité.

4. Réutilisation après free

La réutilisation après free est un pilier du modèle slab. Lorsqu’un objet kernel est libéré, sa mémoire n’est généralement pas rendue immédiatement au système, elle est replacée dans la freelist du cache correspondant et peut être réallouée très rapidement pour un nouvel objet du même type. En conséquence, deux allocations successives du même type peuvent retourner exactement la même adresse, et la mémoire peut conserver des valeurs résiduelles si des mécanismes de durcissement ne sont pas activés. Cette réutilisation agressive offre des performances élevées, mais elle introduit des risques de sécurité majeurs.

---
> Objet kernel
#### Conclusion

Dans le cadre de l’étude du slab allocator et de la sécurité du noyau Linux, les objets kernel constituent le point d’entrée principal pour comprendre comment la mémoire du noyau est structurée, pourquoi certaines primitives d’exploitation sont possibles et comment la performance et la sécurité peuvent entrer en tension.

La combinaison de plusieurs caractéristiques comme la taille fixe, typage fort mais non vérifié à l’exécution, réutilisation rapide et disposition mémoire prévisible fait des objets kernel une cible privilégiée pour les attaques. En parallèle, ces mêmes propriétés en font une abstraction idéale pour optimiser la gestion mémoire du noyau.



### <ins>Slab</ins>
> Slab
#### Contexte et historique:

Avec l’augmentation de la complexité du noyau Linux et la multiplication des objets kernel, les limites des allocateurs génériques sont rapidement apparues. Les premières approches basées sur `kmalloc` ou des allocateurs de type buddy system étaient efficaces pour gérer des pages mémoire, mais peu adaptées à la gestion intensive d’objets de petite taille, fortement typés et fréquemment alloués/libérés.

C’est dans ce contexte que le concept de **slab allocator** a été introduit à la fin des années 1990 (initialement par Sun Microsystems), puis adopté par Linux. L’idée centrale était de regrouper des objets kernel de même type au sein de blocs mémoire appelés *slabs*, afin de réduire les coûts d’allocation, d’améliorer la localité cache et de limiter la fragmentation.

Le slab devient ainsi une unité intermédiaire entre les pages mémoire physiques et les objets kernel individuels.

---
> Slab
#### Définition:

Un **slab** est un ensemble contigu de mémoire, généralement constitué d’une ou plusieurs pages physiques, dédié à contenir des objets kernel **tous du même type et de la même taille**.

Concrètement un slab appartient toujours à un cache précis et contient un nombre fixe d’objets. Ces objets sont soit **libres**, soit **occupés** (**free** ou **used** en anglais )

Chaque slab maintient des métadonnées permettant au noyau de savoir :

- combien d’objets sont alloués
- combien sont libres
- où se trouvent les prochaines zones réutilisables

Le slab agit donc comme une sorte de stockage d’objets prêts à l’emploi**, évitant de devoir solliciter le système de gestion de pages à chaque allocation.

---
> Slab
#### Organisation interne:

Un slab est généralement composé de deux éléments principaux :

- **la zone de données**, qui contient les objets kernel
- **les métadonnées**, utilisées pour la gestion interne (freelist, compteurs, états)

Selon l’implémentation (SLAB, SLUB), ces métadonnées peuvent être soit intégrées directement dans le slab soit partiellement stockées dans l’objet lui-même lorsqu’il est libre ou alors gérées via des structures externes. Dans tous les cas, l’objectif reste le même, permettre une allocation et une libération **rapides et déterministes**.

---

> Slab
#### États d’un slab:
Un slab peut se trouver dans plusieurs états au cours de son cycle de vie :

- **empty** : aucun objet n’est actuellement alloué
- **partial** : certains objets sont alloués, d’autres libres
- **full** : tous les objets sont alloués

Ces états sont cruciaux pour l’allocateur, qui privilégiera :

- un slab *partial* pour une nouvelle allocation
- un slab *empty* si nécessaire
- et pourra éventuellement libérer un slab entièrement vide pour récupérer des pages mémoire

Cette gestion fine permet de concilier performance et maîtrise de la consommation mémoire.

---
> Slab
#### Spécificités

**1. Conteneur d’objets homogènes**

Un slab ne contient **qu’un seul type d’objet kernel**.
Cela signifie que tous les objets à l’intérieur ont la même taille, le même type, ont le même alignement et globalement partagent les mêmes contraintes structurelles.

Cette homogénéité simplifie grandement la gestion mémoire et permet une réutilisation extrêmement rapide des objets libérés.

**2. Alignement**
Les slabs sont faits pour améliorer la localité cache. Les objets sont donc placés les uns à côté des autres en mémoire, ce qui fait que le processeur peut y accéder plus vite. L’alignement respecte les contraintes de l’architecture, donc il n’y a pas de perte de performance. Et comme les accès successifs tombent souvent sur les mêmes lignes de cache, tout devient plus rapide et efficace.

Ce qui améliore significativement les performances, en particulier pour des structures très utilisées comme les `struct file` ou `struct inode`.

**3. Réutilisation agressive**

Lorsqu’un objet est libéré, il retourne simplement dans la freelist du slab, où il peut être réutilisé immédiatement. Cependant, il n’est pas forcément réinitialisé complètement, sans mécanismes de durcissement spécifiques certaine données précédentes peuvent rester en mémoire.

Cette réutilisation rapide est l’un des principaux atouts du slab allocator, mais elle a également un impact direct sur la sécurité.

**4. Prévisibilité mémoire**

Les slabs apportent une organisation mémoire à la fois structurée, répétable et relativement prévisible. Du point de vue des performances, c’est un atout majeur, car cette régularité facilite la gestion et l’accès aux objets. En revanche, du point de vue de la sécurité, cette même prévisibilité peut devenir un avantage pour un attaquant, elle lui permet de forcer la réallocation d’un objet précis, de contrôler l’occupation des slabs ou encore d’influencer la disposition des objets en mémoire du noyau.

---
> Slab
#### Lien avec les vulnérabilités

Le slab joue un rôle central dans de nombreuses vulnérabilités kernel.

Dans un scénario de **use-after-free** :

1. un objet est libéré
2. il reste physiquement dans son slab
3. un nouvel objet peut être alloué exactement au même emplacement

Si l’attaquant parvient à influencer le type ou le contenu du nouvel objet, il peut donc écraser des champs sensibles, détourner des pointeurs ou encore provoquer une confusion de type.

---
> Slab
#### Conclusion

Le slab constitue une brique intermédiaire essentielle entre les objets kernel et les caches du slab allocator. Il permet de regrouper efficacement des objets homogènes, d’optimiser les performances grâce à la localité cache et de réduire drastiquement le coût des allocations répétées.

Cependant, cette organisation structurée et prévisible, combinée à la réutilisation rapide des objets, fait du slab un élément central dans l’analyse et l’exploitation des vulnérabilités du noyau Linux. Comprendre le rôle et le fonctionnement des slabs est donc indispensable avant d’aborder la notion de cache et, plus largement, le fonctionnement global du slab allocator.

> Cache

#### Définition:

Un **cache** (ou *slab cache*) est une structure de gestion représentant un type précis d’objet kernel. Il regroupe un ensemble de slabs et des règles d’allocation et de libération ainsi que les paramètres propres à l’objet (taille, alignement, flags, constructeur, etc.). Chaque cache est strictement associé à **un type d’objet kernel**. Lorsqu’une allocation est demandée, le noyau ne travaille jamais directement avec des slabs, mais toujours via un cache.

---

> Cache

#### Création et cycle de vie:

Les caches peuvent être **statiques** (créés au démarrage du noyau) ou **dynamiques** (créés à l’exécution via `kmem_cache_create`)

Lors de la création d’un cache, plusieurs paramètres critiques sont définis :

- la taille exacte de l’objet
- les contraintes d’alignement
- les flags de sécurité ou de performance
- un éventuel constructeur, appelé à l’initialisation de l’objet

Une fois créé, un cache allouera des slabs à la demande, recyclera les objets libérés et pourra libérer des slabs vides si la pression mémoire augmente

---

> Cache

#### Organisation interne:

Un cache maintient plusieurs listes de slabs, classées selon leur état, full, partial ou empty. Lorsqu’une allocation est demandée, le cache cherche d’abord un slab partiellement rempli, si aucun n’est disponible, il en utilise un vide, voire en alloue un nouveau si nécessaire. Cette organisation hiérarchique permet d’optimiser le temps d’allocation, de favoriser la réutilisation des objets et de maîtriser la consommation mémoire globale.

---

> Cache

#### Spécificités

**1. Un cache par type d’objet** ***(TRES IMPORTANT)**

Chaque cache correspond à un type précis d’objet kernel.
Ce qui garantit :

- une homogénéité totale des objets
- une gestion simplifiée
- une forte prédictibilité du comportement mémoire

Cette propriété est fondamentale pour le fonctionnement du slab allocator, mais elle est également déterminante du point de vue de l’exploitation.

**2. Caches génériques (`kmalloc-*`)**

En plus des caches dédiés à des structures précises, Linux fournit des caches génériques , par exemple: 

* `kmalloc-8`
* `kmalloc-16`
* `kmalloc-32`
* …
* `kmalloc-4096`

Ces caches servent aux allocations dynamiques qui ne nécessitent pas de type strict, à la création de buffers temporaires, ainsi qu’à certaines structures internes plus flexibles. Ils occupent une place centrale dans de nombreuses vulnérabilités, car ils peuvent provoquer des confusions de type entre objets de tailles compatibles.

**3. Flags et options de sécurité**

Les caches peuvent être configurés avec différents flags qui influencent leurs performances, leur comportement mémoire et leur niveau de durcissement. Parmi ces options, on trouve par exemple les redzones, le poisoning, le freelist hardening ou encore les vérifications de débordement. Ces mécanismes renforcent la sécurité et compliquent l’exploitation d’erreurs, sans pour autant éliminer totalement les vulnérabilités liées au slab allocator.


**4. Réutilisation et prévisibilité**

Comme pour les slabs, les caches favorisent une réutilisation rapide des objets, lorsqu’un objet est libéré, il retourne dans le cache, et une nouvelle allocation peut parfois récupérer exactement la même adresse. Ce fonctionnement rend les comportements mémoire répétables, mesurables et donc potentiellement exploitables dans certains contextes de sécurité.

---

> Cache

#### Lien avec les vulnérabilités


Le cache constitue souvent le point d’entrée principal dans les scénarios d’exploitation du noyau Linux. En cas de vulnérabilité de type use-after-free, c’est le cache qui détermine quel type d’objet peut être réalloué. Un attaquant peut ainsi tenter de forcer la réallocation d’un objet qu’il contrôle, ce qui fait de la taille et de la structure du cache des paramètres critiques. Les caches `kmalloc-*` sont particulièrement intéressants à ce titre, puisqu’ils peuvent héberger simultanément des objets de nature différente mais de taille identique, ouvrant la voie à des confusions de type particulièrement puissantes.

De plus, le contrôle du remplissage des caches, qui une technique connue sous le nom de "heap grooming" permet d’influencer la disposition des objets en mémoire, de synchroniser des allocations concurrentes et de rendre des primitives d’exploitation à la fois plus fiables et plus reproductibles.


---
> Cache

#### Conclusion

Le cache constitue la couche de gestion la plus élevée du slab allocator. Il définit les règles, les politiques et les contraintes associées à un type d’objet kernel donné, tout en orchestrant l’utilisation des slabs sous-jacents.

Si les objets kernel représentent la cible et les slabs le conteneur physique, le cache est le **chef d’orchestre** de l’allocation mémoire. Sa compréhension est indispensable pour analyser les performances du noyau Linux, mais surtout pour comprendre et exploiter les vulnérabilités liées à la gestion du heap kernel.

### <ins>Le slab allocator</ins>

> Slab allocator

#### Rôle et objectifs:

Le slab allocator est le mécanisme principal utilisé par le noyau Linux pour gérer l’allocation dynamique des objets kernel. Il repose sur les concepts introduits précédemment, à savoir les objets kernel, les slabs et les caches, afin de proposer un modèle d’allocation spécifiquement adapté aux contraintes du noyau.

Contrairement aux allocateurs génériques, le slab allocator part du constat que le noyau manipule en majorité des objets de taille fixe, fortement typés et alloués de manière répétée. L’objectif est donc de rendre ces allocations aussi rapides et peu coûteuses que possible, tout en limitant la fragmentation mémoire et en améliorant la localité cache.

---

> Slab allocator

#### Vue d’ensemble du fonctionnement:

Le fonctionnement du slab allocator peut être vu comme une chaîne hiérarchique. Lorsqu’une allocation est demandée, le noyau commence par identifier le cache correspondant au type ou à la taille de l’objet souhaité. Ce cache va ensuite sélectionner le slab approprié, à partir duquel un objet libre est extrait et retourné à celui l'ayant appelé.

Le chemin suivi est toujours le même la requête passe par le cache, puis par un slab, avant d’aboutir à un objet kernel. Cette organisation permet de séparer clairement les responsabilités et d’optimiser chaque niveau de la gestion mémoire.

---

> Slab allocator

#### Allocation d’un objet:

Lorsqu’un objet kernel est requis, par exemple via `kmem_cache_alloc` ou `kmalloc`, le slab allocator commence par choisir le cache adapté. Dans le cas d’un objet fortement typé, il s’agira d’un cache dédié, tandis que les allocations plus génériques utiliseront un cache de type `kmalloc-*`.

Une fois le cache sélectionné, celui-ci cherche un slab capable de fournir un objet libre. En pratique, un slab partiellement occupé est privilégié, car il permet de réutiliser de la mémoire déjà active. Si aucun slab adéquat n’est disponible, un nouveau slab est alors créé, généralement à partir de pages mémoire fournies par l’allocateur de pages.

L’objet libre est ensuite extrait de la freelist du slab et marqué comme alloué. Selon la configuration du cache, certaines étapes supplémentaires peuvent avoir lieu, comme l’appel d’un constructeur ou l’application de mécanismes de durcissement.

---

> Slab allocator

#### Libération d’un objet:

La libération d’un objet suit une logique symétrique. Lorsque `kfree` ou `kmem_cache_free` est appelé, le slab allocator identifie le cache auquel appartient l’objet, puis le slab précis dans lequel il se trouve. L’objet est alors replacé dans la freelist du slab, et l’état de ce dernier est mis à jour.

Dans la majorité des cas, la mémoire associée à l’objet n’est pas immédiatement rendue au système. L’objet reste présent dans le slab et peut être réalloué très rapidement. Cette approche permet d’éviter des opérations coûteuses sur les pages mémoire, mais implique également que le contenu de l’objet peut persister après sa libération.

---

> Slab allocator

#### Réutilisation et performances:

La réutilisation rapide des objets est l’un des principaux atouts du slab allocator. En conservant les objets récemment libérés à portée immédiate, le noyau réduit drastiquement le coût des allocations répétées. De plus, comme les objets sont regroupés de manière contiguë au sein des slabs, ils bénéficient souvent d’une excellente localité cache.

Ce modèle est particulièrement efficace pour les structures fortement sollicitées, comme les fichiers ouverts, les sockets ou les structures réseau. Dans ces cas, le slab allocator permet d’atteindre des performances difficilement réalisables avec un allocateur plus généraliste.

---

> Slab allocator

#### Modèle mémoire et prévisibilité:

En organisant la mémoire autour de caches spécialisés et de slabs homogènes, le slab allocator introduit un modèle mémoire relativement stable et prévisible. Les allocations et libérations successives ont tendance à suivre des schémas répétitifs, notamment lorsque les mêmes types d’objets sont utilisés en boucle.

Cette prévisibilité est un avantage du point de vue des performances, mais elle a également des conséquences importantes en matière de sécurité. En observant et en contrôlant l’ordre des allocations, il devient possible d’influencer la disposition des objets en mémoire kernel.

---

> Slab allocator

#### Lien avec la sécurité:

De nombreuses vulnérabilités du noyau Linux impliquant le heap reposent directement sur le fonctionnement du slab allocator. Les scénarios de type use-after-free, double free ou confusion de type exploitent souvent la capacité du slab allocator à réallouer rapidement un objet à une adresse déjà utilisée.

Dans ce contexte, comprendre quel cache est utilisé, comment les slabs sont remplis et dans quel ordre les objets sont réutilisés devient essentiel. Même si des mécanismes de durcissement ont été ajoutés au fil du temps, le modèle fondamental du slab allocator reste inchangé et continue de jouer un rôle central dans l’exploitation des failles mémoire.

---

> Slab allocator

#### Implémentations dans Linux:

Il existe plusieurs implémentations du modèle slab dans Linux, notamment SLAB, SLUB et SLOB. Bien que leurs détails internes diffèrent, elles reposent toutes sur les mêmes principes généraux et proposent une organisation similaire de la mémoire kernel.

Dans la pratique, la majorité des systèmes Linux modernes utilisent SLUB, qui simplifie certaines structures internes et améliore les performances, tout en conservant le même modèle cache–slab–objet.

---

> Slab allocator

#### Conclusion:

Le slab allocator constitue le cœur de la gestion mémoire dynamique des objets kernel dans Linux. En exploitant la nature répétitive et typée de ces objets, il permet d’obtenir des performances élevées et une organisation mémoire efficace.

Cependant, cette efficacité repose sur des choix de conception qui introduisent une forte prévisibilité et une réutilisation agressive de la mémoire. Ces caractéristiques, bien que bénéfiques du point de vue des performances, sont également à l’origine de nombreuses vulnérabilités du noyau. Comprendre le fonctionnement du slab allocator est donc indispensable pour analyser à la fois les performances du système et les mécanismes d’exploitation du heap kernel.



## Points importants et contraintes 

`Rappelez les contraintes de typages et continuez d'investiguer sur plus de détail`

## Use-cases et utilité

`Diger la partie poerformance avec des chiffres, parler de son importance pour les exploits`

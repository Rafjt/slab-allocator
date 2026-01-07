# Notes importantes/utiles  

## First look
Le slab allocator à un lien direct avec les performance de la machine ou il se trouve.  

De ce que je comprends, il s'agit d'un mécanisme d'allocation qui, au lieu de supprimer l'espace mémoire après allocation, garde en cache le slab (emplacement mémoire correspondant à une donné précise ou un type de donnnées je crois ?) correspondant à cette data afin d'éviter les drop de performance du à la suppression mais aussi à améliorer la rapidité

___
#### Définitions (source: [Slab allocation Wikipedia](https://en.wikipedia.org/wiki/Slab_allocation)):

- **Cache**: cache represents a small amount of very fast memory. A cache is a storage for a specific type of object, such as semaphores, process descriptors, file objects, etc.

- **Slab**: slab represents a contiguous piece of memory, usually made of several virtually contiguous pages. The slab is the actual container of data associated with objects of the specific kind of the containing cache.
___

 .

#### Une ***slab*** peut prendre 3 états:

- empty – all objects on a slab marked as free
- partial – slab consists of both used and free objects
- full – all objects on a slab marked as used

#### Workflow
---
- Initialement, toutes les slab sont marqué comme empty.
- À l'apparition d'un nouveau ***"kernel object"*** le slab allocator va chercher une slab en état **partial** avec un cache du même type que l'objet pour pouvoir le stocker.
- Si il le trouve il fait l'allocation.
- Sinon, le slab allocator va créer une nouvelle slab dasn l'état **empty** pour l'associer à un cache avec le bon type et stocker l'objet.

## [Vulgarisation du fonctionnement en vidéo](https://www.youtube.com/watch?v=rsp0rBP61As)

### Objet kernel

- À ne pas oublier :

 - Taille fixe

 - Typage fort (struct précise)

 - Alignement

 - Réutilisation après free

 - Lien avec vulnérabilités (UAF, type confusion)

 ### Slab

- Ensemble de pages physiques

- États (full / partial / free)

- Métadonnées (selon SLAB / SLUB)

- Relation avec le cache

- Localité mémoire

### Cache

- Un cache = un type d’objet

- kmem_cache_create

- aramètres importants (size, flags, ctor)

- Caches génériques (kmalloc-*)

- Rôle central en exploitation

### Le slab allocator

- Création du cache

- Allocation (kmem_cache_alloc)

- Libération (kmem_cache_free)

- Réutilisation d’un objet

- (Optionnel) Différences SLAB / SLUB / SLOB -> mais que si on a du temps vue qu'on est censé ce focus linux 

### Points importants et contraintes

- Typage strict

- Pas d’allocations arbitraires

- Effets des flags (redzone, poisoning, freelist hardening)

- Impact KASAN / CONFIG options

### Questions -> Réponses
- Est-ce que le cache référence les slab comme une sorte de pointeur ? 
Plus ou moins, une slab represente une collections de pages continue, et le cache représente une collection de slabs DONC:
    - Si on un **S1** et **S2** qui sont 2 slabs distinctes, ces dernières peuvent être rassembler dans le cache on aura, **C = S1 + S2**
Par contre un cache ne peut stockerqu'un type de kernel object (fd, semaphores, file objects)

- C'est quoi la fragmentation et dans quelle mesure est ce que la slab allocation l'empeche ?


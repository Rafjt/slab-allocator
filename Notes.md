# Notes importantes/utiles  

## First look
Le slab allocator à un lien direct avec les performance de la machine ou il se trouve.  

De ce que je comprends, il s'agit d'un mécanisme d'allocation qui, au lieu de supprimer l'espace mémoire après allocation, garde en cache le slab (emplacement mémoire correspondant à une donné précise ou un type de donnnées je crois ?) correspondant à cette data afin d'éviter les drop de performance du à la suppression mais aussi à améliorer la rapidité


### Définitions (source: [Slab allocation Wikipedia](https://en.wikipedia.org/wiki/Slab_allocation))
**Cache**: cache represents a small amount of very fast memory. A cache is a storage for a specific type of object, such as semaphores, process descriptors, file objects, etc.

**Slab**: slab represents a contiguous piece of memory, usually made of several virtually contiguous pages. The slab is the actual container of data associated with objects of the specific kind of the containing cache.

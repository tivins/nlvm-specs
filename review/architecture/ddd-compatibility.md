# NL et Domain-Driven Design — Analyse de compatibilité

Ce document analyse si les projets utilisant NLVM peuvent respecter les principes du Domain-Driven Design (DDD). Il s'appuie sur les spécifications du langage NL (specs.md, compiler.md, stdlib.md, vm.md) et le suivi de cohérence (review/coherence.md).

---

## Verdict global

**Oui, les projets NLVM peuvent respecter les principes DDD.** Le langage NL fournit les briques essentielles. Certains aspects nécessitent des conventions ou des patterns à implémenter manuellement.

---

## Points forts pour DDD

| Principe DDD | Support NL | Détails |
|--------------|------------|---------|
| **Value Objects** | Très bon | `readonly` (classe ou propriété), `ValueEquatable`, `const` méthodes/paramètres. Immutabilité et égalité structurelle bien supportées. |
| **Entities** | Bon | Classes avec identité (référence), constructeurs, encapsulation via `private`/`protected`. |
| **Aggregates** | Bon | Encapsulation, invariants dans les constructeurs. Pas de notion native d'« aggregate root », mais c'est un pattern, pas une contrainte du langage. |
| **Domain Services** | Bon | Classes stateless ou méthodes `static`. |
| **Repositories** | Bon | Interfaces pour abstraire la persistance. Ex. : `interface IOrderRepository { Order|null findById(int id); void save(Order o); }`. |
| **Bounded Contexts** | Bon | Namespaces alignés avec la hiérarchie de fichiers (`namespace com.example.order.domain`). Un fichier = une classe. |
| **Ubiquitous Language** | Bon | `typedef` pour types du domaine, nommage explicite. |
| **Infrastructure isolation** | Bon | Le domaine dépend d'interfaces ; l'infrastructure les implémente. |
| **Application Services** | Bon | Classes qui orchestrent les use cases. |

---

## Points à gérer par convention ou implémentation

| Aspect | Situation | Recommandation |
|--------|-----------|----------------|
| **Dependency Injection** | Pas de DI intégré | Injection manuelle via constructeurs. Un point d'entrée (composition root) construit les dépendances et les passe aux services. |
| **Domain Events** | Pas de mécanisme dédié | Implémenter un bus d'événements simple (liste d'abonnés, dispatch synchrone) ou un pattern similaire. |
| **Interfaces multiples** | VM supporte plusieurs interfaces, la syntaxe n'est pas clairement documentée | Cohérence mentionnée dans review/coherence.md (II-7). En pratique, une syntaxe du type `implements A, B, C` est probable. |
| **Contrôle d'accès entre modules** | Pas de notion de module avec visibilité | Organiser par namespaces et conventions (ex. `domain` ne dépend pas de `infrastructure`). |

---

## Exemple de structure DDD en NL

```nl
// domain/Order.nl — Entity
namespace com.example.order.domain;
class Order {
    private int id;
    private OrderStatus status;
    public construct(int id, OrderStatus status) { ... }
}

// domain/OrderStatus.nl — Value Object (enum)
namespace com.example.order.domain;
enum OrderStatus { Pending, Confirmed, Shipped }

// domain/IOrderRepository.nl — Port (interface)
namespace com.example.order.domain;
interface IOrderRepository {
    public Order|null findById(int id);
    public void save(const Order order);
}

// application/PlaceOrderService.nl — Application Service
namespace com.example.order.application;
use com.example.order.domain.*;
class PlaceOrderService {
    private IOrderRepository repo;
    public construct(IOrderRepository repo) { this.repo = repo; }
    public void execute(PlaceOrderCommand cmd) { ... }
}

// infrastructure/FileOrderRepository.nl — Adapter
namespace com.example.order.infrastructure;
use com.example.order.domain.*;
class FileOrderRepository implements IOrderRepository { ... }
```

---

## Synthèse

| Critère | Note | Commentaire |
|---------|------|-------------|
| Modélisation du domaine | 9/10 | Value objects, entities, enums, interfaces. |
| Séparation des couches | 8/10 | Interfaces + namespaces suffisants. |
| Inversion de dépendances | 7/10 | Possible via constructeurs, sans framework. |
| Bounded contexts | 8/10 | Namespaces adaptés. |
| Événements de domaine | 5/10 | À implémenter manuellement. |

---

## Conclusion

NL est adapté à DDD. Les projets devront définir des conventions (couches, composition root, éventuellement un bus d'événements léger), mais le langage offre les bases nécessaires. L'absence de DI intégré n'est pas bloquante : l'injection par constructeur manuelle est une approche courante en DDD.

<hr>

## Explication détaillée

### Value Objects

En DDD, un Value Object est défini par ses attributs, non par une identité. Deux Value Objects avec les mêmes valeurs sont interchangeables. NL supporte ce pattern de plusieurs façons :

- **`readonly` sur une classe** : toutes les propriétés deviennent immuables après construction. Aucune modification n'est possible une fois l'objet créé.
- **`readonly` sur une propriété** : seule cette propriété est figée après le constructeur.
- **`ValueEquatable`** : l'interface impose `valueEquals(const Self|null other)` et `valueHash()`. L'égalité structurelle (mêmes valeurs) remplace l'égalité par référence. Idéal pour les clés de `system.Map` basées sur la valeur.
- **`const`** : sur les méthodes (pas de mutation de l'objet) et les paramètres (pas de modification du paramètre). Permet de garantir qu'un Value Object passé en argument ne sera pas altéré.

Les enums typés (`enum OrderStatus : string`) constituent aussi des Value Objects légers pour des ensembles fermés de valeurs.

### Entities et identité

Une Entity en DDD possède une identité stable qui la distingue des autres, même si ses attributs changent. En NL, les instances de classe ont une identité par référence : `a == b` compare les références, pas les valeurs. L'Entity conserve donc naturellement son identité à travers les mutations. L'encapsulation (`private`, `protected`) permet de protéger l'état et d'imposer des invariants dans les méthodes publiques.

### Aggregates et Aggregate Roots

Un Aggregate est un cluster d'objets du domaine dont les frontières de cohérence sont garanties par un Aggregate Root. NL ne fournit pas de concept dédié, mais le pattern est réalisable :

- L'Aggregate Root est une classe qui expose les opérations de modification.
- Les entités internes sont `private` ou `protected` ; seules les méthodes du Root permettent d'y accéder.
- Les invariants sont vérifiés dans le constructeur et dans les méthodes qui modifient l'état.
- Le Root peut être `final` pour éviter des sous-classes qui contourneraient les invariants.

### Repositories et ports/adapters

Le pattern Repository abstrait la persistance derrière une interface. En NL, on définit une interface dans le domaine (port) et une implémentation concrète dans l'infrastructure (adapter). Le domaine ne dépend que de l'interface ; l'infrastructure dépend du domaine et implémente l'interface. L'inversion de dépendances est ainsi respectée sans framework.

L'Application Service reçoit le Repository via son constructeur. Le point d'entrée (`main`) ou une factory construit les implémentations concrètes et les injecte. C'est le pattern « composition root ».

### Bounded Contexts et namespaces

Un Bounded Context en DDD délimite un modèle et un langage ubiquitaire cohérents. En NL, les namespaces reflètent la hiérarchie de répertoires : `com.example.order.domain`, `com.example.inventory.domain`, etc. Chaque contexte peut avoir son propre sous-modèle : `order.domain`, `order.application`, `order.infrastructure`. Les imports (`use`) rendent explicites les dépendances entre contextes. Une convention stricte (ex. « le domaine ne dépend jamais de l'application ou de l'infrastructure ») peut être appliquée par revue de code ou par outils d'analyse.

### Langage ubiquitaire

Le langage ubiquitaire est le vocabulaire partagé entre experts métier et développeurs. En NL, les `typedef` permettent de nommer des types du domaine :

```nl
typedef string Email;
typedef int CustomerId;
```

Les noms de classes, méthodes et propriétés reflètent le métier. Les enums capturent des états ou des valeurs métier explicites.

### Domain Events

Les Domain Events signalent qu'un fait métier s'est produit. NL n'a pas de mécanisme natif (type `event`, bus d'événements). Une implémentation manuelle consiste à :

- Définir une interface `IDomainEvent` ou une classe de base d'événement.
- Créer un registre d'handlers (liste de callbacks ou d'objets).
- Dispatcher les événements de manière synchrone après une opération du domaine.

Les closures et les interfaces permettent de passer des handlers sans difficulté.

### Ce que NL n'apporte pas (et ce qu'il faut compenser)

- **DI** : pas de conteneur ou d'injection automatique. La composition root est manuelle.
- **Module-level visibility** : pas de « package private » ou de modules avec visibilité explicite. Les conventions et la discipline de l'équipe compensent.
- **RAII / try-with-resources** : le nettoyage des ressources repose sur les destructeurs (timing non garanti) ou les blocs `try/finally`. Un pattern explicite de libération peut être défini dans le domaine.

### Références aux spécifications

Les points ci-dessus s'appuient sur : specs.md (§ Classes, § Visibility, § Readonly, § ValueEquatable, § Extends/Implements, § Imports), stdlib.md (§ Core interfaces), compiler.md (vérifications de visibilité et de types), vm.md (format des modules et représentation des objets).

Anatomy of Rdo as of H5 / H6 unnamespaced and details of the incompatible H6 namespaced version.


Classic RDO anatomy

Horde_Rdo       Constants

Horde_Rdo_Base  Base class for entities
   * Implements ArrayAccess
   * has mapper reference
   * has array of fields
   * can be initialized with key-value maps
   * has a clone method which unsets the primary key
   * has a magic method setter, falling back to a setFieldName method if it exists
       CRITIQUE:
         Makes the object RW in non-obvious ways
   * has a magic isset for fields and relations
   * has magic method getter
      * returns field values unless a getFieldName method is defined
      * Looks for lazy fields on first access. SQL lookup is internal
      * Looks up relations
   * implements IteratorAggregate through Horde_Rdo_Iterator. Looping over the object goes through each property
   * has add/remove relation methods
      CRITIQUE:
         Makes the object RW in non-obvious ways
   * has a save proxy for the mapper's update() method
      CRITIQUE:
         Handy if you only ever intend to use Rdo, but not useful otherwise
         Also, see mapper->update for how it is quirky
   * has a toArray(lazy, relationships) method 
      CRITIQUE:
         -> handy, but gets in the way if you have an own idea how your entity should resolve to array
         -> exposes db table keys which might not be what you want

Horde_Rdo_Factory   An optional, injector-like (but incompatible) sibling mapper provider used mostly when dealing with relations.
    CRITIQUE: 
       very limited/no support for different adapters for read and write operations
       very limitied/no support for different DBs for subsystems (like having resource_a related tables in a different schema or host than resource_b related)



Horde_Rdo_Iterator  An iterator hardcoded into Horde_Rdo_Base

Horde_Rdo_Mapper    Base class for per-table ORM mapper classes
   Features/Behaviours
      Defaults to guessing entity class name
      Defaults to introspecting tables for entity fields
      Automatically works on creation and update fields
      Very explicitly SQLish
      No interface
      Optional relation to Horde_Rdo_Factory
      Optional caching of table structure
      Tricky to implement subtyping of entities
      Eager relations based on JOIN
      Lazy relations based on SELECT
      Little/bad handling of inconsistent relations
      has an exists method for queries or querylikes
      has a create method which eats plain arrays (no translation of keys)
        Inline Insert SQL builder
      has an update method with multiple calling conventions
         update will not CREATE missing objects which has a key/id value
      little/hacky many-to-many support
      has a delete method with multiple calling conventions
      has a find() method with multiple calling conventions
        Find always returns a collection even if empty
        Find delegates SQL to Horde_Rdo_Query and returns a Horde_Rdo_List unconditionally
      has addRelation/removeRelation implemented internally
      has a findOne method with multiple calling convention
          findOne on an emptyish value gives you any/first object
          findOne with a non-empty key value will create a Horde_Rdo_Query
      has a sortBy method which accepts SQLisms (can be overridden at query level)


Horde_Rdo_Query A class representing a query
      No Interface
      Create accepts multiple formats
         Create can clone existing queries for modification
         Create understands primitives as single primary keys
         Create understands associative arrays as ANDed property constraints
      setFields allows to limit fields to query for
      addFields allows to read more fields (preload lazy fields in specific contexts)
      combineWith switches between AND and OR
      addTest allows iteratively adding test conditions
      sortBy, clearSort define sort query snippets
      limit(limit, offset) for paging/subsets
      getQuery build a literal SQL query and bind parameters



H6 redesign

Guiding thoughts:
  Interfaces first
  Autowire-friendly constructors
  Use separate methods for separate purposes
  Do away with super powerful accept-anything interfaces
  BC break quirky behaviours like save() -> update, but only create if key missing
  Default objects are less chatty. The mapper is not a pseudo repo and the default value object comes with much less exposing magic
  No strict need for inheritance, most implementation in traits with accompanying interfaces
  Default value object "just works" without inheritance
  Some way to seemlessly integrate SQL backed and non-SQL backed repositories
  Some proper many-to-many handling?
  Rethink Horde_Rdo_Factory - should this be a compatible of horde\injector? should it be able to yield other repos?
  Forbid any propagation of the internal db object
  more test/mock friendly?




Horde_Rdo -> Horde\Rdo\Constants
verbatim transfer

Horde_Rdo_Base -> Horde\Rdo\RampageObject, Horde\Rdo\EntityInterface, Horde\Rdo\RampageInterface
    RampageObject -> Default implementation of RampageInterface
    EntityInterface -> Minimal interface for any entity objects wrapping ValueObject

Horde_Rdo_Mapper -> Horde\Rdo\Repository, Horde\Rdo\Reader, Horde\Rdo\Writer

How to migrate my Horde_Rdo_Base subtypes?
a) Just don't subtype, use plain RampageObjects
b) subtype or RampageObject or implement RampageInterface
c) Redesign your entity class to implement EntityInterface (use trait ... )

How to migrate my Horde_Rdo_Mapper subtypes?

a) Just don't subtype, use the class as-is and feed it parameters
b) Create a wrapping repo class 

How to migrate find() and findOne()?
Use the appropriate findBy and findOneBy methods (by key, byQuery)

How to substitute find()ing by hash?
Build a Query from the Hash and use findByQuery()

How to subsitute create() and update()?
Create() will only accept RampageInterface objects and will throw exception if a key is present and incompatible
CreateFromHash will create a default object from the hash and will call create with that object.
Update() will only accept RampageInterface objects with key. No implicit creation.
CreateOrUpdate will update if present, create if update returned 0 objects changed. (guaranteed behavior is only the object will be present afterwards)

How about schema and identifiers?

The new Rdo is designed with unique string keys in mind. Database items will often use a numeric, unique surrogate key as primary key (even if it doesn't show up in the schema)
Rdo supports creating unique identifiers (GUID, UUID, LDAP dn or otherwise) in the requesting objects. 

It's best to have both an explicit numeric identifier and a field for collision-free, client-supplied strings, usually called "uuid". 
The numeric identifier should not be exposed to the outside world or given any meaning.

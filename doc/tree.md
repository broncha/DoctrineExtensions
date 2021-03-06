# Tree - Nestedset behavior extension for Doctrine 2

**Tree** nested behavior will implement the standard Nested-Set behavior
on your Entity. Tree supports different strategies. Currently it supports
**nested-set**, **closure-table** and **materialized-path**. Also this behavior can be nested
with other extensions to translate or generated slugs of your tree nodes.

Features:

- Materialized Path strategy for ORM and ODM (MongoDB)
- Closure tree strategy, may be faster in some cases where ordering does not matter
- Support for multiple roots in nested-set
- No need for other managers, implementation is through event listener
- Synchronization of left, right values is automatic
- Can support concurrent flush with many objects being persisted and updated
- Can be nested with other extensions
- Annotation, Yaml and Xml mapping support for extensions

[blog_reference]: http://gediminasm.org/article/tree-nestedset-behavior-extension-for-doctrine-2 "Tree - Nestedset or Closure extension for Doctrine 2 makes tree implementation on entities"
[blog_test]: http://gediminasm.org/test "Test extensions on this blog"

Thanks for contributions to:

- **[comfortablynumb](http://github.com/comfortablynumb) Gustavo Falco** for Closure strategy
- **[everzet](http://github.com/everzet) Kudryashov Konstantin** for TreeLevel implementation
- **[stof](http://github.com/stof) Christophe Coevoet** for getTreeLeafs function

Update **2012-02-23**

- Added a new strategy to support the "Materialized Path" tree model. It works with ODM (MongoDB) and ORM.

Update **2011-05-07**

- Tree is now able to act as **closure** tree, this strategy was refactored
and now fully functional. It is much faster for file-folder trees for instance
where you do not care about tree ordering.

Update **2011-04-11**

- Made in memory node synchronization, this change does not require clearing the cached nodes after any updates
to nodes, except **recover, verify and removeFromTree** operations.

Update **2011-02-08**

- Refactored to support multiple roots
- Changed the repository name, relevant to strategy used
- New [annotations](#annotations) were added


Update **2011-02-02**

- Refactored the Tree to the ability on supporting diferent tree models
- Changed the repository location in order to support future updates

**Note:**

- You can [test live][blog_test] on this blog
- After using a NestedTreeRepository functions: **verify, recover, removeFromTree** it is recommended to clear EntityManager cache
because nodes may have changed values in database but not in memory. Flushing dirty nodes can lead to unexpected behaviour.
- Closure tree implementation is experimental and not fully functional, so far not documented either
- Public [Tree repository](http://github.com/l3pp4rd/DoctrineExtensions "Tree extension on Github") is available on github
- Last update date: **2012-02-23**

**Portability:**

- **Tree** is now available as [Bundle](http://github.com/stof/StofDoctrineExtensionsBundle)
ported to **Symfony2** by **Christophe Coevoet**, together with all other extensions

This article will cover the basic installation and functionality of **Tree** behavior

Content:
    
- [Including](#including-extension) the extension
- Tree [annotations](#annotations)
- Entity [example](#entity-mapping)
- [Yaml](#yaml-mapping) mapping example
- [Xml](#xml-mapping) mapping example
- Basic usage [examples](#basic-examples)
- Build [html tree](#html-tree)
- Advanced usage [examples](#advanced-examples)
- [Materialized Path](#materialized-path)

<a name="including-extension"></a>

## Setup and autoloading

Read the [documentation](http://github.com/l3pp4rd/DoctrineExtensions/blob/master/doc/annotations.md#em-setup)
or check the [example code](http://github.com/l3pp4rd/DoctrineExtensions/tree/master/example)
on how to setup and use the extensions in most optimized way.

<a name="entity-mapping"></a>

## Tree Entity example:

**Note:** that Node interface is not necessary, except in cases there
you need to identify entity as being Tree Node. The metadata is loaded only once then
cache is activated

``` php
<?php
namespace Entity;

use Gedmo\Mapping\Annotation as Gedmo;
use Doctrine\ORM\Mapping as ORM;

/**
 * @Gedmo\Tree(type="nested")
 * @ORM\Table(name="categories")
 * use repository for handy tree functions
 * @ORM\Entity(repositoryClass="Gedmo\Tree\Entity\Repository\NestedTreeRepository")
 */
class Category
{
    /**
     * @ORM\Column(name="id", type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue
     */
    private $id;

    /**
     * @ORM\Column(name="title", type="string", length=64)
     */
    private $title;

    /**
     * @Gedmo\TreeLeft
     * @ORM\Column(name="lft", type="integer")
     */
    private $lft;
    
    /**
     * @Gedmo\TreeLevel
     * @ORM\Column(name="lvl", type="integer")
     */
    private $lvl;
    
    /**
     * @Gedmo\TreeRight
     * @ORM\Column(name="rgt", type="integer")
     */
    private $rgt;
    
    /**
     * @Gedmo\TreeRoot
     * @ORM\Column(name="root", type="integer", nullable=true)
     */
    private $root;
    
    /**
     * @Gedmo\TreeParent
     * @ORM\ManyToOne(targetEntity="Category", inversedBy="children")
     * @ORM\JoinColumn(name="parent_id", referencedColumnName="id", onDelete="SET NULL")
     */
    private $parent;
    
    /**
     * @ORM\OneToMany(targetEntity="Category", mappedBy="parent")
     * @ORM\OrderBy({"lft" = "ASC"})
     */
    private $children;

    public function getId()
    {
        return $this->id;
    }
    
    public function setTitle($title)
    {
        $this->title = $title;
    }

    public function getTitle()
    {
        return $this->title;
    }
    
    public function setParent(Category $parent = null)
    {
        $this->parent = $parent;    
    }
    
    public function getParent()
    {
        return $this->parent;   
    }
}
```

<a name="annotations"></a>

### Tree annotations:

- **@Gedmo\Mapping\Annotation\Tree(type="strategy")** this **class annotation** is used to set the tree strategy by **type** parameter.
Currently **nested**, **closure** or **materializedPath** strategies are supported. An additional "activateLocking" parameter
is available if you use the "Materialized Path" strategy with MongoDB. It's used to activate the locking mechanism (more on that
in the corresponding section).
- **@Gedmo\Mapping\Annotation\TreeLeft** it will use this field to store tree **left** value
- **@Gedmo\Mapping\Annotation\TreeRight** it will use this field to store tree **right** value
- **@Gedmo\Mapping\Annotation\TreeParent** this will identify this column as the relation to **parent node**
- **@Gedmo\Mapping\Annotation\TreeLevel** it will use this field to store tree**level**
- **@Gedmo\Mapping\Annotation\TreeRoot** it will use this field to store tree**root** id value
- **@Gedmo\Mapping\Annotation\TreePath** (Materialized Path only) it will use this field to store the "path". It has an
optional parameter "separator" to define the separator used in the path
- **@Gedmo\Mapping\Annotation\TreePathSource** (Materialized Path only) it will use this field as the source to
 construct the "path"
- **@Gedmo\Mapping\Annotation\TreeLockTime** (Materialized Path - ODM MongoDB only) this field is used if you need to
use the locking mechanism with MongoDB. It persists the lock time if a root node is locked (more on that in the corresponding
section).

<a name="yaml-mapping"></a>

## Yaml mapping example

Yaml mapped Category: **/mapping/yaml/Entity.Category.dcm.yml**

```
---
Entity\Category:
  type: entity
  repositoryClass: Gedmo\Tree\Entity\Repository\NestedTreeRepository
  table: categories
  gedmo:
    tree:
      type: nested
  id:
    id:
      type: integer
      generator:
        strategy: AUTO
  fields:
    title:
      type: string
      length: 64
    lft:
      type: integer
      gedmo:
        - treeLeft
    rgt:
      type: integer
      gedmo:
        - treeRight
    root:
      type: integer
      nullable: true
      gedmo:
        - treeRoot
    lvl:
      type: integer
      gedmo:
        - treeLevel
  manyToOne:
    parent:
      targetEntity: Entity\Category
      inversedBy: children
      joinColumn:
        name: parent_id
        referencedColumnName: id
        onDelete: SET NULL
      gedmo:
        - treeParent
  oneToMany:
    children:
      targetEntity: Entity\Category
      mappedBy: parent
      orderBy:
        lft: ASC
```

<a name="xml-mapping"></a>

## Xml mapping example

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
                  xmlns:gedmo="http://gediminasm.org/schemas/orm/doctrine-extensions-mapping">

    <entity name="Mapping\Fixture\Xml\NestedTree" table="nested_trees">

        <indexes>
            <index name="name_idx" columns="name"/>
        </indexes>

        <id name="id" type="integer" column="id">
            <generator strategy="AUTO"/>
        </id>

        <field name="name" type="string" length="128"/>
        <field name="left" column="lft" type="integer">
            <gedmo:tree-left/>
        </field>
        <field name="right" column="rgt" type="integer">
            <gedmo:tree-right/>
        </field>
        <field name="root" type="integer">
            <gedmo:tree-root/>
        </field>
        <field name="level" column="lvl" type="integer">
            <gedmo:tree-level/>
        </field>

        <many-to-one field="parent" target-entity="NestedTree">
            <join-column name="parent_id" referenced-column-name="id" on-delete="SET_NULL"/>
            <gedmo:tree-parent/>
        </many-to-one>

        <gedmo:tree type="nested"/>

    </entity>

</doctrine-mapping>
```

<a name="basic-examples"></a>

## Basic usage examples:

### To save some **Categories** and generate tree:

``` php
<?php
$food = new Category();
$food->setTitle('Food');

$fruits = new Category();
$fruits->setTitle('Fruits');
$fruits->setParent($food);

$vegetables = new Category();
$vegetables->setTitle('Vegetables');
$vegetables->setParent($food);

$carrots = new Category();
$carrots->setTitle('Carrots');
$carrots->setParent($vegetables);

$this->em->persist($food);
$this->em->persist($fruits);
$this->em->persist($vegetables);
$this->em->persist($carrots);
$this->em->flush();
```

The result after flush will generate the food tree:

```
/food (1-8)
    /fruits (2-3)
    /vegetables (4-7)
        /carrots (5-6)
```

### Using repository functions

``` php
<?php
$repo = $em->getRepository('Entity\Category');

$food = $repo->findOneByTitle('Food');
echo $repo->childCount($food);
// prints: 3
echo $repo->childCount($food, true/*direct*/);
// prints: 2
$children = $repo->children($food);
// $children contains:
// 3 nodes
$children = $repo->children($food, false, 'title');
// will sort the children by title
$carrots = $repo->findOneByTitle('Carrots');
$path = $repo->getPath($carrots);
/* $path contains:
   0 => Food
   1 => Vegetables
   2 => Carrots
*/

// verification and recovery of tree
$repo->verify();
// can return TRUE if tree is valid, or array of errors found on tree
$repo->recover();
$em->clear(); // clear cached nodes
// if tree has errors it will try to fix all tree nodes

UNSAFE: be sure to backup before runing this method when necessary, if you can use $em->remove($node);
// which would cascade to children
// single node removal
$vegies = $repo->findOneByTitle('Vegitables');
$repo->removeFromTree($vegies);
$em->clear(); // clear cached nodes
// it will remove this node from tree and reparent all children

// reordering the tree
$food = $repo->findOneByTitle('Food');
$repo->reorder($food, 'title');
// it will reorder all "Food" tree node left-right values by the title
```

### Inserting node in different positions

``` php
<?php
$food = new Category();
$food->setTitle('Food');

$fruits = new Category();
$fruits->setTitle('Fruits');

$vegetables = new Category();
$vegetables->setTitle('Vegetables');

$carrots = new Category();
$carrots->setTitle('Carrots');

$treeRepository
    ->persistAsFirstChild($food)
    ->persistAsFirstChildOf($fruits, $food)
    ->persistAsLastChildOf($vegitables, $food)
    ->persistAsNextSiblingOf($carrots, $fruits);

$em->flush();
```

For more details you can check the **NestedTreeRepository** __call function

Moving up and down the nodes in same level:

Tree example:

```
/Food
    /Vegitables
        /Onions
        /Carrots
        /Cabbages
        /Potatoes
    /Fruits
```

Now move **carrots** up by one position

``` php
<?php
$repo = $em->getRepository('Entity\Category');
$carrots = $repo->findOneByTitle('Carrots');
// move it up by one position
$repo->moveUp($carrots, 1);
```

Tree after moving the Carrots up:

```
/Food
    /Vegitables
        /Carrots <- moved up
        /Onions
        /Cabbages
        /Potatoes
    /Fruits
```

Moving **carrots** down to the last position

``` php
<?php
$repo = $em->getRepository('Entity\Category');
$carrots = $repo->findOneByTitle('Carrots');
// move it down to the end
$repo->moveDown($carrots, true);
```

Tree after moving the Carrots down as last child:

```
/Food
    /Vegitables
        /Onions
        /Cabbages
        /Potatoes
        /Carrots <- moved down to the end
    /Fruits
```

**Note:** tree repository functions: **verify, recover, removeFromTree**. 
Will require to clear the cache of Entity Manager because left-right values will differ.
So after that use **$em->clear();** if you will continue using the nodes after these operations.

### If you need a repository for your TreeNode Entity simply extend it

``` php
<?php
namespace Entity\Repository;

use Gedmo\Tree\Entity\Repository\NestedTreeRepository;
    
class CategoryRepository extends NestedTreeRepository
{
    // your code here
}

// and then on your entity link to this repository

/**
 * @Gedmo\Tree(type="nested")
 * @Entity(repositoryClass="Entity\Repository\CategoryRepository")
 */
class Category implements Node
{
    //...
}
```

<a name="html-tree"></a>

## Create html tree:

### Retrieving whole tree as array

If you would like to load whole tree as node array hierarchy use:

``` php
<?php
$repo = $em->getRepository('Entity\Category');
$arrayTree = $repo->childrenHierarchy();
```

All node children will stored under **__children** key for each node.

### Retrieving as html tree

To load a tree as **ul - li** html tree use:

``` php
<?php
$repo = $em->getRepository('Entity\Category');
$htmlTree = $repo->childrenHierarchy(
    null, /* starting from root nodes */
    false, /* load all children, not only direct */
    true, /* render html */
    array(
        'decorate' => true,
        'representationField' => 'slug'
    )
);
```

### Customize html tree output

``` php
<?php
$repo = $em->getRepository('Entity\Category');
$options = array(
    'decorate' => true,
    'representationField' => 'slug',
    'rootOpen' => '<ul>',
    'rootClose' => '</ul>',
    'childOpen' => '<li>',
    'childClose' => '</li>',
    'nodeDecorator' => function($node) {
        return '<a href="/page/'.$node['slug'].'">'.$node[$field].'</a>';
    }
);
$htmlTree = $repo->childrenHierarchy(
    null, /* starting from root nodes */
    false, /* load all children, not only direct */
    true, /* render html */
    $options
);
```

### Generate own node list

``` php
<?php
$repo = $em->getRepository('Entity\Category');
$query = $entityManager
    ->createQueryBuilder()
    ->select('node')
    ->from('Entity\Category', 'node')
    ->orderBy('node.root, node.lft', 'ASC')
    ->where('node.root = 1')
    ->getQuery()
;
$options = array('decorate' => true);
$tree = $repo->buildTree($query->getArrayResult(), $options);
```

### Using routes in decorator, show only selected items, return unlimited levels items as 2 levels

```
$controller = $this;
        $tree = $root->childrenHierarchy(null,false,array('decorate' => true,
            'rootOpen' => function($tree) {
                if(count($tree) && ($tree[0]['lvl'] == 0)){
                        return '<div class="catalog-list">';
                }
            },
            'rootClose' => function($child) {
                if(count($child) && ($child[0]['lvl'] == 0)){
                                return '</div>';
                }
             },
            'childOpen' => '',
            'childClose' => '',
            'nodeDecorator' => function($node) use (&$controller) {
                if($node['lvl'] == 1) {
                    return '<h1>'.$node['title'].'</h1>';
                }elseif($node["isVisibleOnHome"]) {
                    return '<a href="'.$controller->generateUrl("wareTree",array("id"=>$node['id'])).'">'.$node['title'].'</a>&nbsp;';
                }
            }
        ));
```

<a name="advanced-examples"></a>

## Advanced examples:

### Nesting Translatatable and Sluggable extensions

If you want to attach **TranslationListener** also add it to EventManager after
the **SluggableListener** and **TreeListener**. It is important because slug must be generated first
before the creation of it`s translation.

``` php
<?php
$evm = new \Doctrine\Common\EventManager();
$treeListener = new \Gedmo\Tree\TreeListener();
$evm->addEventSubscriber($treeListener);
$sluggableListener = new \Gedmo\Sluggable\SluggableListener();
$evm->addEventSubscriber($sluggableListener);
$translatableListener = new \Gedmo\Translatable\TranslationListener();
$translatableListener->setTranslatableLocale('en_us');
$evm->addEventSubscriber($translatableListener);
// now this event manager should be passed to entity manager constructor
```

And the Entity should look like:

``` php
<?php
namespace Entity;

use Gedmo\Mapping\Annotation as Gedmo;
use Doctrine\ORM\Mapping as ORM;

/**
 * @Gedmo\Tree(type="nested")
 * @ORM\Entity(repositoryClass="Gedmo\Tree\Entity\Repository\NestedTreeRepository")
 */
class Category
{
    /**
     * @ORM\Column(name="id", type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue
     */
    private $id;

    /**
     * @Gedmo\Translatable
     * @Gedmo\Sluggable
     * @ORM\Column(name="title", type="string", length=64)
     */
    private $title;

    /**
     * @Gedmo\TreeLeft
     * @ORM\Column(name="lft", type="integer")
     */
    private $lft;
    
    /**
     * @Gedmo\TreeRight
     * @ORM\Column(name="rgt", type="integer")
     */
    private $rgt;
    
    /**
     * @Gedmo\TreeLevel
     * @ORM\Column(name="lvl", type="integer")
     */
    private $lvl;
    
    /**
     * @Gedmo\TreeParent
     * @ORM\ManyToOne(targetEntity="Category", inversedBy="children")
     * @ORM\JoinColumn(name="parent_id", referencedColumnName="id", onDelete="SET NULL")
     */
    private $parent;
    
    /**
     * @ORM\OneToMany(targetEntity="Category", mappedBy="parent")
     */
    private $children;
    
    /**
     * @Gedmo\Translatable
     * @Gedmo\Slug
     * @ORM\Column(name="slug", type="string", length=128)
     */
    private $slug;

    public function getId()
    {
        return $this->id;
    }
    
    public function getSlug()
    {
        return $this->slug;
    }
    
    public function setTitle($title)
    {
        $this->title = $title;
    }

    public function getTitle()
    {
        return $this->title;
    }
    
    public function setParent(Category $parent)
    {
        $this->parent = $parent;    
    }
    
    public function getParent()
    {
        return $this->parent;   
    }
}
```

Yaml mapped Category: **/mapping/yaml/Entity.Category.dcm.yml**

```
---
Entity\Category:
  type: entity
  repositoryClass: Gedmo\Tree\Entity\Repository\NestedTreeRepository
  table: categories
  gedmo:
    tree:
      type: nested
  id:
    id:
      type: integer
      generator:
        strategy: AUTO
  fields:
    title:
      type: string
      length: 64
      gedmo:
        - translatable
        - sluggable
    lft:
      type: integer
      gedmo:
        - treeLeft
    rgt:
      type: integer
      gedmo:
        - treeRight
    lvl:
      type: integer
      gedmo:
        - treeLevel
    slug:
      type: string
      length: 128
      gedmo:
        - translatable
        - slug
  manyToOne:
    parent:
      targetEntity: Entity\Category
      inversedBy: children
      joinColumn:
        name: parent_id
        referencedColumnName: id
        onDelete: SET NULL
      gedmo:
        - treeParent
  oneToMany:
    children:
      targetEntity: Entity\Category
      mappedBy: parent
```

**Note:** that using dql without object hydration, the nodes will not be
translated. Because the postLoad event never will be triggered

Now the generated treenode slug will be translated by Translatable behavior

Easy like that, any suggestions on improvements are very welcome

<a name="materialized-path"></a>

## Materialized Path

### Important notes before defining the schema

- If you use MongoDB you should activate the locking mechanism provided to avoid inconsistencies in cases where concurrent
modifications on the tree could occur. Look at the MongoDB example of schema definition to see how it must be configured.
- If your **TreePathSource** field is of type "string", then the primary key will be concatenated in the form: "value-id".
 This is to allow you to use non-unique values as the path source. For example, this could be very useful if you need to
 use the date as the path source (maybe to create a tree of comments and order them by date).
- **TreePath** field can only be of types: string, text
- **TreePathSource** field can only be of types: id, integer, smallint, bigint, string, int, float (I include here all the
variations of the field types, including the ORM and ODM for MongoDB ones).
- **TreeLockTime** must be of type "date" (used only in MongoDB for now).

### ORM Entity example (Annotations)

``` php
<?php

namespace Entity;

use Gedmo\Mapping\Annotation as Gedmo;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity(repositoryClass="Gedmo\Tree\Entity\Repository\MaterializedPathRepository")
 * @Gedmo\Tree(type="materializedPath")
 */
class Category
{
    /**
     * @ORM\Id
     * @ORM\GeneratedValue
     */
    private $id;

    /**
     * @Gedmo\TreePath
     * @ORM\Column(name="path", type="string", length=3000, nullable=true)
     */
    private $path;

    /**
     * @Gedmo\TreePathSource
     * @ORM\Column(name="title", type="string", length=64)
     */
    private $title;

    /**
     * @Gedmo\TreeParent
     * @ORM\ManyToOne(targetEntity="Category", inversedBy="children")
     * @ORM\JoinColumns({
     *   @ORM\JoinColumn(name="parent_id", referencedColumnName="id", onDelete="SET NULL")
     * })
     */
    private $parent;

    /**
     * @Gedmo\TreeLevel
     * @ORM\Column(name="lvl", type="integer", nullable=true)
     */
    private $level;

    /**
     * @ORM\OneToMany(targetEntity="Category", mappedBy="parent")
     */
    private $children;

    public function getId()
    {
        return $this->id;
    }

    public function setTitle($title)
    {
        $this->title = $title;
    }

    public function getTitle()
    {
        return $this->title;
    }

    public function setParent(Category $parent = null)
    {
        $this->parent = $parent;
    }

    public function getParent()
    {
        return $this->parent;
    }

    public function setPath($path)
    {
        $this->path = $path;
    }

    public function getPath()
    {
        return $this->path;
    }

    public function getLevel()
    {
        return $this->level;
    }
}

```

### MongoDB example (Annotations)

``` php
<?php

namespace Document;

use Gedmo\Mapping\Annotation as Gedmo;
use Doctrine\ODM\MongoDB\Mapping\Annotations as MONGO;

/**
 * @MONGO\Document(repositoryClass="Gedmo\Tree\Document\MongoDB\Repository\MaterializedPathRepository")
 * @Gedmo\Tree(type="materializedPath", activateLocking=true)
 */
class Category
{
    /**
     * @MONGO\Id
     */
    private $id;

    /**
     * @MONGO\Field(type="string")
     * @Gedmo\TreePathSource
     */
    private $title;

    /**
     * @MONGO\Field(type="string")
     * @Gedmo\TreePath(separator="|")
     */
    private $path;

    /**
     * @Gedmo\TreeParent
     * @MONGO\ReferenceOne(targetDocument="Category")
     */
    private $parent;

    /**
     * @Gedmo\TreeLevel
     * @MONGO\Field(type="int")
     */
    private $level;

    /**
     * @Gedmo\TreeLockTime
     * @MONGO\Field(type="date")
     */
    private $lockTime;

    public function getId()
    {
        return $this->id;
    }

    public function setTitle($title)
    {
        $this->title = $title;
    }

    public function getTitle()
    {
        return $this->title;
    }

    public function setParent(Category $parent = null)
    {
        $this->parent = $parent;
    }

    public function getParent()
    {
        return $this->parent;
    }

    public function getLevel()
    {
        return $this->level;
    }

    public function getPath()
    {
        return $this->path;
    }

    public function getLockTime()
    {
        return $this->lockTime;
    }
}

```

### Path generation

When an entity is inserted, a path is generated using the value of the field configured as the TreePathSource.
If For example:

``` php
$food = new Category();
$food->setTitle('Food');

$em->persist($food);
$em->flush();

// This would print "Food-1" assuming the id is 1.
echo $food->getPath();

$fruits = new Category();
$fruits->setTitle('Fruits');
$fruits->setParent($food);

$em->persist($fruits);
$em->flush();

// This would print "Food-1,Fruits-2" assuming that $food id is 1,
// $fruits id is 2 and separator = "," (the default value)
echo $fruits->getPath();

```

### Locking mechanism for MongoDB

Why you need a locking mechanism for MongoDB? Sadly, MongoDB lacks of full transactional support, so if two or more
users try to modify the same tree concurrently, it could lead to an inconsistent tree. So we've implemented a simple
locking mechanism to avoid this type of problems. It works like this: As soon as a user tries to modify a node of a tree,
it first check if the root node is locked (or if the current lock has expired).

If it is locked, then it throws an exception of type "Gedmo\Exception\TreeLockingException". If it's not locked,
it locks the tree and proceed with the modification. After all the modifications are done, the lock is freed.

If, for some reason, the lock couldn't get freed, there's a lock timeout configured with a default time of 3 seconds.
You can change this value using the **lockingTimeout** parameter under the Tree annotation (or equivalent in XML and YML).
You must pass a value in seconds to this parameter.

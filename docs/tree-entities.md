# 树形实体

TypeORM支持邻接表和闭包表模式存储树形结构。

想了解更多关于层次结构表的知识，请看这个[比尔·卡尔文的精彩演讲](https://www.slideshare.net/billkarwin/models-for-hierarchical-data).

* [邻接表](#邻接表)
* [嵌套集合](#嵌套集合)
* [物化路径（也叫路径枚举）](#物化路径（也叫路径枚举）)
* [闭包表](#闭包表)
* [使用树形实体](#使用树形实体)

### 邻接表

Adjacency list is a simple model with self-referencing. 
The benefit of this approach is simplicity, 
drawback is that you can't load big trees in all at once because of join limitations.
To learn more about the benefits and use of Adjacency Lists look at [this article by Matthew Schinckel](http://schinckel.net/2014/09/13/long-live-adjacency-lists/).
Example:

```typescript
import {Entity, Column, PrimaryGeneratedColumn, ManyToOne, OneToMany} from "typeorm";

@Entity()
export class Category {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column()
    description: string;

    @ManyToOne(type => Category, category => category.children)
    parent: Category;

    @OneToMany(type => Category, category => category.parent)
    children: Category[];
}
     
```

## 嵌套集合

Nested set is another pattern of storing tree structures in the database.
Its very efficient for reads, but bad for writes.
You cannot have multiple roots in nested set.
Example:

```typescript
import {Entity, Tree, Column, PrimaryGeneratedColumn, TreeChildren, TreeParent, TreeLevelColumn} from "typeorm";

@Entity()
@Tree("nested-set")
export class Category {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @TreeChildren()
    children: Category[];

    @TreeParent()
    parent: Category;
}
```

## 物化路径（也叫路径枚举）

Materialized Path (also called Path Enumeration) is another pattern of storing tree structures in the database.
Its simple and effective.
Example:

```typescript
import {Entity, Tree, Column, PrimaryGeneratedColumn, TreeChildren, TreeParent, TreeLevelColumn} from "typeorm";

@Entity()
@Tree("materialized-path")
export class Category {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @TreeChildren()
    children: Category[];

    @TreeParent()
    parent: Category;
}
```

## 闭包表

Closure table stores relations between parent and child in a separate table in a special way. 
It's efficient in both reads and writes.
Example:

```typescript
import {Entity, Tree, Column, PrimaryGeneratedColumn, TreeChildren, TreeParent, TreeLevelColumn} from "typeorm";

@Entity()
@Tree("closure-table")
export class Category {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @TreeChildren()
    children: Category[];

    @TreeParent()
    parent: Category;
}
```

## 使用树形实体

To make bind tree entities to each other its important to set to children entities their parent and save them,
for example:

```typescript
const manager = getManager();

const a1 = new Category("a1");
a1.name = "a1";
await manager.save(a1);

const a11 = new Category();
a11.name = "a11";
a11.parent = a1;
await manager.save(a11);

const a12 = new Category();
a12.name = "a12";
a12.parent = a1;
await manager.save(a12);

const a111 = new Category();
a111.name = "a111";
a111.parent = a11;
await manager.save(a111);

const a112 = new Category();
a112.name = "a112";
a112.parent = a11;
await manager.save(a112);
```

To load such a tree use `TreeRepository`:

```typescript
const manager = getManager();
const trees = await manager.getTreeRepository(Category).findTrees();
```

`trees` will be following:

```json
[{
    "id": 1,
    "name": "a1",
    "children": [{
        "id": 2,
        "name": "a11",
        "children": [{
            "id": 4,
            "name": "a111"
        }, {
            "id": 5,
            "name": "a112"
        }]
    }, {
        "id": 3,
        "name": "a12"
    }]
}]
```

There are other special methods to work with tree entities thought `TreeRepository`:

* `findTrees` - Returns all trees in the database with all their children, children of children, etc.

```typescript
const treeCategories = await repository.findTrees();
// returns root categories with sub categories inside
```

* `findRoots` - Roots are entities that have no ancestors. Finds them all.
Does not load children leafs.

```typescript
const rootCategories = await repository.findRoots();
// returns root categories without sub categories inside
```

* `findDescendants` - Gets all children (descendants) of the given entity. Returns them all in a flat array.

```typescript
const childrens = await repository.findDescendants(parentCategory);
// returns all direct subcategories (without its nested categories) of a parentCategory
```

* `findDescendantsTree` - Gets all children (descendants) of the given entity. Returns them in a tree - nested into each other.

```typescript
const childrensTree = await repository.findDescendantsTree(parentCategory);
// returns all direct subcategories (with its nested categories) of a parentCategory
```

* `createDescendantsQueryBuilder` - Creates a query builder used to get descendants of the entities in a tree.

```typescript
const childrens = await repository
    .createDescendantsQueryBuilder("category", "categoryClosure", parentCategory)
    .andWhere("category.type = 'secondary'")
    .getMany();
```

* `countDescendants` - Gets number of descendants of the entity.

```typescript
const childrenCount = await repository.countDescendants(parentCategory);
```

* `findAncestors` - Gets all parent (ancestors) of the given entity. Returns them all in a flat array.

```typescript
const parents = await repository.findAncestors(childCategory);
// returns all direct childCategory's parent categories (without "parent of parents")
```

* `findAncestorsTree` - Gets all parent (ancestors) of the given entity. Returns them in a tree - nested into each other.

```typescript
const parentsTree = await repository.findAncestorsTree(childCategory);
// returns all direct childCategory's parent categories (with "parent of parents")
```

* `createAncestorsQueryBuilder` - Creates a query builder used to get ancestors of the entities in a tree.

```typescript
const parents = await repository
    .createAncestorsQueryBuilder("category", "categoryClosure", childCategory)
    .andWhere("category.type = 'secondary'")
    .getMany();
```

* `countAncestors` - Gets the number of ancestors of the entity.

```typescript
const parentsCount = await repository.countAncestors(childCategory);
```
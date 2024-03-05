---
description: TypeORM中数据库表之间的数据关联，实现表数据的树形结构
---

# 数据关联与树形嵌套结构

## TypeORM数据关联

文章和标签之间的关联为例。

### 表结构 Entity

通过文章和标签两张表进行关联。

一个标签可以对应多篇文章，一篇文章也可以对应多个标签。

创建两个Entity，使用`Entity`装饰器。

配置好表的列后，通过`ManyToMany`装饰器可以设置两个Entity的数据关联。

例如 `@ManyToMany(() => PostEntity, (post) => post.tags)` ，在TagEntity这一边就关联上了PostEntity。

<details>

<summary>文章Entity</summary>

<pre class="language-typescript"><code class="lang-typescript"><strong>Exclude()
</strong>@Entity('content_posts')
export class PostEntity extends BaseEntity {
    @Expose()
    @PrimaryColumn({ type: 'varchar', generated: 'uuid', length: 36 })
    id: string;

    ...

    @Expose()
    @Type(() => TagEntity)
    @ManyToMany(() => TagEntity, (tag) => tag.posts, {
        cascade: ['insert'],
    })
    @JoinTable()
    tags: Relation&#x3C;TagEntity>[];
}
</code></pre>

</details>

<details>

<summary>标签Entity</summary>

```typescript
@Exclude()
@Entity('content_tags')
@Index(['name'], { unique: true, fulltext: true })
export class TagEntity {
    @Expose()
    @PrimaryColumn({ type: 'varchar', generated: 'uuid', length: 36 })
    id: string;

    ...

    @ManyToMany(() => PostEntity, (post) => post.tags)
    posts: Relation<PostEntity[]>;
}
```

</details>

#### 关于数据关联装饰器的基本理解

**`typeFunction`**

指定当前Entity与另外一个有关联Entity的类。是必须设置的，用于告诉TypeORM如何进行两张表的关联。

**`inverseSide`**

用于指定关系另一端实体对应的属性名称



## 树形数据嵌套结构实现

以Category分类为例来实现树形数据。



### Entity设置

装饰器选择`@Tree`使用物理路径`materialized-path`来设置树形数据。

添加了`parent`和`children`字段，并为他们添加上`@TreeParent`和`@TreeChildren`装饰器，代表父节点和子节点

```typescript
@Tree('materialized-path')
@Entity('content_categories')
export class CategoryEntity extends BaseEntity {
    // ...
    depth = 0;

    @TreeParent({ onDelete: 'NO ACTION' })
    parent: Relation<CategoryEntity> | null;

    @TreeChildren({ cascade: true })
    children: Relation<CategoryEntity>[];
}

```



### 树形存储类TreeRepository

#### `buildBaseQB()`

* **目的**：构建基础查询器。
* **操作**：创建一个查询构建器，为'category'指定别名，并联结选择其'parent'。

#### `findTrees(options?: FindTreeOptions)`

* **目的**：树形结构查询。
* **参数**：`options` - 可选的树查询选项。
* **操作**：首先查找根节点，然后对每个根节点使用`findDescendantsTree`方法递归查找其所有后代，构建整棵树。

#### `findRoots(options?: FindTreeOptions)`

* **目的**：查询顶级分类。
* **参数**：`options` - 可选的树查询选项。
* **操作**：构建基于`buildBaseQB`的查询，筛选出顶级分类（即没有父分类的分类）。

#### `findDescendants(entity: CategoryEntity, options?: FindTreeOptions)`

* **目的**：查询后代分类。
* **参数**：`entity` - 目标实体；`options` - 可选的树查询选项。
* **操作**：使用`createDescendantsQueryBuilder`创建查询构建器，查询指定实体的所有后代。

#### `findAncestors(entity: CategoryEntity, options?: FindTreeOptions)`

* **目的**：查询祖先分类。
* **参数**：`entity` - 目标实体；`options` - 可选的树查询选项。
* **操作**：使用`createAncestorsQueryBuilder`创建查询构建器，查询指定实体的所有祖先。

#### `findDescendantsTree(entity: CategoryEntity, options?: FindTreeOptions)`

* **目的**：查询后代树。
* **参数**：`entity` - 目标实体；`options` - 可选的树查询选项。
* **操作**：使用`createDescendantsQueryBuilder`创建查询构建器，并通过递归构建实体及其后代的树形结构。

#### `findAncestorsTree(entity: CategoryEntity, options?: FindTreeOptions)`

* **目的**：查询祖先树。
* **参数**：`entity` - 目标实体；`options` - 可选的树查询选项。
* **操作**：使用`createAncestorsQueryBuilder`创建查询构建器，并通过递归构建实体及其祖先的树形结构。

#### `countDescendants(entity: CategoryEntity)`

* **目的**：统计后代元素数量。
* **参数**：`entity` - 目标实体。
* **操作**：使用`createDescendantsQueryBuilder`创建查询构建器，计算指定实体的所有后代数量。

#### `countAncestors(entity: CategoryEntity)`

* **目的**：统计祖先元素数量。
* **参数**：`entity` - 目标实体。
* **操作**：使用`createAncestorsQueryBuilder`创建查询构建器，计算指定实体的所有祖先数量。

#### `toFlatTrees(trees: CategoryEntity[], depth = 0, parent: CategoryEntity | null = null)`

* **目的**：打平并展开树。
* **参数**：`trees` - 树数组；`depth` - 当前深度，默认为0；`parent` - 父节点，默认为null。
* **操作**：递归遍历树，为每个节点设置深度和父节点，然后将树结构展平为数组。

<details>

<summary>CategoryRepository</summary>

```typescript
@CustomRepository(CategoryEntity)
export class CategoryRepository extends TreeRepository<CategoryEntity> {
    /**
     * 构建基础查询器
     */
    buildBaseQB() {
        return this.createQueryBuilder('category').leftJoinAndSelect('category.parent', 'parent');
    }

    /**
     * 树形结构查询
     * @param options
     */
    async findTrees(options?: FindTreeOptions) {
        const roots = await this.findRoots(options);
        await Promise.all(roots.map((root) => this.findDescendantsTree(root, options)));
        return roots;
    }

    /**
     * 查询顶级分类
     * @param options
     */
    findRoots(options?: FindTreeOptions) {
        const escapeAlias = (alias: string) => this.manager.connection.driver.escape(alias);
        const escapeColumn = (column: string) => this.manager.connection.driver.escape(column);

        const joinColumn = this.metadata.treeParentRelation!.joinColumns[0];
        const parentPropertyName = joinColumn.givenDatabaseName || joinColumn.databaseName;
        const qb = this.buildBaseQB().orderBy('category.customOrder', 'ASC');
        FindOptionsUtils.applyOptionsToTreeQueryBuilder(qb, options);
        return qb
            .where(`${escapeAlias('category')}.${escapeColumn(parentPropertyName)} IS NULL`)
            .getMany();
    }

    /**
     * 查询后代分类
     * @param entity
     * @param options
     */
    findDescendants(entity: CategoryEntity, options?: FindTreeOptions) {
        const qb = this.createDescendantsQueryBuilder('category', 'treeClosure', entity);
        FindOptionsUtils.applyOptionsToTreeQueryBuilder(qb, options);
        qb.orderBy('category.customOrder', 'ASC');
        return qb.getMany();
    }

    /**
     * 查询祖先分类
     * @param entity
     * @param options
     */
    findAncestors(entity: CategoryEntity, options?: FindTreeOptions) {
        const qb = this.createAncestorsQueryBuilder('category', 'treeClosure', entity);
        FindOptionsUtils.applyOptionsToTreeQueryBuilder(qb, options);
        qb.orderBy('category.customOrder', 'ASC');
        return qb.getMany();
    }

    /**
     * 查询后代树
     * @param entity
     * @param options
     */
    async findDescendantsTree(entity: CategoryEntity, options?: FindTreeOptions) {
        const qb = this.createDescendantsQueryBuilder('category', 'treeClosure', entity)
            .leftJoinAndSelect('category.parent', 'parent')
            .orderBy('category.customOrder', 'ASC');
        FindOptionsUtils.applyOptionsToTreeQueryBuilder(qb, pick(options, ['relations', 'depth']));
        const entities = await qb.getRawAndEntities();
        const relationMaps = TreeRepositoryUtils.createRelationMaps(
            this.manager,
            this.metadata,
            'category',
            entities.raw,
        );
        TreeRepositoryUtils.buildChildrenEntityTree(
            this.metadata,
            entity,
            entities.entities,
            relationMaps,
            {
                depth: -1,
                ...pick(options, ['relations']),
            },
        );

        return entity;
    }

    /**
     * 查询祖先树
     * @param entity
     * @param options
     */
    async findAncestorsTree(entity: CategoryEntity, options?: FindTreeOptions) {
        const qb = this.createAncestorsQueryBuilder('category', 'treeClosure', entity)
            .leftJoinAndSelect('category.parent', 'parent')
            .orderBy('category.customOrder', 'ASC');
        FindOptionsUtils.applyOptionsToTreeQueryBuilder(qb, options);

        const entities = await qb.getRawAndEntities();
        const relationMaps = TreeRepositoryUtils.createRelationMaps(
            this.manager,
            this.metadata,
            'category',
            entities.raw,
        );
        TreeRepositoryUtils.buildParentEntityTree(
            this.metadata,
            entity,
            entities.entities,
            relationMaps,
        );
        return entity;
    }

    /**
     * 统计后代元素数量
     * @param entity
     */
    async countDescendants(entity: CategoryEntity) {
        const qb = this.createDescendantsQueryBuilder('category', 'treeClosure', entity);
        return qb.getCount();
    }

    /**
     * 统计祖先元素数量
     * @param entity
     */
    async countAncestors(entity: CategoryEntity) {
        const qb = this.createAncestorsQueryBuilder('category', 'treeClosure', entity);
        return qb.getCount();
    }

    /**
     * 打平并展开树
     * @param trees
     * @param depth
     * @param parent
     */
    async toFlatTrees(trees: CategoryEntity[], depth = 0, parent: CategoryEntity | null = null) {
        const data: Omit<CategoryEntity, 'children'>[] = [];
        for (const item of trees) {
            item.depth = depth;
            item.parent = parent;
            const { children } = item;
            unset(item, 'children');
            data.push(item);
            data.push(...(await this.toFlatTrees(children, depth + 1, item)));
        }
        return data as CategoryEntity[];
    }
}
```

</details>

查询列表时，通过树的扁平化处理

前端可以根据depth来做一些页面的间隔展示

```
category1
- sub-category1
-- sub-sub-category1
category2
- sub-category2

转换成

category1
sub-category1
sub-sub-category1
category2
sub-category2
```



### CategoryService服务类

#### `findTrees()`

* **目的**：查询分类树。
* **操作**：调用`repository.findTrees`方法，获取整个分类的树形结构。

#### `paginate(options: QueryCategoryDto)`

* **目的**：获取分页数据。
* **参数**：`options` - 分页选项。
* **操作**：首先查询整棵树，然后调用`toFlatTrees`方法将树形结构展平，最后根据分页参数`options`对展平的数据进行分页处理。

#### `detail(id: string)`

* **目的**：获取数据详情。
* **参数**：`id` - 分类的唯一标识符。
* **操作**：使用`findOneOrFail`方法查找并返回具有指定`id`的分类详情，包括其父分类的关联信息。

#### `create(data: CreateCategoryDto)`

* **目的**：新增分类。
* **参数**：`data` - 新增分类的数据。
* **操作**：创建新的分类实例，并设置其父分类。首先检查父分类是否存在，然后保存新分类，并返回新创建的分类详情。

#### `update(data: UpdateCategoryDto)`

* **目的**：更新分类。
* **参数**：`data` - 包含更新信息的对象。
* **操作**：更新分类信息，排除`id`和`parent`属性。检查是否需要更新父分类，如果是，则更新父分类关系，最后返回更新后的分类实例。

#### `delete(id: string)`

* **目的**：删除分类。
* **参数**：`id` - 分类的唯一标识符。
* **操作**：首先检索分类及其关联的父分类和子分类。如果存在子分类，将这些子分类提升一级（即将它们的父分类设置为当前分类的父分类），然后删除当前分类。

#### `getParent(current?: string, parentId?: string)`

* **目的**：获取请求传入的父分类。
* **参数**：`current` - 当前分类的ID；`parentId` - 父分类的ID。
* **操作**：检查`parentId`是否存在，如果存在，验证父分类是否存在于数据库中，如果父分类不存在，则抛出异常。此方法用于在创建或更新分类时处理父分类的逻辑。

<details>

<summary>CategoryService</summary>

```typescript
/**
 * 分类数据操作
 */
@Injectable()
export class CategoryService {
    constructor(protected repository: CategoryRepository) {}

    /**
     * 查询分类树
     */
    async findTrees() {
        return this.repository.findTrees();
    }

    /**
     * 获取分页数据
     * @param options 分页选项
     */
    async paginate(options: QueryCategoryDto) {
        const tree = await this.repository.findTrees();
        const data = await this.repository.toFlatTrees(tree);
        return treePaginate(options, data);
    }

    /**
     * 获取数据详情
     * @param id
     */
    async detail(id: string) {
        return this.repository.findOneOrFail({
            where: { id },
            relations: ['parent'],
        });
    }

    /**
     * 新增分类
     * @param data
     */
    async create(data: CreateCategoryDto) {
        const item = await this.repository.save({
            ...data,
            parent: await this.getParent(undefined, data.parent),
        });
        return this.detail(item.id);
    }

    /**
     * 更新分类
     * @param data
     */
   async update(data: UpdateCategoryDto) {
        await this.repository.update(data.id, omit(data, ['id', 'parent']));
        const item = await this.repository.findOneOrFail({
            where: { id: data.id },
            relations: ['parent'],
        });
        const parent = await this.getParent(item.parent?.id, data.parent);
        const shouldUpdateParent =
            (!isNil(item.parent) && !isNil(parent) && item.parent.id !== parent.id) ||
            (isNil(item.parent) && !isNil(parent)) ||
            (!isNil(item.parent) && isNil(parent));
        // 父分类单独更新
        if (parent !== undefined && shouldUpdateParent) {
            item.parent = parent;
            await this.repository.save(item, { reload: true });
        }
        return item;
    }

    /**
     * 删除分类
     * @param id
     */
    async delete(id: string) {
        const item = await this.repository.findOneOrFail({
            where: { id },
            relations: ['parent', 'children'],
        });
        // 把子分类提升一级
        if (!isNil(item.children) && item.children.length > 0) {
            const nchildren = [...item.children].map((c) => {
                c.parent = item.parent;
                return item;
            });

            await this.repository.save(nchildren, { reload: true });
        }
        return this.repository.remove(item);
    }

    /**
     * 获取请求传入的父分类
     * @param current 当前分类的ID
     * @param id
     */
    protected async getParent(current?: string, parentId?: string) {
        if (current === parentId) return undefined;
        let parent: CategoryEntity | undefined;
        if (parentId !== undefined) {
            if (parentId === null) return null;
            parent = await this.repository.findOne({ where: { id: parentId } });
            if (!parent)
                throw new EntityNotFoundError(
                    CategoryEntity,
                    `Parent category ${parentId} not exists!`,
                );
        }
        return parent;
    }
}
```

</details>

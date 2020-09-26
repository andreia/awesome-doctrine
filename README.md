# Awesome Doctrine

A curated list of useful Doctrine snippets.

Contributions are highly encouraged and very welcome :)

## Table of Contents

- [DQL](#dql)
    - [Defining a Column to be the Key of the Result Hydrated as Array](#defining-a-column-to-be-the-key-of-the-result-hydrated-as-array)
    - [Fetch Only Parts of an Entity](#fetch-only-parts-of-your-entities)
    - [IN Clause in Raw SQL](#in-clause-in-raw-sql)
    - [INDEX BY in QueryBuilder](#index-by-in-querybuilder)
    - [Get Single Row or Null](#get-single-row-or-null)
    - [Ordering with Expressions](#ordering-with-expressions)
    - [Return Only a Value](#return-only-a-value)
    - [Select Directly by Foreign Key Without Join the Foreign Table](#select-directly-by-foreign-key-without-join-the-foreign-table)
    - [WHERE IN Clause](#where-in-clause)
- [Performance](#performance)
    - [Bulk Update Using 'update' Statement Instead of Iterating Through Entities - Object Persisting](#bulk-update-using-update-statement-instead-of-iterating-through-entities-object-persisting)
    - [Temporarily Mark Entities as Read-Only at Runtime](#temporarily-mark-entities-as-read-only-at-runtime)
- [Raw SQL - DBAL](#raw-sql---dbal)
    - [Update, Insert](#update-insert)
    - [Query Data](#query-data)
        - [Select with Parameters](#select-with-parameters)
        - [Select with Named Parameters](#select-with-named-parameters)
    - [Transaction](#transaction)
    - [Truncate Table](#truncate-table)

## DQL

### Defining a Column to be the Key of the Result Hydrated as Array
```php
$em = $this->getEntityManager();

$query = $em->createQuery('SELECT c FROM SomeBundle:Configuration c INDEX BY c.name');
$query->getResult(\Doctrine\ORM\Query::HYDRATE_ARRAY);
```
[[1]](http://docs.doctrine-project.org/en/latest/reference/dql-doctrine-query-language.html#using-index-by)

### Fetch Only Parts of an Entity
```sql
SELECT partial b.{id, title} FROM Book b
```

### IN Clause in Raw SQL
```sql
$stmt = $this->getDoctrine()->getEntityManager()
    ->getConnection()
    ->prepare('SELECT t.id, t.name
        FROM table t
        WHERE t.id IN (:ids)');

$stmt->bindValue('ids', array(1, 2, 3, 4, 5, 6), \Doctrine\DBAL\Connection::PARAM_INT_ARRAY);
$stmt->execute();
```
or
```sql
$stmt = $this->getDoctrine()->getEntityManager()
    ->getConnection()
    ->executeQuery('SELECT t.id, t.name
        FROM table t
        WHERE t.id IN (:ids)',
        array('ids' => array(1, 2, 3, 4, 5, 6)),
        array('ids' => \Doctrine\DBAL\Connection::PARAM_INT_ARRAY)
    )
;
```
[[1]](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/data-retrieval-and-manipulation.html#list-of-parameters-conversion)

### INDEX BY in QueryBuilder
```php
$qb = $em->createQueryBuilder();

$qb->select('u')
   ->from('SomeUserBundle:User', 'u', 'u.id')
   ->add('where', $qb->expr()->like('u.roles', ':role'))
   ->setParameter('role', $role);
```

### Get Single Row or Null
```php
$query->getOneOrNullResult();
```
- no result: return `null`
- more than one result: throw an `NonUniqueResultException` exception

[[1]](http://doctrine-orm.readthedocs.org/en/latest/reference/dql-doctrine-query-language.html#query-result-formats) [[2]](https://github.com/doctrine/doctrine2/blob/master/lib/Doctrine/ORM/AbstractQuery.php#L763)

### Ordering with Expressions
```sql
SELECT m, (m.comments + m.likes_count) AS HIDDEN score FROM Midia m ORDER BY score
```

### Return Only a Value
```sql
$query = $entityManager->createQuery('SELECT COUNT(u.id) FROM User u');
$count = $query->getSingleScalarResult();
```

### Select Directly by Foreign Key Without Join the Foreign Table
```sql
$q = $rep->createQueryBuilder('t')
    ->where('IDENTITY(t.user) = :userId')
    ->orderBy('t.id', 'DESC')
    ->setParameter('userId', $id)
    ->getQuery();
```
or
```sql
SELECT p FROM Product p WHERE IDENTITY(p.shop) = :shopId
```

### WHERE IN Clause
Doctrine 2.4
```php
$categories = ... 
$categoryIds = array();

foreach ($categories as $category) {
    $categoryIds[] = $category->getId();
} 
$queryBuilder = $this
    ->where('model.category IN (:category_ids)')
    ->setParameter('category_ids', $categoryIds);
```
Doctrine 2.5+ supports `ArrayCollection`
```php
$queryBuilder = $this
    ->where('model.category IN (:categories)') 
    ->setParameter('categories', $categories);
```

## Performance

### Bulk Update Using 'update' Statement Instead of Iterating Through Entities - Object Persisting
```php
$em = $this->getDoctrine()->getManager();
$repo = $em->getRepository('AppBundle:User');
$active = true;

$qb = $repo->createQueryBuilder('u');
    $qb->update()
        ->set('u.active', ':userActive')
        ->setParameter('userActive', $active);

$qb->getQuery()->execute();
```

### Temporarily Mark Entities as Read-Only at Runtime
If you have a very large UnitOfWork but know that a large set of entities has not changed, just mark them as read only.
```php
$entityManager->getUnitOfWork()->markReadOnly($entity)
```
## Raw SQL - DBAL

### Update, Insert

```php
$count = $conn->executeUpdate('UPDATE user SET username = ? WHERE id = ?', array('andreia', 1));
echo $count; // 1
```

### Query Data

#### Select with Parameters

```php
$sql = "SELECT * FROM site WHERE id = ?";
$stmt = $conn->prepare($sql);
$stmt->bindValue(1, $id);
$stmt->execute();
$sites = $stmt->fetchAll();
```

#### Select with Named Parameters

```php
$sql = "SELECT * FROM user WHERE name = :name";
$stmt = $conn->prepare($sql);
$stmt->bindValue("name", $name);
$stmt->execute();
$users = $stmt->fetchArray();
```
### Transaction

```php
use Doctrine\DBAL\Connection;

class SomeClass
{
    private $conn;

    // ...

    public function __construct(Connection $conn)
    {
        $this->conn = $conn;
    }

    // ...

    function updateDatabase()
    {
        // ...
        
        try {
            $this->conn->beginTransaction(); 
            $this->conn->setAutoCommit(false);

            $this->conn->executeUpdate('INSERT INTO table1 (field1, field2, field3) VALUES(?, ?, ?)', [$field1, $field2, $field3]);
            $this->conn->executeUpdate('INSERT INTO table2 (field1, field2) VALUES(?, ?)', [$field1, $field2]);

            $this->conn->commit();
        } catch (\Exception $e) {
            // ...

            $this->conn->rollback();
        }
    }

    // ...
}
```

### Truncate Table

```php
$platform = $this->conn->getDatabasePlatform();
$this->conn->executeQuery('SET FOREIGN_KEY_CHECKS = 0;');
$truncateSql = $platform->getTruncateTableSQL('table_name');
$this->conn->executeUpdate($truncateSql);
$this->conn->executeQuery('SET FOREIGN_KEY_CHECKS = 1;');
```

[[1]](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/unitofwork.html#how-doctrine-detects-changes)

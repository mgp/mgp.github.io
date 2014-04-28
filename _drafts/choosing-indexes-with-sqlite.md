---
layout: post
title:  "Choosing indexes with SQLite"
tags: ["sqlite", "indexes", "mobile"]
---

TODO

## The SQLite command line shell

OS X Mavericks should come with SQLite installed by default. On your Linux distribution, install it using your favorite package manager, such as with `apt-get install sqlite3` or `yum install sqlite`. For Windows and other platforms, consult the [SQLite download page](http://www.sqlite.org/download.html).

Once installed, type `sqlite3` to run the command line shell for SQLite:

```text
$ sqlite3
SQLite version 3.7.13 2012-07-17 17:46:21
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> 
```

Say we are building a mobile application where a user can curate a list restaurants. After eating at a restaurant, a user can optionally assign it a rating on a scale from 1 to 5 stars, and a price range on a scale from 1 to 4 dollar signs. Our application will store this data in a SQLite database, and allow the user to query the database by rating and/or by price.

To store this data, we create a table in SQLite named `RESTAURANTS`:

```text
sqlite> create table RESTAURANTS (_id integer primary key, NAME text not null, ADDRESS not null,
   ...>                           RATING integer, PRICE integer);
```

Every restaurant has a name and an address, and so the `NAME` and `ADDRESS` columns are constrained as `not null`. The rating and price assessments are optional, and so the corresponding `RATING` and `PRICE` columns are not similarly constrainted.

The `_id` column is defined as our primary key, which is implicitly constrained as `not null`. If we insert a restaurant into the table without specifying a value for `_id`, SQLite chooses a value that does not already exist in the table. This ensures that each row in the table has a primary key, and each primary key is unique in the table. Thus we can insert 8 restaurants with their corresponding ratings and prices into the table without choosing their primary keys:

```text
sqlite> insert into RESTAURANTS(NAME, ADDRESS, RATING, PRICE) values
   ...>   ('Lavash', '511 Irving St', 4, 2),
   ...>   ('Art''s Cafe', '747 Irving St', 4, 1),
   ...>   ('Naan-N-Curry', '642 Irving St', 3, 1),
   ...>   ('Pasion', '737 Irving St', 4, 3),
   ...>   ('Manna', '845 Irving St', 4, 1),
   ...>   ('San Tung #2', '1033 Irving St', 3, 2),
   ...>   ('Koo', '408 Irving St', 4, 3),
   ...>   ('Kitchen Kura', '1525 Irving St', 4, 2);
```

Note that while the `create table` statement above permits `null` values for `RATING` and `PRICE`, the remainder of this post is more interesting with values assigned.

## B-trees

The following query returns the `_id`, `NAME`, and `ADDRESS` columns for all rows in the `RESTAURANTS` table:

```text
sqlite> select _id, NAME, ADDRESS from RESTAURANTS;
1|Lavash|511 Irving St
2|Art's Cafe|747 Irving St
3|Naan-N-Curry|642 Irving St
4|Pasion|737 Irving St
5|Manna|845 Irving St
6|San Tung #2|1033 Irving St
7|Koo|408 Irving St
8|Kitchen Kura|1525 Irving St
```

The columns of each row are returned in the order in which they appeared in the `create table` statement, and their values are separated by the pipe character `|`. Therefore the first row has an `_id` value of `1`, a `NAME` value of `Lavash`, and an `ADDRESS` value of `511 Irving St`. Its `RATING` and `PRICE` values were not returned.

Note that SQLite ordered the rows by their primary key, which is the first column. (This is equivalent to the order in which they were inserted because SQLite chose monoticially increasing primary keys. The algorithm used is described by the [SQLite documentation](http://sqlite.org/autoinc.html).)

SQLite, like any SQL database, ordered the rows by their primary key because the B-tree for the table orders their corresponding elements by their primary key.

TODO: diagram

To return all rows from the `RESTAURANTS` table, SQLite simply iterates over all the elements in this B-tree. Hence they are ordered by their primary key.

Say the application used the preceding query to display each restaurant as a pin on a map. When the user taps on the pin for a given restaurant, we can use the `_id` associated with that pin to display all its details on a new screen. For example, say the user tapped the pin for *Naan-N-Curry*, which has an `_id` of `3`:

```text
sqlite> select * from RESTAURANTS where _id=3;
3|Naan-N-Curry|642 Irving St|3|1
```

Selecting this single row from the table is very efficient. The B-tree for the `RESTAURANTS` table simply descends to the element with an `_id` of `3`, which is again efficient because the B-tree chose to index its elements by the `_id` column values of their corresponding rows.

## Query plans

To confirm that this is efficient, we can use the SQLite command `explain query plan`.

SQLite, again like any SQL database, generates a [query plan](http://en.wikipedia.org/wiki/Query_plan) for each query. Such a plan specifies what tables and indexes the database engine should use to return the resulting rows, and how it should use them. Such a plan is, hopefully, the most efficient use of those tables and indexes, meaning that is the plan that will return the rows as quickly as possible.

We preface our previous command with `explain query plan`:

```text
sqlite> explain query plan select * from RESTAURANTS where _id=3;
0|0|0|SEARCH TABLE RESTAURANTS USING INTEGER PRIMARY KEY (rowid=?) (~1 rows)
```

Compare this with prefacing the earlier command to return the `_id`, `NAME`, and `ADDRESS` columns for all rows in the `RESTAURANTS` table:

```text
sqlite> explain query plan select _id, NAME, ADDRESS from RESTAURANTS;
0|0|0|SCAN TABLE RESTAURANTS (~1000000 rows)
```

Look closely at the output. The first query plan uses the term `SEARCH TABLE`, meaning SQLite descended a B-tree to the rows to return. The second query plan uses the term `SCAN TABLE`, meaning SQLite had to iterate over element in the B-tree to generate the rows to return. For the second query plan, we must iterate over every element in the B-tree, because the query must return every row in the `RESTAURANTS` table.

Although the primary keys are opaque identifiers, we can TODO.

```text
sqlite> explain query plan select * from RESTAURANTS where _id>=3 and _id<=5;
0|0|0|SEARCH TABLE RESTAURANTS USING INTEGER PRIMARY KEY (rowid>? AND rowid<?) (~62500 rows)
```

While this query has no practical application, we see that SQLite did not require iterating over every element in the B-tree for `RESTAURANTS`. Instead, it descended to the smallest element with an `_id` greater than or equal to `3`, and then iterated only until it reached the largest element with an `_id` less than or equal to `5`. The remaining B-tree elements, namely those with an `_id` less than `3` or greater than `5`, were not iterated over.

As we will explore below, efficient queries come from B-trees containing the resulting rows in contiguous elements. If a query requires iterating across all the elements in a B-tree to find the resulting rows, then it will be inefficient.

## Inefficient queries

Again, our application should allow the user to query the database by rating and/or price. As a first step, say we allow the user to search for restaurants by price only. The following query returns the cheapest restaurants, or those which have a `PRICE` value of `1`:

```text
sqlite> select * from RESTAURANTS where PRICE==1;
2|Art's Cafe|747 Irving St|4|1
3|Naan-N-Curry|642 Irving St|3|1
5|Manna|845 Irving St|4|1
```

Or the application could allow the user to choose the maximum price that he or she is willing to pay. The following query returns all restaurants with a `PRICE` value not exceeding `2`:

```text
sqlite> select * from RESTAURANTS where PRICE<=2;
1|Lavash|511 Irving St|4|2
2|Art's Cafe|747 Irving St|4|1
3|Naan-N-Curry|642 Irving St|3|1
5|Manna|845 Irving St|4|1
6|San Tung #2|1033 Irving St|3|2
8|Kitchen Kura|1525 Irving St|4|2
```

Again, using `explain query plan`, let's look at the plans for both queries:

```text
sqlite> explain query plan select * from RESTAURANTS where PRICE==1;
0|0|0|SCAN TABLE RESTAURANTS (~100000 rows)
sqlite> explain query plan select * from RESTAURANTS where PRICE<=2;
0|0|0|SCAN TABLE RESTAURANTS (~333333 rows)
```

Both plans contain `SCAN TABLE`, meaning for both queries, SQLite had to scan over all elements in the B-tree for the `RESTAURANTS` table in order to return the resulting rows.

Note that in the second query, with the condition `PRICE<=2`, the returned rows are not ordered by price. For example, the first row contains *Lavash* with a `PRICE` value of `2`, the second row contains *Art's Cafe* with a `PRICE` value of `1`, and the fifth row contains *San Tung #2* with another `PRICE` value of `2`. To sort the returned rows by price, we can append the `order by PRICE` clause to our query:

```text
sqlite> select * from RESTAURANTS where PRICE<=2 order by PRICE;
2|Art's Cafe|747 Irving St|4|1
3|Naan-N-Curry|642 Irving St|3|1
5|Manna|845 Irving St|4|1
1|Lavash|511 Irving St|4|2
6|San Tung #2|1033 Irving St|3|2
8|Kitchen Kura|1525 Irving St|4|2
```

The returned rows are now ordered by price, instead of by primary key. Especially interesting is its corresponding query plan:

```text
sqlite> explain query plan select * from RESTAURANTS where PRICE<=2 order by PRICE;
0|0|0|SCAN TABLE RESTAURANTS (~333333 rows)
0|0|0|USE TEMP B-TREE FOR ORDER BY
```

The first line specifies that the query had to scan over all the elements in the B-tree for the `RESTAURANTS` table, just as it had to when we omitted the `order by PRICE` clause. The second line says that SQLite is populating a temporary B-tree, which is distinct from the B-tree for the `RESTAURANTS` table, so that it can fulfill the `order by PRICE` clause. What is happening here?

Instead of SQLite immediately returning an element from the B-tree for `RESTAURANTS` that satisfies the condition `where PRICE<=2`, it copies the element to another B-tree. The key for this B-tree is not just the `_id` column of the underlying row, but the column pair `PRICE, _id`. Such keys are [lexicographically ordered](http://en.wikipedia.org/wiki/Lexicographical_order). The remaining returned columns occupy the value of each element in the new B-tree:

| key (`PRICE`, `_id`) | value (`NAME`, `ADDRESS`, `RATING`)   |
| -------------------- | ------------------------------------- |
| `1`, `2`             | `Art's Cafe`, `747 Irving St`, `4`    |
| `1`, `3`             | `Naan-N-Curry`, `642 Irving St`, `3`  |
| `1`, `5`             | `Manna`, `845 Irving St`, `4`         |
| `2`, `1`             | `Lavash`, `511 Irving St`, `4`        |
| `2`, `6`             | `San Tung #2`, `1033 Irving St`, `3`  |
| `2`, `8`             | `Kitchen Kura`, `1525 Irving St`, `4` |

Choosing `PRICE` as the first column of the key ensures that the returned rows are ordered by price. Appending `_id` to the key ensures that each key in the new B-tree is unique. This also explains why all rows sharing a given `PRICE` value were sorted by `_id`. The first three rows returned had a `PRICE` of `1` and primary keys of `2`, `3`, and `5`:

```text
2|Art's Cafe|747 Irving St|4|1
3|Naan-N-Curry|642 Irving St|3|1
5|Manna|845 Irving St|4|1
```

The last three rows returned had a `PRICE` of `2` and primary keys of `1`, `6`, and `8`:

```text
1|Lavash|511 Irving St|4|2
6|San Tung #2|1033 Irving St|3|2
8|Kitchen Kura|1525 Irving St|4|2
```

Therefore, an `order by` clause like `order by PRICE` is equivalent to `order by PRICE, _id`.

## Creating an index

We've just seen how an `order by` clause creates a temporary B-tree that is ordered by a column other than the primary key. An index in SQLite, and in any SQL database, is little more than a permanent B-tree. With the correct indexes, the database engine can create a plan for a query that uses these indexes. Such efficiently use precludes to scanning over all elements in a B-tree, or create temporary B-trees for `order by` clauses.

We can create an index on the `PRICE` column of the `RESTAURANTS` table like so:

```text
sqlite> create index BY_PRICE on RESTAURANTS (PRICE);
```

This iterates over all the elements in the B-tree of `RESTAURANTS` to create a B-tree for the index `BY_PRICE`. This B-tree has the following elements:

| key (`PRICE`, `_id`) | value  |
| -------------------- | ------ |
| `1`, `2`             | `null` |
| `1`, `3`             | `null` |
| `1`, `5`             | `null` |
| `2`, `1`             | `null` |
| `2`, `6`             | `null` |
| `2`, `8`             | `null` |
| `3`, `4`             | `null` |
| `3`, `7`             | `null` |

Note that the value of each element is `null`. We will explain why in a moment.

Let's regenerate the plan for the query returning those restaurants with a `PRICE` value of `1`:

```text
sqlite> explain query plan select * from RESTAURANTS where PRICE==1;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_PRICE (PRICE=?) (~10 rows)
```

Note that this plan now uses `SEARCH TABLE` instead of `SCAN TABLE`, meaning that it no longer iterate over every element in the B-tree of restaurants. `USING INDEX BY_PRICE` explains why. Instead, SQLite descended to the smallest element with a `PRICE` value equal to `1`, which is the element with the key (`1`, `2`). It then iterated only until it reached the largest element with a `PRICE` value equal to `1`, which is the element with the key (`1`, `5`). The remaining B-tree elements, namely those with a `PRICE` larger than `1`, were not iterated over.

Let's also regenerate the plan for the query returning those restaurants with a `PRICE` value not exceeding `2`:

```text
sqlite> explain query plan select * from RESTAURANTS where PRICE<=2;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_PRICE (PRICE<?) (~250000 rows)
```

Again, we see `USING INDEX BY_PRICE`. Since there is no lower bound on `PRICE`, SQLite descended to the smallest element in the B-tree, which is the element with the key (`1`, `2`). It then iterated only until it reached the largest element with a `PRICE` value not exceeding `2`, which is the element with the key (`2`, `7`). The remaining B-tree elements, namely those with a `PRICE` larger than `2`, were not iterated over.

Note that SQLite uses the B-tree for `BY_PRICE` for both queries. But this B-tree contains only the `PRICE` and `_id` columns for each returned row. The `select *` clause in each query means that SQLite must also return the `RATING` and `ADDRESS` column values. To accomplish this, SQLite extracts the `_id` value from a key in the `BY_PRICE` B-tree, and uses it to look up the `RATING` and `ADDRESS` column values in the B-tree for the `RESTAURANTS` table. As we established earlier, this B-tree indexes its elements by their `_id` column values, and so selecting the row is very efficient.

TODO. This would be a form of [data denormalization](http://en.wikipedia.org/wiki/Denormalization), but it is not done for two reasons:

1. SQLite persists the B-trees of indexes to disk, just like it does for the B-trees of tables. This could therefore significantly increase disk usage.
1. As we will explore below, B-trees increase the cost of inserting, updating, and deleting data. If we update the `PRICE` of a row in the `RESTAURANTS` table, then SQLite must also update its corresponding element in the B-tree for `BY_INDEX`. If SQLite did not do this, then a query using the index could return the wrong rows. If we update the `RATING` of a row in the `RESTAURANTS` table, then SQLite does not need to update any corresponding element in the B-tree for `BY_INDEX`, because that B-tree does not refer to the `RATING` column. This would not be the case if the B-tree for `BY_INDEX` contained the `RATING` for each restaurant. In fact, changing any value for any column of a table would require updating *all indexes* created on that table.

Finally, let's modify the previous query so that it uses the clause `order by PRICE DESC`. It therefore returns prices in descending order instead of ascending order. SQLite generates the following plan for it:

```text
sqlite> explain query plan select * from RESTAURANTS where PRICE<=2 order by PRICE DESC;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_PRICE (PRICE<?) (~250000 rows)
```

Again, SQLite uses the B-tree for `BY_PRICE`. Here SQLite descended to the largest element in the B-tree with a `PRICE` value not exceeding `2`, which is the element with the key (`2`, `7`). Since there is no lower bound on `PRICE`, it then iterated "backward" until it reached the smallest element in the B-tree, which is the element with the key (`1`, `2`). Again, the remaining B-tree elements, namely those with a `PRICE` larger than `2`, were not iterated over.

## Creating a compound index

Now say the application allows the user to choose both a desired rating and price. The following query returns the cheapest restaurants with four stars:

```text
sqlite> select * from RESTAURANTS where RATING=4 and PRICE=1;
2|Art's Cafe|747 Irving St|4|1
5|Manna|845 Irving St|4|1
```

Let's examine the plan for this query:

```text
sqlite> explain query plan select * from RESTAURANTS where RATING=4 and PRICE=1;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_PRICE (PRICE=?) (~2 rows)
```

Note that the plan contians a `PRICE=?` clause, but no corresponding `RATING=?` clause. This means that SQLite could use the `BY_PRICE` index to efficiently find all restaurants with a `PRICE` value of `1`. But among those restaurants, it could not use the index to efficiently find only those with a `RATING` of `4`. This is because the `BY_PRICE` index only ensures that restaurants with the same `PRICE` value are contiguous elements in its B-tree. It does not ensure that restaurants with the same `PRICE` *and* `RATING` value are contiguous elements.

To demonstrate this, recall that the first three elements in this B-tree are:

| key (`PRICE`, `_id`) | value  |
| -------------------- | ------ |
| `1`, `2`             | `null` |
| `1`, `3`             | `null` |
| `1`, `5`             | `null` |

Here, the restaurants with the primary keys of `2` and `5` are *Art's Cafe* and *Manna*, both with a `RATING` of `4`. But the restaurant separating them, with a primary key of `3`, is *Naan-N-Curry* with a `RATING` of `3`.

To address, we can create a *compound index* on both `RATING` and `PRICE`:

```text
sqlite> create index BY_RATING_AND_PRICE on RESTAURANTS (RATING DESC, PRICE ASC);
```

This iterates over all the elements in the B-tree of `RESTAURANTS` to create a B-tree for the index `BY_RATING_AND_PRICE`. This B-tree has the following elements:

| key (`RATING`, `PRICE`, `_id`) | value  
| ------------------------------ | ------ |
| `4`, `1`, `2`                  | `null` |
| `4`, `1`, `5`                  | `null` |
| `4`, `2`, `1`                  | `null` |
| `4`, `2`, `8`                  | `null` |
| `4`, `3`, `4`                  | `null` |
| `4`, `3`, `7`                  | `null` |
| `3`, `1`, `3`                  | `null` |
| `3`, `2`, `6`                  | `null` |

Note that the B-tree orders restaurants by descending `RATING` values, so that higher-rated restaurants are first. For restaurants that have the same `RATING` value, the B-tree orders them by ascending `PRICE` value, so that cheaper restaurants are first. (Whether the most expensive four-star restaurant really ranks higher than the least expensive three-star restaurant isn't really important. This index is interesting when exploring more query plans below.)

If we now reexamine the plan for the query:

```text
sqlite> explain query plan select * from RESTAURANTS where RATING=4 and PRICE=1;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (RATING=? AND PRICE=?) (~9 rows)
```

In the B-tree for `BY_RATING_AND_PRICE`, SQLite descended to the smallest element with a `RATING` value equal to `4` and a `PRICE` equal to `1`, which is the element with the key (`4`, `1`, `2`). It then iterated only until it reached the largest element with a `RATING` equal to `4` and a `PRICE` equal to `1`, which is the element with the key (`4`, `1`, `5`). The remaining B-tree elements, namely those with a `RATING` not equal to `4`, or those with a `RATING` equal to `4` but a `PRICE` larger than `1`, were not iterated over.

Later, say the application designers acknowledge that you might need to pay a little more for a nice restaurant. Therefore, for a given rating, the user can specify the maximum price that he or she is willing to pay. The following query returns all four-star restaurants with a `PRICE` value not exceeding `2`:

```text
sqlite> select * from RESTAURANTS where RATING=4 and PRICE<=2;
2|Art's Cafe|747 Irving St|4|1
5|Manna|845 Irving St|4|1
1|Lavash|511 Irving St|4|2
8|Kitchen Kura|1525 Irving St|4|2
```

Note that these restaurants correspond to the first four entries of the B-tree for `BY_RATING_AND_PRICE`. The plan for this query confirms that SQLite is using this index:

```text
sqlite> explain query plan select * from RESTAURANTS where RATING=4 and PRICE<=2;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (RATING=? AND PRICE<?) (~2 rows)
```

We can return all restaurants ordered by descending `RATING`, and then by ascending `PRICE`:

```text
sqlite> select * from RESTAURANTS order by RATING DESC, PRICE ASC;
2|Art's Cafe|747 Irving St|4|1
5|Manna|845 Irving St|4|1
1|Lavash|511 Irving St|4|2
8|Kitchen Kura|1525 Irving St|4|2
4|Pasion|737 Irving St|4|3
7|Koo|408 Irving St|4|3
3|Naan-N-Curry|642 Irving St|3|1
6|San Tung #2|1033 Irving St|3|2
```

This is the same compound ordering defined by the `BY_RATING_AND_PRICE` index. Instead of SQLite scanning over the B-tree for `RESTAURANTS` and constructing a temporary B-tree that uses this ordering, it simply scans over the B-tree for `BY_RATING_AND_PRICE`:

```text
sqlite> explain query plan select * from RESTAURANTS order by RATING DESC, PRICE ASC;
0|0|0|SCAN TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (~1000000 rows)
```

Finally, let's return all restaurants ordered by ascending `RATING`, and then by descending `PRICE`:

```text
sqlite> select * from RESTAURANTS order by RATING ASC, PRICE DESC;
6|San Tung #2|1033 Irving St|3|2
3|Naan-N-Curry|642 Irving St|3|1
7|Koo|408 Irving St|4|3
4|Pasion|737 Irving St|4|3
8|Kitchen Kura|1525 Irving St|4|2
1|Lavash|511 Irving St|4|2
5|Manna|845 Irving St|4|1
2|Art's Cafe|747 Irving St|4|1
```

Although it's not obvious, `order by RATING ASC, PRICE DESC` is the inverse ordering of `order by RATING DESC, PRICE ASC`. Looking closely, this `select` statement reverses the order of the rows returned by the `select` statement that preceded it. It also uses the B-tree for the `BY_RATING_AND_PRICE` index:

```text
sqlite> explain query plan select * from RESTAURANTS order by RATING ASC, PRICE DESC;
0|0|0|SCAN TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (~1000000 rows)
```

SQLite descended to the largest element in this B-tree, and then iterated "backward" until it reached the smallest element in the B-tree.

#### Re-use of compound indexes

Note that we named the compound index `BY_RATING_AND_PRICE`, with an ordering of `RATING DESC, PRICE ASC`. Choosing to order by `RATING` before `PRICE`, instead of `PRICE` before `RATING`, was deliberate. Recall the B-tree for this index:

| key (`RATING`, `PRICE`, `_id`) | value  |
| ------------------------------ | ------ |
| `4`, `1`, `2`                  | `null` |
| `4`, `1`, `5`                  | `null` |
| `4`, `2`, `1`                  | `null` |
| `4`, `2`, `8`                  | `null` |
| `4`, `3`, `4`                  | `null` |
| `4`, `3`, `7`                  | `null` |
| `3`, `1`, `3`                  | `null` |
| `3`, `2`, `6`                  | `null` |

Previously we demonstrated efficienly querying rows by specifying both `RATING` and `PRICE`. But note that its entries are not just sorted by `RATING` and `PRICE`, but also by `RATING` only. Therefore we can use it to efficiently select rows by `RATING` only.

For example, the following query returns all restaurants with a `RATING` value equal to `4`:

```text
sqlite> select * from RESTAURANTS where RATING==4;
2|Art's Cafe|747 Irving St|4|1
5|Manna|845 Irving St|4|1
1|Lavash|511 Irving St|4|2
8|Kitchen Kura|1525 Irving St|4|2
4|Pasion|737 Irving St|4|3
7|Koo|408 Irving St|4|3
```

And the following query returns all restaurants with a `RATING` value not exceeding `3`:

```text
sqlite> select * from RESTAURANTS where RATING<=3;
3|Naan-N-Curry|642 Irving St|3|1
6|San Tung #2|1033 Irving St|3|2
```

SQLite constructs plans for both queries that use the `BY_RATING_AND_PRICE` index, even though they only constrain the `RATING` column:

```text
sqlite> explain query plan select * from RESTAURANTS where RATING==4;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (RATING=?) (~10 rows)
sqlite> explain query plan select * from RESTAURANTS where RATING<=3;
0|0|0|SEARCH TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (RATING<?) (~250000 rows)
```

Say we had instead named the index `BY_PRICE_AND_RATING`, and ordered by:

```text
sqlite> create index BY_PRICE_AND_RATING on RESTAURANTS (PRICE ASC, RATING DESC);
```

Its B-tree would have the following entries:

| key (`PRICE`, `RATING`, `_id`) | value  |
| ------------------------------ | ------ |
| `1`, `4`, `2`                  | `null` |
| `1`, `4`, `5`                  | `null` |
| `1`, `3`, `3`                  | `null` |


2|Art's Cafe|747 Irving St|4|1
5|Manna|845 Irving St|4|1
3|Naan-N-Curry|642 Irving St|3|1
1|Lavash|511 Irving St|4|2
8|Kitchen Kura|1525 Irving St|4|2
6|San Tung #2|1033 Irving St|3|2
4|Pasion|737 Irving St|4|3
7|Koo|408 Irving St|4|3


### Leftover

Looking at the plan for this query:

```text
sqlite> explain query plan select * from RESTAURANTS order by RATING DESC, PRICE ASC;
0|0|0|SCAN TABLE RESTAURANTS USING INDEX BY_RATING_AND_PRICE (~1000000 rows)
```

To return the resulting rows, SQLite iterated over every element in the B-tree for `BY_RATING_AND_INDEX`. This is becuase its ordering matches that of the query. But this B-tree contains only the `RATING`, `PRICE`, and `_id` columns for each returned row. The `select *` clause in the query means that SQLite must also return the `ADDRESS` column value. Again, SQLite extracts the `_id` value from a key in the `BY_RATING_AND_INDEX` B-tree, and uses it to look up the `ADDRESS` column value in the B-tree for the `RESTAURANTS` table.

If `BY_RATING_AND_INDEX` did not exist, then SQLite would have to create a temporary B-tree with the same key of `RATING DESC, PRICE ASC, _id`. As we established earlier, constructing temporary B-trees is expensive.



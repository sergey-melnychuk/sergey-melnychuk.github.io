---
layout: post
title:  "Recursive SQL queries over tree"
date:   2023-11-17 22:32:42 +0200
categories: sql tree recursive
---

Some 10+ years ago on a job interview I was asked to query the database for the comments tree in a front-end friendly way, meaning no heavy processing/sorting to happen on the back-end. Turns out this question was to check if I knew about the [Hierarchical Queries with Oracle](https://docs.oracle.com/cd/B13789_01/server.101/b10759/queries003.htm) (`WITH RECURSIVE` on other databases), which I did not back then.

So this is how an idiomatic way to do it looks like, with SQLite.

Sample comment tree:

- (id=101) uno: A
  - (id=102) due: B
    - (id=108) tre: H
    - (id=110) uno: I
  - (id=109) tre: J
- (id=103) due: C
  - (id=104): uno: D
- (id=105) tre: E
  - (id=106) uno: F 
    - (id=107) due: G 

Create and open SQLite database:

```bash
touch test.db
sqlite3 test.db
```

Define the table:

```sql
CREATE TABLE comments (
    entry_id SERIAL PRIMARY KEY,
    reply_to BIGINT UNSIGNED,
    issue_id BIGINT UNSIGNED NOT NULL,
    entry_datetime DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    entry_content TEXT NOT NULL,
    entry_author TEXT NOT NULL,

    FOREIGN KEY (reply_to) REFERENCES comments(entry_id)
);
```

Insert comments:

```sql
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (101, NULL, 42, 'uno', 'A');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (102,  101, 42, 'due', 'B');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (103, NULL, 42, 'due', 'C');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (104,  103, 42, 'uno', 'D');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (105, NULL, 42, 'tre', 'E');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (106,  105, 42, 'uno', 'F');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (107,  106, 42, 'due', 'G');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (108,  102, 42, 'due', 'H');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (109,  101, 42, 'tre', 'I');
INSERT INTO comments (entry_id, reply_to, issue_id, entry_author, entry_content) 
    VALUES (110,  102, 42, 'uno', 'J');
```

Query the tree (pre-order traversal):

`WITH RECURSIVE` is followed by a table-defining clause, that describes a starting rows set (`WHERE reply_to is NULL`) and a "connection" between tree layers with (`JOIN comments c ON c.reply_to = ct.entry_id`) and then glues them together using `UNION ALL`.

```sql
WITH RECURSIVE comment_tree (
    issue_id, 
    entry_id, 
    reply_to, 
    entry_author, 
    entry_content, 
    depth
) AS (
        SELECT 
            issue_id, 
            entry_id, 
            reply_to, 
            entry_author, 
            entry_content, 
            0 AS depth
        FROM comments
        WHERE reply_to is NULL
    UNION ALL
        SELECT 
            c.issue_id, 
            c.entry_id, 
            c.reply_to, 
            c.entry_author, 
            c.entry_content, 
            ct.depth+1 AS depth
        FROM comment_tree ct
        JOIN comments c ON c.reply_to = ct.entry_id
)
SELECT entry_id, reply_to, entry_author, entry_content, depth 
FROM comment_tree 
WHERE issue_id = 42
ORDER BY entry_id, reply_to
;
```

Get the results (with `NULL` represented as `---` just to keep formatting straight):

```
101|---|uno|A|0
102|101|due|B|1
103|---|due|C|0
104|103|uno|D|1
105|---|tre|E|0
106|105|uno|F|1
107|106|due|G|2
108|102|due|H|2
109|101|tre|I|1
110|102|uno|J|2
```

More details in the book: [SQL Antipatterns, Volume 1](https://pragprog.com/titles/bksap1/sql-antipatterns-volume-1/).

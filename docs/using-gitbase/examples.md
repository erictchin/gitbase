# Examples

## Get all the repositories where a specific user contributes on HEAD reference

```sql
SELECT refs.repository_id
FROM refs
NATURAL JOIN commits
WHERE commits.commit_author_name = 'Javi Fontan' AND refs.ref_name = 'HEAD';
```

## Get all the HEAD references from all the repositories

```sql
SELECT * FROM refs WHERE ref_name = 'HEAD';
```

## First commit on HEAD history for all repositories

```sql
SELECT
    file_path,
    ref_commits.repository_id
FROM
    commit_files
NATURAL JOIN
    ref_commits
WHERE
    ref_commits.ref_name = 'HEAD'
    AND ref_commits.history_index = 0;
```

## Commits that appear in more than one reference

```sql
SELECT * FROM (
    SELECT COUNT(c.commit_hash) AS num, c.commit_hash
    FROM ref_commits r
    INNER JOIN commits c
        ON r.repository_id = c.repository_id AND r.commit_hash = c.commit_hash
    GROUP BY c.commit_hash
) t WHERE num > 1;
```

##  Get the number of blobs per HEAD commit

```sql
SELECT COUNT(c.commit_hash), c.commit_hash
FROM ref_commits as r
INNER JOIN commits c
    ON r.ref_name = 'HEAD'
    AND r.repository_id = c.repository_id
    AND r.commit_hash = c.commit_hash
INNER JOIN commit_blobs cb
    ON cb.repository_id = c.repository_id AND cb.commit_hash = c.commit_hash
GROUP BY c.commit_hash;
```

## Get commits per committer, per month in 2015

```sql
SELECT COUNT(*) as num_commits, month, repo_id, committer_email
FROM (
    SELECT
        MONTH(committer_when) as month,
        r.repository_id as repo_id,
        committer_email
    FROM ref_commits r
    INNER JOIN commits c
        ON YEAR(c.committer_when) = 2015
        AND r.repository_id = c.repository_id
        AND r.commit_hash = c.commit_hash
    WHERE r.ref_name = 'HEAD'
) as t
GROUP BY committer_email, month, repo_id;
```

## Files from first 6 commits from HEAD references that contains some key and are not in vendor directory

```sql
select
    files.file_path,
    ref_commits.repository_id,
    files.blob_content
FROM
    files
NATURAL JOIN
    commit_files
NATURAL JOIN
    ref_commits
WHERE
    ref_commits.ref_name = 'HEAD'
    AND ref_commits.history_index BETWEEN 0 AND 5
    AND is_binary(blob_content) = false
    AND files.file_path NOT REGEXP '^vendor.*'
    AND (
        blob_content REGEXP '(?i)facebook.*[\'\\"][0-9a-f]{32}[\'\\"]'
        OR blob_content REGEXP '(?i)twitter.*[\'\\"][0-9a-zA-Z]{35,44}[\'\\"]'
        OR blob_content REGEXP '(?i)github.*[\'\\"][0-9a-zA-Z]{35,40}[\'\\"]'
        OR blob_content REGEXP 'AKIA[0-9A-Z]{16}'
        OR blob_content REGEXP '(?i)reddit.*[\'\\"][0-9a-zA-Z]{14}[\'\\"]'
        OR blob_content REGEXP '(?i)heroku.*[0-9A-F]{8}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{12}'
        OR blob_content REGEXP '.*-----BEGIN PRIVATE KEY-----.*'
        OR blob_content REGEXP '.*-----BEGIN RSA PRIVATE KEY-----.*'
        OR blob_content REGEXP '.*-----BEGIN DSA PRIVATE KEY-----.*'
        OR blob_content REGEXP '.*-----BEGIN OPENSSH PRIVATE KEY-----.*'
    );
```

## Create an index for columns on a table

You can create an index either on a specific column or on several columns:

```sql
CREATE INDEX commits_hash_idx ON commits USING pilosa (commit_hash);

CREATE INDEX files_commit_path_blob_idx ON commit_files USING pilosa (commit_hash, file_path, blob_hash);
```

## Create an index for an expression on a table

Note that just one expression at a time is allowed to be indexed.

```sql
CREATE INDEX files_lang_idx ON files USING pilosa (language(file_path, blob_content));
```

## Drop a table's index

```sql
DROP INDEX files_lang_idx ON files;
```

# UAST UDFs Examples

First of all, you should check out the [bblfsh documentation](https://docs.sourced.tech/babelfish) to get yourself familiar with UAST concepts.

Also, you can take a look to all the UDFs and their signatures in the [functions section](/docs/using-gitbase/functions.md)

## Retrieving UASTs with the UDF `uast`

```sql
SELECT file_path, uast(blob_content, language(file_path)) FROM files;
```

This function allows you to directly filter the retrieved UAST by performing a XPATH query on it:

```sql
SELECT file_path, uast(blob_content, language(file_path), "//uast:FunctionGroup") FROM files;
```

This UDF will give you `semantic` UASTs by default. To get some other type see the UDF [`uast_mode`](#retrieving-different-kinds-of-uasts-using-uast_mode).

## Retrieving different kinds of UASTs using `uast_mode`

[bblfsh](https://docs.sourced.tech/babelfish) UAST modes: `semantic`, `annotated`, `native`

```sql
SELECT file_path, uast_mode("semantic", blob_content, language(file_path)) FROM files;

SELECT file_path, uast_mode("annotated", blob_content, language(file_path)) FROM files;

SELECT file_path, uast_mode("native", blob_content, language(file_path)) FROM files;
```

## Filtering UASTs by XPath queries

```sql
SELECT file_path, uast_xpath(uast(blob_content, language(file_path)), "//uast:Identifier") FROM files;

SELECT file_path, uast_xpath(uast_mode("annotated", blob_content, language(file_path)), "//*[@role='Function']") FROM files;
```

## Extracting information from UAST nodes

You can retrieve information from the UAST nodes either through the special selectors `@type`, `@token`, `@role`, `@pos`:

```sql
SELECT file_path, uast_extract(uast(blob_content, language(file_path), "//uast:Block"), "@pos") FROM files;
```

or through a specific property:

```sql
SELECT file_path, uast_extract(uast(blob_content, language(file_path), "//uast:Identifier"), "Name") FROM files;
```

As result, you will get an array showing a list of the retrieved information. Each element in the list matches a node in the given sequence of nodes having a value for that property. It means that the length of the properties list may not be equal to the length of the given sequence of nodes:

```sh
+-------------------------------------------------------------------------------------------------------------------+
| file_path        | uast_extract(uast(files.blob_content, language(files.file_path), "//uast:Identifier"), "Name") |
+-------------------+-----------------------------------------------------------------------------------------------+
| _example/main.go | ["main","driver","NewDefault","sqle","createTestDatabase","AddDatabase","driver","auth"]       |
+-------------------+-----------------------------------------------------------------------------------------------+
```

## Getting the children of a list of nodes

The UDF `uast_children` will return a flattened array of the children nodes from all the nodes in the given array.

```sql
SELECT file_path, uast_children(uast(blob_content, language(file_path), "//uast:Alias")) FROM files;
```

## Monitor the progress of a query

You can monitor the progress of a gitbase query (either a regular query or an index creation query using `SHOW PROCESSLIST`).

Let's say we do the following query over a huge repository:

```sql
SELECT language(file_path, blob_content) FROM files
```

With this query we can monitor its progress:

```sql
SHOW PROCESSLIST
```

We'll get the following output:

```
+------+------+----------------+---------+---------+------+------------+-----------------------------------------------------+
| Id   | User | Host           | db      | Command | Time | State      | Info                                                |
+------+------+----------------+---------+---------+------+------------+-----------------------------------------------------+
|    2 | root | 127.0.0.1:3306 | gitbase | query   |   36 | files(1/3) | select language(file_path, blob_content) from files |
|   12 | root | 127.0.0.1:3306 | gitbase | query   |    0 | running    | show processlist                                    |
+------+------+----------------+---------+---------+------+------------+-----------------------------------------------------+
2 rows in set (0,00 sec)
```

From this output, we can obtain some information about our query:
- It's been running for 36 seconds.
- It's only querying files table and has processed 1 out of 3 partitions.

To kill a query that's currently running you can use the value in `Id`. If we were to kill the previous query, we would need to use the following query:

```sql
KILL QUERY 2
```
# EFCore.Merge

> It's purely a design document!

This library adds support for "Merge" operations to EF Core of Postgres.

Uses the `Merge` sql command.

Requirements:
- Postgres >= 15

## Synopsys

```
[ WITH _`with_query`_ [, ...] ]
MERGE INTO [ ONLY ] _`target_table_name`_ [ * ] [ [ AS ] _`target_alias`_ ]
USING _`data_source`_ ON _`join_condition`_
_`when_clause`_ [...]
[ RETURNING { * | _`output_expression`_ [ [ AS ] _`output_name`_ ] } [, ...] ]

where _`data_source`_ is:

{ [ ONLY ] _`source_table_name`_ [ * ] | ( _`source_query`_ ) } [ [ AS ] _`source_alias`_ ]

and _`when_clause`_ is:

{ WHEN MATCHED [ AND _`condition`_ ] THEN { _`merge_update`_ | _`merge_delete`_ | DO NOTHING } |
  WHEN NOT MATCHED BY SOURCE [ AND _`condition`_ ] THEN { _`merge_update`_ | _`merge_delete`_ | DO NOTHING } |
  WHEN NOT MATCHED [ BY TARGET ] [ AND _`condition`_ ] THEN { _`merge_insert`_ | DO NOTHING } }

and _`merge_insert`_ is:

INSERT [( _`column_name`_ [, ...] )]
[ OVERRIDING { SYSTEM | USER } VALUE ]
{ VALUES ( { _`expression`_ | DEFAULT } [, ...] ) | DEFAULT VALUES }

and _`merge_update`_ is:

UPDATE SET { _`column_name`_ = { _`expression`_ | DEFAULT } |
             ( _`column_name`_ [, ...] ) = [ ROW ] ( { _`expression`_ | DEFAULT } [, ...] ) |
             ( _`column_name`_ [, ...] ) = ( _`sub-SELECT`_ )
           } [, ...]

and _`merge_delete`_ is:

DELETE
```

## Use cases
### 1. When matched

#### 1.1
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN MATCHED [AND <condition>] THEN
    <update_statement>
```
#### 1.2
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN MATCHED [AND <condition>] THEN
    <delete_statement>
```

### 2.  When not matched (by target)

#### 2.1
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN NOT MATCHED [AND <condition>] THEN
    <insert_statement>
```
### 3. When not matched by source

#### 3.1
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN NOT MATCHED BY SOURCE [AND <condition>] BY SOURCE THEN
    <update_statement>
```

#### 3.2
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN NOT MATCHED BY SOURCE [AND <condition>] BY SOURCE THEN
    <delete_statement>
```
### 4. Combination

#### 4.1 Upsert
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN MATCHED [AND <condition>] THEN
    <update_statement>
WHEN NOT MATCHED [AND <condition>] THEN
    <insert_statement>
```

#### 4.2 Synchronize
```sql
MERGE INTO "Target" AS n
USING "Source" AS n
ON <condition>
WHEN MATCHED [AND <condition>] THEN
    <update_statement>
WHEN NOT MATCHED [AND <condition>] THEN
    <insert_statement>
WHEN NOT MATCHED [AND <condition>] BY SOURCE THEN
    <delete_statement>
```

## Api
Only 4.1 and 4.2 are interesting.

Variant 1. One method:
```csharp
public async Task MergeAsync(
	this DbSet<TEntity> target,
	TEntity[] source,
	Func? on,
	Func? update,
	bool shouldDelete = false
	)
	{}
```
Example

syncronize
```csharp
await Users.MergeAsync(
	data,
	on: (user) => user.ExternalId,
	update: (old, new) => {
			old.Name = new.Name;
			old.Surname = new.Surname;
		}
	shouldDelete: true
	);
```

upsert
```csharp
await Users.MergeAsync(
	data,
	on: (user) => user.ExternalId,
	update: (old, new) => {
			old.Name = new.Name;
			old.Surname = new.Surname;
		}
	);
```
Match by primary key. Update all values
```csharp
await Users.MergeAsync(data);
```

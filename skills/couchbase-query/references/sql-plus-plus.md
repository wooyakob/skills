# SQL++ (N1QL) Language Reference

## Data Types

| SQL++ Type | JSON Equivalent | Notes |
|-----------|----------------|-------|
| NULL | null | Explicit null value |
| MISSING | absent key | Field doesn't exist in document |
| BOOLEAN | true/false | |
| NUMBER | number | Integers and floats |
| STRING | "string" | UTF-8 |
| ARRAY | [] | Ordered |
| OBJECT | {} | Unordered key-value pairs |

`IS NULL` tests for null. `IS MISSING` tests for absent field. `IS NOT VALUED` = IS NULL OR IS MISSING.

## Keyspace Paths

```sql
-- Fully qualified
`bucket-name`.`scope-name`.`collection-name`

-- Short form (when bucket/scope are implicit, e.g., scope.query())
`collection-name`

-- With alias
`bucket`.`scope`.`users` AS u
```

## SELECT Clauses

```sql
-- All fields
SELECT * FROM `bucket`.`scope`.`users` WHERE type = "user";

-- Specific fields (returned as top-level in result row)
SELECT name, email, createdAt FROM `bucket`.`scope`.`users`;

-- Nested field access
SELECT address.city, address.zip FROM `bucket`.`scope`.`users`;

-- With document key
SELECT META().id AS id, name FROM `bucket`.`scope`.`users`;

-- Rename fields
SELECT name AS fullName, email AS emailAddress FROM `bucket`.`scope`.`users`;

-- Computed fields
SELECT name, price * 0.9 AS discountedPrice FROM `bucket`.`scope`.`products`;

-- Construct new object
SELECT {"user": name, "contact": email} AS info FROM `bucket`.`scope`.`users`;

-- DISTINCT
SELECT DISTINCT category FROM `bucket`.`scope`.`products`;

-- RAW — returns scalar values instead of objects
SELECT RAW name FROM `bucket`.`scope`.`users` WHERE type = "user";
-- Returns: ["Alice", "Bob", "Carol"]
```

## WHERE Clause

```sql
WHERE type = "user"
  AND status IN ["active", "trial"]
  AND score >= 100
  AND name LIKE "Alice%"        -- % = any sequence, _ = one char
  AND email IS NOT MISSING      -- field exists
  AND premium IS NOT NULL       -- field is not null
  AND NOT archived
  AND (role = "admin" OR role = "moderator")
```

## String Functions

```sql
UPPER(name), LOWER(email), LENGTH(description)
SUBSTR(name, 0, 5)              -- 0-indexed
CONCAT(firstName, " ", lastName)
TRIM(name), LTRIM(name), RTRIM(name)
REPLACE(text, "old", "new")
SPLIT(csv, ",")                 -- returns array
CONTAINS(description, "sale")  -- boolean
POSITION(text, "word")         -- index of first occurrence (-1 if not found)
REGEXP_MATCHES(name, "^[A-Z]") -- boolean
REGEXP_REPLACE(text, "\\d+", "NUM")
```

## Date/Time Functions

```sql
NOW_STR()                          -- current UTC timestamp: "2024-03-15T14:32:01.123Z"
NOW_MILLIS()                       -- milliseconds since epoch
DATE_FORMAT_STR(str, "2006-01-02") -- reformat date string (Go reference time)
DATE_PART_STR("2024-03-15", "month")   -- returns 3
DATE_DIFF_STR("2024-03-15", "2024-01-01", "day")  -- returns 74
DATE_ADD_STR(NOW_STR(), -7, "day")     -- 7 days ago
STR_TO_MILLIS("2024-03-15T00:00:00Z") -- convert to ms
MILLIS_TO_STR(1710460800000)           -- convert to string
```

## Array Functions

```sql
ARRAY_LENGTH(tags)                   -- count elements
ARRAY_CONTAINS(tags, "sale")         -- boolean membership test
ARRAY_APPEND(tags, "new-tag")        -- returns new array with element added
ARRAY_PREPEND("first", tags)         -- prepend (note: value, array order)
ARRAY_REMOVE(tags, "old-tag")        -- remove all occurrences
ARRAY_DISTINCT(tags)                 -- deduplicate
ARRAY_CONCAT(tags1, tags2)           -- merge arrays
ARRAY_FLATTEN(nested, 1)             -- flatten one level
ARRAY_SORT(nums)                     -- sort ascending
ARRAY_REVERSE(arr)                   -- reverse order
ARRAY_RANGE(0, 5)                    -- [0, 1, 2, 3, 4]

-- ARRAY constructor
SELECT ARRAY x * 2 FOR x IN nums END AS doubled
FROM `bucket`.`scope`.`data`;

-- FIRST (first element satisfying condition)
SELECT FIRST x FOR x IN tags WHEN x LIKE "elec%" END AS mainCategory
FROM `bucket`.`scope`.`products`;
```

## UNNEST (Expand Arrays into Rows)

```sql
-- Each array element becomes a separate result row
SELECT p.name, t AS tag
FROM `bucket`.`scope`.`products` AS p
UNNEST p.tags AS t
WHERE t LIKE "audio%";

-- LEFT UNNEST — keep documents with empty arrays
SELECT p.name, t
FROM `bucket`.`scope`.`products` AS p
LEFT UNNEST p.tags AS t;
```

## Subqueries

```sql
-- Subquery in WHERE
SELECT * FROM `bucket`.`scope`.`orders` WHERE userId IN (
    SELECT RAW META().id FROM `bucket`.`scope`.`users` WHERE role = "vip"
);

-- Correlated subquery
SELECT u.name,
    (SELECT COUNT(*) FROM `bucket`.`scope`.`orders` AS o WHERE o.userId = META(u).id)[0] AS orderCount
FROM `bucket`.`scope`.`users` AS u;

-- Subquery as FROM source
SELECT avg_price.category, avg_price.avg
FROM (
    SELECT category, AVG(price) AS avg FROM `bucket`.`scope`.`products` GROUP BY category
) AS avg_price
WHERE avg_price.avg > 50;
```

## Window Functions

```sql
SELECT
    name,
    price,
    category,
    RANK() OVER (PARTITION BY category ORDER BY price DESC) AS priceRank,
    ROW_NUMBER() OVER (ORDER BY createdAt DESC) AS rowNum,
    SUM(price) OVER (PARTITION BY category) AS categoryTotal
FROM `bucket`.`scope`.`products`;
```

## META() Functions

```sql
META().id          -- document key
META().cas         -- CAS value
META().expiration  -- TTL expiry timestamp (0 = no expiry)
META().type        -- document type (internal)

-- Access meta of a specific keyspace alias
SELECT META(p).id, p.name FROM `bucket`.`scope`.`products` AS p;

-- Lookup by key directly (USE KEYS)
SELECT * FROM `bucket`.`scope`.`users` USE KEYS ["user::u1", "user::u2"];
```

## OBJECT Functions

```sql
OBJECT_KEYS(doc)              -- array of field names
OBJECT_VALUES(doc)            -- array of values
OBJECT_PAIRS(doc)             -- array of {"name": k, "val": v}
OBJECT_LENGTH(doc)            -- count of fields
OBJECT_REMOVE(doc, "field")   -- return doc without field
OBJECT_UNWRAP({"key": val})   -- extract single value from single-key object
OBJECT_ADD(doc, "key", val)   -- return doc with new field (no-op if exists)
OBJECT_PUT(doc, "key", val)   -- return doc with field set (overwrites)

-- OBJECT constructor
SELECT OBJECT v.name: v.score FOR v IN members END AS scoreMap
FROM `bucket`.`scope`.`teams`;
```

## Type Functions

```sql
TYPE(value)            -- returns "null", "boolean", "number", "string", "array", "object", "missing"
IS_STRING(value)       -- boolean
IS_NUMBER(value)
IS_ARRAY(value)
IS_OBJECT(value)
TONUMBER("42.5")       -- convert string to number
TOSTRING(42)           -- convert to string
TOBOOLEAN(1)           -- convert to boolean
```

## Prepared Statements

```python
from couchbase.options import QueryOptions

# Named prepared statement — SDK caches automatically on first execution
result = cluster.query(
    "SELECT name, email FROM `bucket`.`scope`.`users` WHERE type = $type AND status = $status",
    QueryOptions(
        named_parameters={"type": "user", "status": "active"},
        adhoc=False  # False = use prepared statement (cached after first run)
    )
)
```

The SDK automatically prepares on first run and reuses on subsequent calls when `adhoc=False`.

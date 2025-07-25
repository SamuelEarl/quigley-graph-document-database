# Schema Definition Rules

Explicit is better than implicit. The schema file will be the single-source of truth that provides a clear and declarative reference for the data structure of the database. Everything will be defined in the schema file (e.g. data types, required and optional properties, indexes, constraints). The CLI commands will reference the schema file and use the definitions that are in the schema file.

## ID VALUES

* FalkorDB generates an internal ID, which is managed by FalkorDB. It does not appear that you can query nodes by the internal ID. However, you can retrieve the internal ID, if necessary, by using the ID function like so:

```
MATCH (n) RETURN ID(n) LIMIT 3
```

* Unique IDs must be generated for each node (and relation?) by default. Most (or all?) GQL/Cypher implementations do not generate unique IDs for you. FalkorDB, for example, generates an auto-incrementing internal ID, but you cannot query for nodes by that internal ID. So you will need to explicitly define an ID property for each node (and relation?) in your schema.
* Since FalkorDB has an internal ID property, it is recommended to name your unique ID something else (e.g. `uuid`, `ulid`). That way when you query your data, those ID properties will be distinct from each other. Define your unique ID property with an `autoGenerate` function so that a unique ID will be auto-generated for you with each CREATE query. For example:

```
node User {
  props: {
    uuid: {
      type: String,
      autoGenerate: () => uuidv7(),
    },
    # Other props...
  },
};
```

* You can use any unique identifeir for your ID properties. If you are not sure what to use, then you can look into UUIDv4, UUIDv7, and ULID as possible options for generating ID values. (See [UUIDv4 vs UUIDv7 vs ULID: Choosing the Right Identifier for Database Performance](https://medium.com/@ciro-gomes-dev/uuidv4-vs-uuidv7-vs-ulid-choosing-the-right-identifier-for-database-performance-1f7d1a0fe0ba))
* Quigley does not make any assumptions about the unique ID value that you want to use (e.g. Int vs String), so you need to explicitly define the type of ID that you want in your schema.

## INDEXES

Indexes can be defined on the specific properties that they apply to. See the `schema.v1.gqls` file for examples. 

They can also be defined on the node/relation if multiple properties in included. (See Multicolumn Indexes: https://www.postgresql.org/docs/current/indexes-multicolumn.html)

(Read more about indexes: https://www.postgresql.org/docs/current/indexes.html.)

## CONSTRAINTS

Constraints can be defined on the specific properties that they apply to or they can be defined on the node/relation if multiple properties are included (e.g. a "unique" constraint across multiple properties). See the `schema.v1.gqls` file for examples.

_See https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-UNIQUE-CONSTRAINTS_

### REQUIRED & OPTIONAL VALUES

* Required Fields: A field is required if it is defined in the schema (with no `optional` or `default` definitions). In a query, if no value is provided for a required field, then an error will be thrown. Required fields cannot have default values defined in the schema.
    * Tip: In most database designs the majority of properties should be required. (https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-NOT-NULL)
* Optional Fields: A field is optional if it is marked with `optional: <defaultValue>`. An optional field must have a default value defined in the schema. In a query, if a value is not provided in a query, then the default value will be inserted. NULL is not an acceptable default value. The default value must be the same data type as the field.
* This means that all properties in all nodes/relations will be present and will have a non-null value. This is a stricter enforcement of the values that are inserted into the database, which is what I prefer for data integrity.

---

MORE NOTES:

* I will allow for required and optional values and I will reconstruct the query string and add any default values, as opposed to trying to inject default values into the existing query string.
* Required and default values seem to be beneficial features if they are implemented at the database level because you can enforce those constraints for users who access the database outside of the web app. Since I am creating Quigley for a web app, I can enforce required and optional values through application code and I think the following notes are the best approach for a web app. I don't know if it makes sense to define "required" and "optional" fields in the schema because I can simply assume that any field that is defined in the schema should be treated as a required field in CREATE and MERGE clauses and developers will have to provide default values in the code if a user does not provide a default value through the web app. See the following notes for more details.
* If a property is defined in the schema, then it is considered to be a required property in CREATE and MERGE clauses. 
  * If I allow properties to be optional, then I would have to insert default values depending on the data type of the property (e.g. "" for string properties, 0 for number properties). I would have to insert the default value in both the query string and in the params object, but that seems too complex and maybe unnecessary. Maybe I should just require users to specify all values in the query string and the params object instead of allowing users to define default values in the schema. Since Cypher does not have a schema there is no way to define default values in Cypher, but I might want to change that with Quigley.
  * In version 1 I will require users to explicitly define the values and/or default values that should be inserted into the graph/database in each query that they define. Users would have to define their own default values in their code before they define a query. For example: `const displayName = payload.displayName ? payload.displayName : "";`
* TODO: Maybe this will be a version 2 feature, if I decide that I want to do this: If a user does not pass a required property (and param) in a CREATE or MERGE clause, then Quigly will throw an error, unless the user provides a default value in the schema. If the user does not pass a required property (and param) but does provide a default value in the schema, then the default value will be inserted into the database.
* If a user introduces a new property as part of a schema migration, then the user will need to define a default value that will be inserted into nodes or relations that existed before the migration and that did not have that property previously. That way the schema will be consistent across the entire graph/database.
* As part of the validation process for values that are passed to a query, I want to check for ints and floats instead of just numbers. So I need to figure out how to check for ints and floats in my query validation.

### OTHER CONSTRAINTS

* See which constraints from Postgres make sense for graph databases: https://www.postgresql.org/docs/current/ddl-constraints.html. (e.g. Primary Keys and Foreign Keys are not used in graph databases, so they would not make sense.)

## RELATIONS

* relations are enforced through constraints that are defined on the database (i.e. they are not enforced through this package). So the `from` and `to` properties in a `relation` object are not validated against the schema. However, the `from` and `to` properties are for your reference and can be used to help define the structure of your schema.

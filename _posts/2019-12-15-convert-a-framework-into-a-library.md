---
layout: post
title: "Convert a framework into a library"
subtitle: "SQL type provider"
---

Recently I was working on a project where I thought the the F# SQL provider could be a good solution. Unfortunately I found that there were too many limitations. Despite many attempts at hacking the library, the code was just too tightly coupled to make it work. I came across Thomas Petricek's [article][1] describing the difference between libraries and frameworks.

- **Frameworks**. When using a framework, the framework is in charge of running the system. It defines some extensibility points (interfaces) where you need to put your implementation.

- **Libraries**. When using a library, you are in charge of running the system. The library defines some points through which you can access it (functions and types) and your code can call it as it needs.

Tomas also advises working towards libraries rather than frameworks, claiming that:

- Frameworks do not compose
- Frameworks are hard to explore
- Frameworks shape how you code

Despite Tomas being the author of the SQL provider, it feels more like a framework than a library. I wanted to experiment to see if I could tease out some kind of underlying functionality and make it more user-friendly.

### Refactoring
Interfaces are great place to start when refactoring because they give so much meaning without the noise of code, so let's start there.

```fsharp
    // ISqlDataContext
    val CallSproc : RunTimeSprocDefinition * QueryParameter [] * obj [] -> obj
    val CallSprocAsync : RunTimeSprocDefinition * QueryParameter [] * obj [] -> Async<SqlEntity>
    val ClearPendingChanges : unit -> unit
    val CreateConnection : unit -> IDbConnection
    val CreateEntities : string -> IQueryable<SqlEntity>
    val CreateEntity : string -> SqlEntity
    val CreateRelated : SqlEntity * string * string * string * string * string * RelationshipDirection -> IQueryable<SqlEntity>
    val GetIndividual : string * obj -> SqlEntity
    val GetPendingEntities : unit -> SqlEntity list
    val GetPrimaryKeyDefinition : string -> string
    val ReadEntities : string * ColumnLookup * IDataReader -> SqlEntity []
    val ReadEntitiesAsync : string * ColumnLookup * DbDataReader -> Async<SqlEntity []>
    val SaveContextSchema : string -> unit
    val SubmitChangedEntity : SqlEntity -> unit
    val SubmitPendingChanges : unit -> unit
    val SubmitPendingChangesAsync : unit -> Async<unit>
    val CommandTimeout : Option<int>
    val ConnectionString : string
    val SqlOperationsInSelect : SelectOperations

    // ISqlProvider
    val CreateConnection : string -> IDbConnection 
    val CreateCommand : IDbConnection * string -> IDbCommand 
    val CreateCommandParameter : QueryParameter * obj -> IDbDataParameter 
    val GetTables : IDbConnection * CaseSensitivityChange -> Table list 
    val GetTableDescription : IDbConnection * string -> string 
    val GetColumns : IDbConnection * Table -> ColumnLookup 
    val GetColumnDescription : IDbConnection * string * string -> string 
    val GetPrimaryKey : Table -> string option 
    val GetRelationships : IDbConnection * Table -> Relationship list * Relationship list 
    val GetSprocs : IDbConnection -> Sproc list 
    val GetIndividualsQueryText : Table * int -> string 
    val GetIndividualQueryText : Table * string -> string 
    val GetSchemaCache : unit -> SchemaCache 
    val ProcessUpdates : IDbConnection * ConcurrentDictionary<SqlEntity,DateTime> * TransactionOptions * Option<int> -> unit 
    val ProcessUpdatesAsync : DbConnection * ConcurrentDictionary<SqlEntity,DateTime> * TransactionOptions * Option<int> -> Async<unit> 
    val GenerateQueryText : SqlQuery * string * Table * Dictionary<string,ResizeArray<ProjectionParameter>> * bool * IDbConnection -> string * ResizeArray<IDbDataParameter> 
    val ExecuteSprocCommand : IDbCommand * QueryParameter [] * QueryParameter [] * obj [] -> ReturnValueType 
    val ExecuteSprocCommandAsync : DbCommand * QueryParameter [] * QueryParameter [] * obj [] -> Async<ReturnValueType> 
    val GetLockObject : unit -> obj 
```

It seems there is a nice separation of concerns; context deals with abstract concepts and provider deals with the actual implementation, one for each SQL technology MySQL etc. So what can I do next?

### Finding structure
A typical SQL database looks like this:
````fsharp
type DB = DbName * Schema list
type Schema = SchemaName * Table list
type Table = TableName * Row list
type Row = Column list
type Column = ColumnName * PrimaryKey * ForeignKey * IsUnique * MaxLength (* etc... *)
type Cell = Value
````

Looking at the signatures I don't really see much underlying structure yet. The first point of action is to abstract away domain-specific terms. Terms such as table, column, primary key etc. are too specific to SQL and hinder my ability to abstract. So let's start breaking them down.

### Flattening the structure
A `Cell` is associated with a `Table`, `Row` and `Column` so I can create a new type `Result` *(in expanded form)*

```fsharp
type Result = (TableName * RowIndex * ColumnName) * (Value * PrimaryKey * ForeignKey * IsUnique * MaxLength (* etc... *))
```

- `Column` is a tuple which is limiting, can I improve on this? **Yes**. I can make it into a dictionary. `primaryKey:true; foreignKey:false; unique:true; maxLength:4000; type:string]`.
- I do the same technique for `TableName`, `ColumnName` etc. by making a `Name list`. Both `Name` and `RowIndex` are indexes which collectively we call an `Identifier`.
- Next, if I redefine the `value` as `[key:value]`, I can just include it with the dictionary. Also both `Key` and `Query` are just other ways of saying `Identifier`.

Simply put.

```fsharp
type Result = (Identifier * Map<Identifier,Value>)
```

Let's take a step back and think what am I doing on a very high level? I'm sending a query to a database and receiving a set of results. *(At this stage I'm not specifying what the actual implementation is.)* 

```fsharp
type Provider = Query -> Result list
```

In other words

```fsharp
type Query = Identifier -> Value list
type Provider = Query -> Query list
val run : Query -> Value list
```

We've abstracted away SQL specific concepts and it actually makes perfect sense. Think about a graph database for a moment. Each node is connected to other nodes and/or a dictionary of key/values. Each time we drill down, we are essentially querying and returning back a set of nodes, which are also queryable. Exactly what we have here.

### Converting back into SQL paradigm
To prove this works, let's look back at SQL. Assuming that my identifier is some valid DSL.

- **Properties of Northwind database?** 
  - `[() -> db:Northwind -> .*]`
- **Size of Northwind database?** 
  - `[() -> db:Northwind -> .size]`
- **Northwind table names?** 
  - `[() -> db:Northwind -> table:* -> .name]`
- **Schema of product table?** 
  - `[() -> db:Northwind -> table:product -> column:* -> [.name .type .maxlength]]`
- **Schema of Northwind database?** 
  - `[() -> db:Northwind -> table:* -> column:* -> [table.name .name .type .maxlength]]`
- **Schema of order-history stored procedure?** 
  - `[() -> db:Northwind -> procedure:order-history -> [.name .args]]`

## In conclusion
By analyzing the type signatures of our original provider and simplifying, I now have a generic QueryProvider that has a very simple recursive structure. Domain specific terms such as tables and columns have been completely abstracted away. By creating a universal querying language as above it means I can delegate the task of generating/parsing queries per technology including but not limited to SQL, thus creating a powerful, reusable library.

```fsharp
val expressionToQuery   : Expr<'a> -> Identifier
val queryToSql          : Identifier -> string
val queryToNeo4j        : Identifier -> string
```

#### Links
- [Library patterns: Why frameworks are evil][1]
- [Type Driven Development][2]
- [SQLProvider][3]

[1]: http://tomasp.net/blog/2015/library-frameworks/
[2]: https://blog.ploeh.dk/2015/08/10/type-driven-development/
[3]: https://fsprojects.github.io/SQLProvider/
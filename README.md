### Tables.jl

The Tables.jl package provides simple, yet powerful interface functions for working with all kinds tabular data through predictable access patterns.

```julia
    Tables.rows(table) => Rows
    Tables.columns(table) => Columns
```
Where `Rows` and `Columns` are the duals of each other:
* `Rows` is an iterator of property-accessible objects (any type that supports `propertynames(row)` and `getproperty(row, nm::Symbol`)
* `Columns` is a property-accessible object of iterators (i.e. each column is an iterator)

In addition to these `Rows` and `Columns` objects, it's useful to be able to query properties of these objects:
* `Tables.schema(rows_or_columns)`: returns a `Tables.Schema` object
  * column names can be accessed as a tuple of Symbols like `sch.names`
  * column types can be accessed as a tuple of types like `sch.types`, though note that `sch.types` may return `nothing`, indicating column types are not computable or otherwise unknown

A big part of the power in these simple interface functions is that each (`Tables.rows` & `Tables.columns`) is defined for any table type, even if the table type only explicitly implements one interface function or the other.
This is accomplished by providing performant, generic fallback definitions in Tables.jl itself (though obviously nothing prevents a table type from implementing each interface function directly).

With these simple definitions, powerful workflows are enabled:
* A package providing data cleansing, manipulation, visualization, or analysis can automatically handle any number of decoupled input table types
* A tabular file format can have automatic integration with in-memory structures and translation to other file formats

So how does one go about satisfying the Tables.jl interface functions? It mainly depends on what you've alrady defined and the natural access patterns of your table:

To support `Rows`:
* Define `Tables.rowaccess(::Type{MyTable}) = true`: this signals to other types that `MyTable` supports valid `Row`-iteration
* Define `Tables.rows(x::MyTable)`: return a `Row`-iterator object (perhaps the table itself if already defined)

To support `Columns`:
* Define `Tables.columnaccess(::Type{MyTable}) = true`: this signals to other types that `MyTable` supports returning a valid `Columns` object
* Define `Tables.columns(x::MyTable)`: return a `Columns`, property-accessible object (perhaps the table itself if it naturally supports property-access to columns)

In addition, it can be helpful to define the following:
* `Tables.istable(::Type{MyTable}) = true`: there are runtime checks to see if an object implements the interface, but this can provide an explicit affirmation
* `Tables.schema(rows_or_columns)`: allow users to get the column names & types of `MyTable`'s `Rows` or `Columns` objects as a tuple of Symbols and tuple of types; by default, it will call `propertynames` on the first row or `Columns` object directly for names; column types will be `nothing` by default

The final question is how `MyTable` can be a "sink" for any other table type. The answer is quite simple: use the interface functions!

* Define a function or constructor that takes, at a minimum, a single, untyped argument and then calls `Tables.rows` or `Tables.columns` on that argument to construct an instance of `MyTable`

For example, if `MyTable` is a row-oriented format, I might define my "sink" function like:
```julia
function MyTable(x)
    Tables.istable(x) || throw(ArgumentError("MyTable requires a table input"))
    rows = Tables.rows(x)
    sch = Tables.schema(rows)
    names = sch.names
    types = sch.types
    # custom constructor that creates an "empty" MyTable according to given column names & types
    # note that the "unknown" schema case should be considered, i.e. when `sch.types => nothing`
    mytbl = MyTable(names, types)
    for row in rows
        # a convenience function provided in Tables.jl for "unrolling" access to each column/property of a `Row`
        # it works by applying a provided function to each value; see `?Tables.eachcolumn` for more details
        Tables.eachcolumn(sch, row, mytbl) do val, col, name, mytbl
            push!(mytbl[col], val)
        end
    end
    return mytbl
end
```

Alternatively, if `MyTable` is column-oriented, perhaps my definition would be more like:
```julia
function MyTable(x)
    Tables.istable(x) || throw(ArgumentError("MyTable requires a table input"))
    cols = Tables.columns(x)
    # here we use Tables.eachcolumn to iterate over each column in a `Columns` object
    return MyTable(collect(Tables.names(cols)), [collect(col) for col in Tables.eachcolumn(cols)])
end
```

Obviously every table type is different, but via a combination of `Tables.rows` and `Tables.columns` each table type should be able to construct an instance of itself.
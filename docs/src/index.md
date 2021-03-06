# FlatBuffers.jl Documentation

#### Overview
FlatBuffers.jl provides native Julia support for reading and writing binary structures following the google flatbuffer schema (see [here](https://google.github.io/flatbuffers/flatbuffers_internals.html) for a more in-depth review of the binary format).

The typical language support for flatbuffers involves utilizing the `flatc` compiler to translate a flatbuffer schema file (.fbs) into a langugage-specific set of types/classes and methods. See [here](https://google.github.io/flatbuffers/flatbuffers_guide_writing_schema.html) for the official guide on writing schemas.

This Julia package provides the serialization primitives used by code that has been generated by `flatc`. Since it was originally built without `flatc` support, it can also be used as a minimal set of macros to provide flatbuffer-compatible serialization of existing Julia types. This has led to the Julia code generated by `flatc` appearing somewhat more readable than for other languages.

For example, for this schema:
```
namespace example;

table SimpleType {
  x: int = 1;
}

root_type SimpleType;
```
the code generated by `flatc` looks like this:
```julia
module Example

using FlatBuffers
@with_kw mutable struct SimpleType
    x::Int32 = 1
end

# ... other generated stuff
end
```
If you don't want to write a schema, you can pepper your existing Julia types
with these macros and then call the functions below to produce flatbuffer-compatible
binaries.

#### Usage
`FlatBuffers` provides the following functions for reading and writing flatbuffers:
```
FlatBuffers.serialize(stream::IO, value::T) 
FlatBuffers.deserialize(stream::IO, ::Type{T})
```
These methods are not exported to avoid naming clashes with the `Serialization` module.
For convenience, there are also two additional constructors defined for each generated type:
* `T(buf::AbstractVector{UInt8}, pos::Integer=0)`
* `T(io::IO)`

Here is an example showing how to use them to serialize the example type above.
```julia
import FlatBuffers, Example

# create an instance of our type
val = Example.SimpleType(2)

# serialize it to example.bin
open("example.bin", "w") do f FlatBuffers.serialize(f, val) end

# read the value back again from file
val2 = open("example.bin", "r") do f Example.SimpleType(f) end
```
In addition, this package provides the following types and methods, which are useful
when inspecting and constructing flatbuffers:
* `FlatBuffers.Table{T}` - type for deserializing a Julia type `T` from a flatbuffer
* `FlatBuffers.Builder{T}` - type for serializing a Julia type `T` to a flatbuffer
* `FlatBuffers.read` - performs the actual deserializing on a `FlatBuffer.Table`
* `FlatBuffers.build!` - performs the actual serializing on a `FlatBuffer.Builder`

#### Methods for Generated Types
For a generated type `T`, in addition to the constructors mentioned above:
* if `T` has default values, constructors will be defined as per the `@with_kw` macro in [Parameters.jl](https://github.com/mauro3/Parameters.jl)
* `FlatBuffers.file_extension(T)` - returns the `file_extension` specified in the schema (if any)
* `FlatBuffers.file_identifier(T)` - returns the `file_identifier` specified in the schema (if any)
* `FlatBuffers.has_identifier(T, bytes)` - returns whether the given bytes contain the identifier for `T` at the offset designated by the flatbuffers specification
* `FlatBuffers.slot_offsets(T)` - an array containing the positions of the slots in the vtable for type `T`, accounting for gaps caused by deprecated fields
* `FlatBuffers.root_type(T)` - returns whether the type is designated as the root type by the schema. Also note however that no `root_type` definition is necessary in Julia; any of the generated `mutable struct`s can be a valid root table type.

#### Circular References
It's a bit unfortunate that the flatbuffers example uses mutually referential types, something which Julia doesn't have support for yet.
However, there is a [workaround](https://github.com/JuliaLang/julia/issues/269#issuecomment-68421745) - by modifying the
code generated by `flatc` slightly to add a type parameter, we can refer to a type that hasn't yet been defined.
```julia
FlatBuffers.@with_kw mutable struct Monster{T}
    # ...
    test::T = nothing
    # ...
end
```
In general though, try to avoid schemas which introduce these kinds of circular references.
For the full `Monster` example see the test suite [here](https://github.com/JuliaData/FlatBuffers.jl/blob/master/test/MyGame/Example/Monster.jl).

#### Internal Utilities
These functions are used by the code generated by `flatc`. Documentation is also included for many
internal methods and may be queried using `?` at the REPL.
* `@ALIGN T size_in_bytes` - convenience macro for forcing a flatbuffer alignment on the Julia type `T` to `size_in_bytes`
* `@with_kw mutable struct T fields...` - convenience macro for defining default field values for Julia type `T`
* `@UNION T Union{T1,T2,...}` - convenience macro for defining a flatbuffer union type `T`
* `@STRUCT struct T fields... end` - convenience macro for defining flatbuffer struct types, ensuring any necessary padding gets added to the type definition


# Property Classes

Property Classes represent data structures with dynamic runtime reflection
enabled on them, all of them inheriting from a common `PropertyClass` base.

On a high-level, they can be thought of as

```cc
class PropertyClass {
    /** Called before serialization. */
    virtual void OnPreSave() = 0;

    /** Called after serialization. */
    virtual void OnPostSave() = 0;

    /** Called before deserialization. */
    virtual void OnPreLoad() = 0;

    /** Called after deserialization. */
    virtual void OnPostLoad() = 0;
};
```

## Properties

Properties are the reflected C++ members of a class. Note that these are not
representative of the actual amount of members in total or their memory layout
at all.

Notoriously, properties can either be enum types, bitflag types, or regular
value types. They all have a name (which potentially differs from the actual C++
member name), a unique integer ID, and a set of configuration flags assigned.

Distinction between pointer types (raw and smart), references and value types is
performed.

The subsections of discuss all the important attributes and their importance.

### Names

It is valid to name properties after a nested path into the object it carries as
a value.

Consider the following example:

```cpp
union gid {
    unsigned __int64 m_full;
};
```

A property `gid m_id;` may in reality carry the name `m_id.m_full`, which
indicates that instead of the `gid` serialization routine, the
`unsigned __int64` routine should be used for the `m_full` member.

Such a `m_id.m_full` property would correspondingly also report
`unsigned __int64` as its type instead of `union gid`.

### Flags

Each property may carry a set of flag bits which influence its behavior. The
following is a non-exhaustive list of known flag bits and their purpose:

| Bit | Purpose                                                            |
| --- | ------------------------------------------------------------------ |
| 0   | Property value can be saved                                        |
| 1   | Property value may be copied/cloned                                |
| 2   | Property has public visibility                                     |
| 3   | Property may be transmitted to players over the network            |
| 4   | Property may be transmitted to privileged players over the network |
| 5   | Property may be persistently saved in database                     |
| 6   | The property is deprecated and must be skipped by the serializer   |
| 7   | ???                                                                |
| 8   | Property may only be re-serialized if modified                     |
| 9   | The `std::string` property stores a binary blob                    |
| 16  | Property must not be edited                                        |
| 17  | `std::string`s holding file names                                  |
| 18  | Set on properties of `class Color` type (some were forgotten)      |
| 20  | C-style bit flag enum type                                         |
| 21  | C++ enum class type where variants are scoped                      |
| 22  | `std::wstring`s holding i18n keys for GUI elements                 |
| 23  | `std::string`s holding string table key values                     |
| 24  | Property is a key for its containing class                         |
| 25  | Property is a key for a different class                            |
| 27  | Property that holds the name of its containing class               |
| 28  | Set on properties that have the `__BASECLASS` option               |

Note that bits 16-28 are mostly editor hints without any semantic features to
them.

### Container

Every property further has a container type associated with it. Containers are
dynamically-sized, homogenous collections of elements of the same type.

Containers are dynamically-sized, homogenous collections of elements of type
`T`.

A property always reports the `T` as its type. Strictly speaking, a
`List<std::string>` property actually has type `std::string` and the `"List"`
container assigned to it.

The actual containers directly map onto C++ types:

| Container | Property type | C++ type signature |
| --------- | ------------- | ------------------ |
| `Static`  | `T`           | `T`                |
| `Vector`  | `T`           | `std::vector<T>`   |
| `List`    | `T`           | `std::list<T>`     |

### Enum

Enums come in two variants:

- Plain, old C-style enums which are used as bit flag types.

- C++ enums with scoped variants

In both cases, the enum variants are runtime-accessible as [options](#options)
of variant name and value pairs.

### Options

Options are pairs of names and values that can be assigned to a
[properties](#properties) on an individual basis.

Most commonly, they occur with the aforementioned enum types, but they can also
be used with other types to map commonly expected values to names.

Traditionally, they hold either integral or string values.

A handful of them have a special meanings associated with them. These are
detailed in the following subsections.

#### Base classes

Some properties with `std::string` type have been observed to hold a special
option named `__BASECLASS` which holds the name of a class type.

They are believed to be editor hints without any semantic impact.

#### Default values

Properties can be assigned default values using a special `__DEFAULT` option
which holds a string representation of the default value a property should
default to if no other value is given during initialization.

- To set an integer property to value 1 - `"__DEFAULT": "1"`

- To set a `Foo`-typed property to enum variant `Foo::kBar` -
  `"__DEFAULT": "kBar"`

## Reflection

Reflection allows runtime interaction with property classes in one of the
following ways:

- Introspection of values with unknown types

- Iterating over properties

- Querying properties by name/ID

- Accessing aforementioned property metadata

- Check if class is subclass of a certain other type

- Clone/copy property values into different types

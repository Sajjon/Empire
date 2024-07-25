<div align="center">

[![Build Status][build status badge]][build status]
[![Platforms][platforms badge]][platforms]
[![Documentation][documentation badge]][documentation]

</div>

# Empire

A record store for Swift

Empire is an experiment in persistence.

- Schema is defined by your types
- Macro-based API that is both typesafe and low-overhead
- Built for Swift 6 (or at least that's the idea - currently cannot be compiled by Xcode 16b4 and crashes more recent stuff)
- Support for CloudKit's `CKRecord`
- Backed by a sorted-key index data store ([LMDB][LMDB])

> [!CAUTION]
> This is still a WIP.

```swift
import Empire

@IndexKeyRecord("name")
struct Person {
    let name: String
    let age: Int
}

let store = try Store(path: "/path/to/store")

try await store.withTransaction { context in
    try context.insert(Person(name: "Korben", age: 45))
    try context.insert(Person(name: "Leeloo", age: 2000))
}
	
let records = try await store.withTransaction { context in
    try Person.select(in: context, name: .lessThan("Zorg"))
}

print(record.first!) // Person(name: "Leeloo", age: 2000)
```

Limitations:

- Arbitrary key sorting is not yet supported, and could end up being impossible
- Macro-based query generation is hitting a [compiler bug][]
- Lots of Swift types don't yet support serialization, and even less support efficient sorting/queries

## Integration

```swift
dependencies: [
    .package(url: "https://github.com/mattmassicotte/Empire", branch: "main")
]
```

## Data Modeling and Queries

Empire uses a data model that is **extremely** different from a traditional SQL-backed data store. It is pretty unforgiving and can be a challenge, even if you are familiar with it.

Conceptually, you can think of every record as being split into two tuples: the "index key" and "fields".

### Keys

The index key is a critical component of your record. Queries are **only** possible on components of the index key.

```swift
@IndexKeyRecord("lastName", "firstName")
struct Person {
    let lastName: String
    let firstName: String
    let age: Int
}
```

The arguments to the `@IndexKeyRecord` macro define the properties that make up the index key. The `Person` records are sorted first by `lastName`, and then by `firstName`. The ordering of key components is very important. Only the last component of a query can be a non-equality comparison. If you want to look for a range of a key component, you must restrict all previous components.

```swift
// scan query on the first component
store.select(lastName: .greaterThan("Dallas"))

// constrain first component, scan query on the second
store.select(lastName: "Dallas", firstName: .lessThanOrEqual("Korben"))

// ERROR: an unsupported key arrangement
store.select(lastName: .lessThan("Zorg"), firstName: .lessThanOrEqual("Jean-Baptiste"))
```

The code generated for a `@IndexKeyRecord` type makes it a compile-time error to write invalid queries.

As a consequence of the limited query capability, you must model your data by starting with the queries you need to support. This can require denormalization, which may or may not be appropriate for your expected number of records.

### Format

Your types **are** the schema. The type's data is serialized directly to a binary form using code generated by the macro. Making changes to your types will make deserialization of unmigrated data fail. Non-key fields do not need any special properties, but they must conform to both the `Serialization` and `Deserialization` protocols.

Right now, there are limits what the kinds of types that can be used for fields and keys.

Supported Types: `String`, `UInt`, `Int`, `UUID`, `Data`, `Date`

> [!Note]
> `Date` encoding is lossy and only preserves accuracy down to the millisecond.

## Query Generation Workaround

Currently, the macro that generates type-safe queries [crashes the compiler][compiler bug]. Here's how you construct them manually in the meantime.

```swift
@IndexKeyRecord("lastName", "firstName")
struct Person {
    let lastName: String
    let firstName: String
    let age: Int
}

extension Person {
    static func select(in context: TransactionContext, lastName: String, firstName: String) throws -> [Self] {
        try context.select(query: Query(lastName, last: firstName))
    }

    static func select(in context: TransactionContext, lastName: String, firstName: ComparisonOperator<String>) throws -> [Self] {
        try context.select(query: Query(lastName, last: firstName))
    }

    static func select(in context: TransactionContext, lastName: ComparisonOperator<String>) throws -> [Self] {
        try context.select(query: Query(last: lastName))
    }
}
```

## `IndexKeyRecord` Conformance

The `@IndexKeyRecord` macro expands to a protocol conformance to the `IndexKeyRecord` protocol. You can use this directly, but it isn't easy. You have to handle binary serialization and deserialization of all your fields. It's also critical that you version your type's serialization format.

```swift
@IndexKeyRecord("name")
struct Person {
    let name: String
    let age: Int
}

// Equivalent to this:
extension Person: IndexKeyRecord {
    public typealias IndexKey = Tuple<String, Int>

    public static var schemaVersion: Int {
        1
    }

    public var indexKeySerializedSize: Int {
        name.serializedSize
    }

    public var fieldsSerializedSize: Int {
        age.serializedSize
    }

    public var indexKey: IndexKey {
        Tuple(name)
    }

    public func serialize(into buffer: inout SerializationBuffer) {
        name.serialize(into: &buffer.keyBuffer)
        age.serialize(into: &buffer.valueBuffer)
    }

    public init(_ buffer: inout DeserializationBuffer) throws {
        self.name = try String(buffer: &buffer.keyBuffer)
        self.age = try UInt(buffer: &buffer.valueBuffer)
    }
}

extension Person {
    // this will eventually have queries once I figure out a workaround
}
```

## `CloudKitRecord` Conformance

Empire supports CloudKit's `CKRecord` type via the `CloudKitRecord` macro. You can also use the associated protocol independently.

```swift
@CloudKitRecord
struct Person {
    let name: String
    let age: Int
}

// Equivalent to this:
extension Person: CloudKitRecord {
    public init(ckRecord: CKRecord) throws {
        try ckRecord.validateRecordType(Self.ckRecordType)

        self.name = try ckRecord.getTypedValue(for: "name")
        self.age = try ckRecord.getTypedValue(for: "age")
    }

    public func ckRecord(with recordId: CKRecord.ID) -> CKRecord {
        let record = CKRecord(recordType: Self.ckRecordType, recordID: recordId)

        record["name"] = name
        record["age"] = age

        return record
    }
}
```

Optionally, you can override `ckRecordType` to customize the name of the CloudKit record used. If your type also uses `IndexKeyRecord`, you get access to:

```swift
func ckRecord(in zoneId: CKRecordZone.ID)
```

## Questions

### Why does this exist?

I'm not sure! I haven't used [CoreData](https://developer.apple.com/documentation/coredata) or [SwiftData](https://developer.apple.com/documentation/swiftdata) too much. But I have used the distributed database [Cassandra](https://cassandra.apache.org) quite a lot and [DynamoDB](https://aws.amazon.com/dynamodb/) a bit. Then one day I discovered [LMDB][LMDB]. Its data model is quite similar to Cassandra and I got interested in playing around with it. This just kinda materialized from those experiments.

### Can I use this?

Sure!

### *Should* I use this?

User data is important. This library has a bunch of tests, but it has no real-world testing. I plan on using this myself, but even I haven't gotten to that yet. It should be considered *functional*, but experimental.

## Contributing and Collaboration

I'd love to hear from you! Get in touch via [mastodon](https://mastodon.social/@mattiem), an issue, or a pull request.

I prefer collaboration, and would love to find ways to work together if you have a similar project.

I prefer indentation with tabs for improved accessibility. But, I'd rather you use the system you want and make a PR than hesitate because of whitespace.

By participating in this project you agree to abide by the [Contributor Code of Conduct](CODE_OF_CONDUCT.md).

[build status]: https://github.com/mattmassicotte/Empire/actions
[build status badge]: https://github.com/mattmassicotte/Empire/workflows/CI/badge.svg
[platforms]: https://swiftpackageindex.com/mattmassicotte/Empire
[platforms badge]: https://img.shields.io/endpoint?url=https%3A%2F%2Fswiftpackageindex.com%2Fapi%2Fpackages%2Fmattmassicotte%2FEmpire%2Fbadge%3Ftype%3Dplatforms
[documentation]: https://swiftpackageindex.com/mattmassicotte/Empire/main/documentation
[documentation badge]: https://img.shields.io/badge/Documentation-DocC-blue
[LMDB]: https://www.symas.com/lmdb
[compiler bug]: https://github.com/swiftlang/swift/issues/74865

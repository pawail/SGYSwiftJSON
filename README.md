# SGYSwiftJSON
A library seeking to provide an automatic and type-safe approach to converting Swift types to and from JSON.

- [Summary](#summary)
- [Quick Start](#quick-start)
- [Serialization](#serialization)
 - [Common Serialization Problems](#serialization-problems)
- [Deserialization](#deserialization)
 - [Common Deserialization Problems](#deserialization-problems)
- [Examples](#examples)

<a name="summary"></a>
## Summary
SGYSwiftJSON is a library that seeks to dramatically simplify serialization and deserialization of **Swift only** models. The primary goal of this library is to eliminate the majority of the code required to convert arbitrary objects into JSON and vice versa. This includes recursive conversion of types contained in collections, dictionaries, and complex objects. Out-of-the-box functionality includes support for the majority of common Foundation types and a ready to inherit base class for complex types (*not required, but easier*).  Protocols are provided which allow extending functionality to unusual objects.

<a name="quick-start"></a>
## Quick Start
The majority of models written with JSON serialization in mind are already supported.  Any object graph that conforms to the following should work out-of-the-box:
* All collections are the  `Array`, `Set`, `NSArray`, or `NSMutableArray` types with an element type that adheres to this collection of rules.
* All dictionaries have a `String` or `NSString` key type and a value type which adheres to this collection of rules.
* All complex types conform to `JSONKeyValueCreatable`.  This is most easily achieved by using `JSONCreatableObject` as the base class.
* All numeric types are `NSNumber`, `NSDecimalNumber` or can be bridged to `NSNumber` and are not declared optional.

If you do not wish to have to adhere to the above limitations then it is possible to extend most other types using the defined protocols. Details on these protocols and how they're evaluated during serialization and deserialization can be found below.

*Note: Several of the above limitations do not apply or are much more lenient if you do not wish to implement deserialization.*

<a name="serialization"></a>
### Serialization
Serialization is supported via protocols and the use of Swift's `Mirror`. Any object passed to be serialized is checked for the following conditions:
 1. Conforms to `JSONProxyProvider` - The `jsonProxy` property of the object will be retrieved and passed through this same logic tree.
 2. Conforms to `JSONLeafRepresentable` - Objects adhering to this protocol can be represented as a JSON leaf object.  I.e `NSString`, `NSNumber`, or `NSNull`.  Beyond the 3 leaf values accepted the structs `String`, `Bool`, `Int`, `Float`, and `Double` conform to this protocol.
 3. Is an `NSDate` -  If the object is an `NSDate` and does not conform to any of the above protocols then the *dateConversionBlock*, if provided, is used to convert to `JSONLeafRepresentable` or the property is skipped if no block exists.
 4. Conforms to `SGYDictionaryReflection` - The object will be converted to a dictionary of strings keys and object values. The generic `Dictionary` and `NSDictionary` both adhere to this protocol.
 5. Conforms to `SGYCollectionReflection` - The object's contained elements will be converted and put into an array. `Array`, `NSArray`, and `Set` adhere to this protocol.
 6. None of the above - The object's property's and values will be enumerated using `Mirror` and converted to a dictionary.
 
<a name="serialization-problems"></a>
#### Common Serialization Problems
* `Enum` - Enumerations are most appropriately handled by having the enumeration conform to `JSONLeafRepresentable` and return the `JSONLeafValue` struct initialized with their string or number *rawValue*.
* `NSDate` - There are 2 strategies for handling `NSDate` based on the project:
 * If all dates are represented using the same format then extend `NSDate` to conform to `JSONLeafRepresentable` and construct `JSONLeafValue` using the specific string or numeric representation.
 * If dates use different formatting for different situations then instead initialize a separate `SGYJSONSerializer` instance for each format and assign an appropriate *dateConversionBlock*.

<a name="deserialization"></a>
### Deserialization
Deserialization is considerably more difficult than serialization as it requires all types have a parameterless initializer, can assign arbitrary values, and are able to report the types they contain.  Upon deserialization of an `NSArray` or `NSDictionary` (the only objects produced by `NSJSONSerialization`) the following logic is performed:
 1. The type arguments are determined by the protocol that the object being deserialized into conforms:
  *  If the object conforms to `JSONKeyValueCreatable` the object's properties and type values are determined using `Mirror`.  The object produced by `NSJSONSerialization` must be an `NSDictionary`or an error is thrown.
  *  If the object conforms to `JSONDictionaryCreatable` the object's key and value type are retrieved using the protocol's *keyValueTypes* property. The object produced by `NSJSONSerialization` must be an `NSDictionary`or an error is thrown.
  *  If the object conforms to `JSONCollectionCreatable` the object's element type is retrieved using the protocol's *elementType* property. The object produced by `NSJSONSerialization` must be an `NSArray`or an error is thrown.
 2. If the value to be deserialized into is an array then all values will be converted to the array's containing `Element` type.  Similarly, dictionaries have the containing values converted to their `Value` type.  For complex objects the value is converted using its `Mirror` property representation.  This conversion is done using the following logic:
  1. If the declared type is `AnyObject` or the declared type matches the deserialized type then the deserialized type is assigned directly.
  2. If the deserialized value is a leaf value then the deserialized type must conform to `JSONLeafConvertable` and will be constructed using the leaf value and assigned.  Otherwise the deserialized value is skipped.
  3. If the deserialized value is the `[AnyObject]` type and the declared type is `JSONCollectionCreatable` an array will be initialized and returned using the array conversion logic.  Otherwise the deserialized value is skipped.
  4. If the deserialized value is the `[String: AnyObject]` type and the declared type is `JSONDictionaryCreatable` an array will be initialized and returned using the dictionary conversion logic.  Otherwise the deserialized value is skipped.
 
<a name="deserialization-problems"></a>
### Common Deserialization Problems
* Inheriting from `JSONCreatableObject` and assigning value types (ie. `CGSize`, `Int`, etc) - Using `JSONCreatableObject` as a `JSONKeyValueCreatable` base class greatly simplifies implementing the protocol.  But since it utilizes `NSObject`'s *setValue:forKey:* method it is vulnerable to the same limitations- the inability to assign values that do not inherit from `NSObject`. There are two strategies for dealing with this limitation:
 * If the value can be automatically bridged to an `NSObject` subtype (ie. `Int` to `NSNumber`) then it will be bridged.  **However** the property cannot be defined as optional or an error is still thrown.  Instead a variable with a default value should be defined.
 * For more complicated value types the only safe alternative is overriding *setValue:property:* and assigning the value directly.

<a name="examples"></a>
## Examples
For demonstration I'll consider a `Person` class using 'Swift type' properties.  The use of this class is irrelevant but contains many of the common properties used in web models.
```swift
enum Color { case Red, Blue, Green, Yellow }

class Person {
 var name: String?
 var birthdate: NSDate?
 var favoriteColor: Color?
 var friends: [Person]?
 var followersCount: Int?
}
```
#### Serialization
This class can nearly be serialized as is (the `Color` enum and `NSDate` are the only problem types). We must modify `Color` so that it can be associated with some JSON value: 
```swift
enum Color: Int, JSONLeafRepresentable { // 1 & 2
 case Red, Blue, Green, Yellow
 public var jsonLeafValue: JSONLeafValue? { return JSONLeafValue(self.rawValue) } // 3
}
```
In the block above we:
 1. Gave `Color` a raw value type of `Int`.  This way we can convert `Color` to a JSON number and vice versa.
 2. Made `Color` conform to `JSONLeafRepresentable`.  This tells the serializer the type has a JSON leaf value representation.
 3. Completed the conformance to `JSONLeafRepresentable` by providing a property that returns a `JSONLeafValue` enum.

Next serialize. Assume `personObject` is some instance of `Person` filled with arbitrary values:
```swift
let serializer = SGYJSONSerializer() // 1
let formatter = NSDateFormatter() // 2
serializer.dateConversionBlock = { (date: NSDate) -> JSONLeafValue in
 return JSONLeafValue(formatter.stringFromDate(date))
}
do {
 let jsonData = try serializer.serialize(personObject) // 3
} catch err as NSError {
 // Optionally catch specific thrown errors // 4
}
```
In the above block we:
 1. Created a new `SGYJSONSerializer` instance.
 2. Created an `NSDateFormatter` and assigned a `dateConversionBlock` to the serializer in order to convert our `NSDate` property.
 3. Serialize `Person` instance into JSON data.
 4. Caught the generic `NSError`.  Optionally you may catch the specific errors that serialization is documented to throw in order to get a better idea of the failure.

#### Deserialization
In order to deserialize the same `Person` class some more additions are required. We already modified our `Color` enum to provide a valid JSON leaf value.  Now we must modify it to be constructable from a JSON leaf value by adhering to `JSONLeafCreatable`:
```swift
enum Color: Int, JSONLeafRepresentable, JSONLeafCreatable {
 case Red, Blue, Green, Yellow
 
 init?(jsonValue: JSONLeafValue) {
    switch jsonValue {
    case .Number(let number):
        self.init(rawValue: number.integerValue) // Initialize with Int
    case .String(let string):
        guard let int = Int(string as String) else { return nil } // Attempt parsing Int from String
        self.init(rawValue: int) // Initialize with parsed Int
    case .Null(_):
        return nil // Fail outright
    }
 }
 
 public var jsonLeafValue: JSONLeafValue? { return JSONLeafValue(self.rawValue) } // Bridges our Int value to NSNumber
}
```
Next we must make a few additions to our `Person` class:
```swift
class Person: JSONCreatableObject { // 1
 var name: String?
 var birthdate: NSDate?
 var favoriteColor: Color?
 var friends: [Person]?
 var followersCount: Int?
 
 override func setValue(value: Any, property: String) throws { // 2
    if property == "favoriteColor" { color = value as? Color }
    else { try super.setValue(value, property: property) }
 }
}
```
We made two changes:
 1. Inherited `Person` from `JSONCreatableObject`.  We need a way to conform to `JSONKeyValueCreatable` and this class does the majority of the work for us.
 2. Since our `Color` enum conforms to `JSONLeafCreatable` it will be constructed from a JSON leaf value.  But enums cannot be assigned via key-value-coding (which `JSONCreatableObject` uses).  So instead we bypass KVC and assign the value directly.

Finally we deserialize:
```swift
let deserializer = SGYJSONDeserializer() // 1
let dateFormatter = NSDateFormatter() // 2
deserializer.dateConversionBlock = { (jsonValue: AnyObject) -> NSDate? in
 guard let date = jsonValue as? NSDate else { return nil }
 return dateFormatter.dateFromString(date)
}

do {
 let deserializedPerson: Person = try deserializer.deserialize(jsonData) // 3
} catch let error as NSError {
  // Optionally catch specific errors // 4
}
```
Here we:
 1. Created a new instance of `SGYJSONDeserializer`.
 2. Assigned the inverse of the `dateConversionBlock` we assigned during serialization.
 3. Deserialize into a `Person` instance. Assume `jsonData` is some arbitrary JSON.
 4. Caught the generic `NSError`.  Optionally you may catch the specific errors that deserialization is documented to throw in order to get a better idea of the failure.

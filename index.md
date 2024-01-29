# Kotlin Multiplatform links and documentations
## Intro
Kotlin Multiplatform is an exciting technology. I want to keep track and share some of the posts and links I found and find interesting.  
This is really a reminder page for me to have notes and snippets at the same place.

## Writing Swift-Friendly Kotlin Multiplatform APIs
### Part 1
[https://betterprogramming.pub/writing-swift-friendly-kotlin-multiplatform-apis-part-i-1173ec405a20](https://betterprogramming.pub/writing-swift-friendly-kotlin-multiplatform-apis-part-i-1173ec405a20)  
Introduction on how Kotlin is translated to Objective-C and then used in Kotlin. Some quick and easy to make the Kotlin code more `SWiftable`

From:  
```
// KOTLIN API CODE
fun findElementInList(elem: Int, list: List<Int>): Int = list.indexOf(elem)
```
```
// EXPORTED OBJ-C HEADER
__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("ExampleKt")))
@interface SharedExampleKt : SharedBase
+ (int32_t)findElementInListElem:(int32_t)elem list:(NSArray<SharedInt *> *)list __attribute__((swift_name("findElementInList(elem:list:)")));
@end
```
```
// SWIFT CLIENT CODE
let index = ExampleKt.findElementInList(elem: 1, list: [1, 2, 3])
```

to:
```
// KOTLIN API CODE
@OptIn(ExperimentalObjCName::class)
@ObjCName("find")
fun findElementInList(@ObjCName("_")elem: Int, @ObjCName("in") list: List<Int>): Int = list.indexOf(elem)
```
```
// EXPORTED OBJ-C HEADER
+ (int32_t)findElem:(int32_t)elem in:(NSArray<SharedInt *> *)in __attribute__((swift_name("find(_:in:)")));
```
```
// SWIFT CLIENT CODE
let index = ExampleKt.find(1, in: [1, 2, 3])
```
### Part 2 - Name clash
[https://betterprogramming.pub/writing-swift-friendly-kotlin-multiplatform-apis-part-ii-bbee12090415](https://betterprogramming.pub/writing-swift-friendly-kotlin-multiplatform-apis-part-ii-bbee12090415)
Kotlin has namespaces but Objective-C doesn't. This can lead to some name clash. The Kotlin compiler will add an underscore `_` to one of the conflicting name.
```
// KOTLIN API CODE
package io.aoriani.network

class Item

...

package io.aoriani.models

class Item
```
```
// EXPORTED OBJ-C HEADER
__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Item")))
@interface SharedItem : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
@end

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Item_")))
@interface SharedItem_ : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
@end
```

To fix this, we can add the `@ObjCName` on top of some to have it a different name in the Swift side.
```
// KOTLIN API CODE
package io.aoriani.network

import kotlin.experimental.ExperimentalObjCName
import kotlin.native.ObjCName

@OptIn(ExperimentalObjCName::class)
@ObjCName("ItemResponse")
class Item

...

package io.aoriani.Models

class Item
```
```
// EXPORTED OBJ-C HEADER
__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("ItemResponse")))
@interface SharedItemResponse : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
@end

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Item")))
@interface SharedItem : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
@end
```
```
// SWIFT CLIENT CODE
let itemDomainModel = Item()
let itemNetworkResponse = ItemResponse()
```

The same can be done for functions
```
// KOTLIN API CODE
fun sort(data: List<Int>){}
fun sort(data: Map<String, Int>){}

// EXPORTED OBJ-C HEADER
+ (void)sortData:(NSArray<SharedInt *> *)data __attribute__((swift_name("sort(data:)")));
+ (void)sortData_:(NSDictionary<NSString *, SharedInt *> *)data __attribute__((swift_name("sort(data_:)")));

// SWIFT CLIENT CODE
ExampleKt.sort(data: [1, 2, 3])
ExampleKt.sort(data_: ["A":1, "B": 2])
```
to:
```
// KOTLIN API CODE
fun sort(listOfInts: List<Int>){}
fun sort(mapOfStringToInt: Map<String, Int>){}

// EXPORTED OBJ-C HEADER
+ (void)sortListOfInts:(NSArray<SharedInt *> *)listOfInts __attribute__((swift_name("sort(listOfInts:)")));
+ (void)sortMapOfStringToInt:(NSDictionary<NSString *, SharedInt *> *)mapOfStringToInt __attribute__((swift_name("sort(mapOfStringToInt:)")));

// SWIFT CLIENT CODE
ExampleKt.sort(listOfInts: [1, 2, 3])
ExampleKt.sort(mapOfStringToInt: ["A" : 1, "B": 2])
```


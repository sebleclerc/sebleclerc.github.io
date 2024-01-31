# Kotlin Multiplatform links and documentations
## Intro
Kotlin Multiplatform is an exciting technology. I want to keep track and share some of the posts and links I found and find interesting.  
This is really a reminder page for me to have notes and snippets at the same place.

## Writing Swift-Friendly Kotlin Multiplatform APIs
### Part 1 - Intro
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
### Part 2 - Clashing
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
### Part 3 - Disappearing Types (TBC)
[https://betterprogramming.pub/writing-swift-friendly-kotlin-multiplatform-apis-part-iii-e8ab8327260e](https://betterprogramming.pub/writing-swift-friendly-kotlin-multiplatform-apis-part-iii-e8ab8327260e)
Using a `Map` (or other similar) of objects on the Kotlin side, it's translated to a `id<Some>` in the Objective-C side. But, when consumed in Swift, that id type gets erased. For example:
```
// KOTLIN API CODE
class Order {
    fun setOptions(options: Map<Option, Int>) {}
}

// EXPORTED OBJ-C HEADER
__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Order")))
@interface SharedOrder : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
- (void)setOptionsOptions:(NSDictionary<id<SharedOption>, SharedInt *> *)options __attribute__((swift_name("setOptions(options:)")));
@end

// SWIFT CLIENT CODE
let myOrder = Order()
myOrder.setOptions(options: ["Not the intended type" : -1])
```
Even if the Kotlin side wants a map of `Option,Int`, which is not taken into account when calling this function from the Swift side.

### Part 4 - Convenience
[https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-iv-d962c31edfee](https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-iv-d962c31edfee)  
Adding specific code to target Swift code.

Do not use `interface`
```
// KOTLIN API CODE
interface Person {
    val firstName: String
    val lastName: String
}

fun Person.fullName() = "$firstName $lastName"

// HEADER "TRANSLATED" TO SWIFT
public protocol Person {
    var firstName: String { get }
    var lastName: String { get }
}

public class ExampleKt : KotlinBase {
    open class func fullName(_ receiver: Person) -> String
}
```
Use `abstract class`
```
// KOTLIN API CODE
abstract class Person(
    val firstName: String,
    val lastName: String
)

fun Person.fullName() = "$firstName $lastName"

// HEADER "TRANSLATED" TO SWIFT
open class Person : KotlinBase {
    public init(firstName: String, lastName: String)
    open var firstName: String { get }
    open var lastName: String { get }
}

extension Person {
    open func fullName() -> String
}
```

Define platform specific extensions
```
// KOTLIN API CODE

// commonMain
import io.ktor.http.Url

object ServiceConfig {
    val baseUrl = Url("https://api.server.com/service")
}

// androidMain
import io.ktor.http.Url
import java.net.URL

fun Url.toURL(): URL = URL(this.toString())

// iosMain
import io.ktor.http.Url
import platform.Foundation.NSURL

fun Url.toURL(): NSURL  = NSURL(string = toString())

// Android Client code 
val url = ServiceConfig.baseUrl.toURL()
val connection = url.openConnection()
```

TBC

### Part 5 - Exceptions
[https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-v-exceptions-425998f579e1](https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-v-exceptions-425998f579e1)

Exceptiosn thrown from function can't be catched by default in the Swift code. We need to annotate Kotlin APIS with `@Throws` with a list of all possible exceptions that could be thrown by the code.

```
// KOTLIN API CODE
import io.ktor.utils.io.errors.IOException

class Reader {
    @Throws(IOException::class)
    fun read(): Int {
        throw IOException("No data to read")
    }
}

// EXPORTED OBJ-C HEADER
__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Reader")))
@interface SharedReader : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));

/**
 * @note This method converts instances of IOException to errors.
 * Other uncaught Kotlin exceptions are fatal.
*/
- (int32_t)readAndReturnError:(NSError * _Nullable * _Nullable)error __attribute__((swift_name("read()"))) __attribute__((swift_error(nonnull_error)));
@end

// HEADER "TRANSLATED" TO SWIFT
public class Reader : KotlinBase {
    public init()
    /**
     * @note This method converts instances of IOException to errors.
     * Other uncaught Kotlin exceptions are fatal.
    */
    open func read() throws -> Int32
}

// Swift code to used
func readData() -> Int32 {
    let reader = Reader()
    do {
        let value = try reader.read()
        return value
    } catch let error as NSError {
        print("NSError: \(error)")
        
        switch error.kotlinException {
        case let ioException as Ktor_ioIOException:
            print ("Caught IOException")
            print (ioException.message ?? "")
            ioException.printStackTrace()
        case let illegalStateException as KotlinIllegalStateException:
            print ("Caught IllegalStateException")
            print (illegalStateException.message ?? "")
        default:
            print ("Caught something")
        }
        return -1
    }
}
```

### Part 6 -Enum and sealed classes
[https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-vi-e2c802238ae4](https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-vi-e2c802238ae4)

There are no real way to define an enum and correctly translate to Swift enum. The best way is to define them in Kotlin as `sealed class` and then create a wrapper in the Swift cdoe
```
// KOTLIN API CODE
sealed class ReturnValue {
    object Loading : ReturnValue()
    class Success(val data: String) : ReturnValue()
    class Error(val errorCode: Int) : ReturnValue()
}

// SWIFT CLIENT CODE
enum ReturnValueSwift {
    case loading
    case success(data: String)
    case error(errorCode: Int)
}

extension ReturnValueSwift {
    init?(_ value: ReturnValue) {
        switch value {
        case is ReturnValue.Loading:
            self = .loading
        case let success as ReturnValue.Success:
            self = .success(data: success.data)
        case let error as ReturnValue.Error:
            self = .error(errorCode: Int(error.errorCode))
        default:
            return nil
        }
    }
}

func returnValueTest() {
    if let result = ReturnValueSwift(ReturnValue.Loading.shared) {
        switch result {
        case .loading:
            print("loading")
        case let .success(data):
            print("success \(data)")
        case let .error(errorCode):
            print("error \(errorCode)")
            
        }
    }
}
```


### Part 7 - Coroutines
[https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-vii-coroutines-5c76da7a4e72](https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-vii-coroutines-5c76da7a4e72)

By default, suspend functions are translated with a completion handler AND an async version ([Apple Doc](https://developer.apple.com/documentation/swift/calling-objective-c-apis-asynchronously)).

But, the cancel function on the task doesn't work as expected.

TBC

### Part 8 - Generics
[https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-viii-generics-da682b6aa748](https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-viii-generics-da682b6aa748)

Generic answer, it's not really supported.

Other interesting reading:
- [Kotlin Native Interop Generics](https://medium.com/@kpgalligan/kotlin-native-interop-generics-15eda6f6050f)
- Kotlin for iOS developers from KotlinConf' (link to header)

### Part 9 - Flow
[https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-ix-flow-d4b6ada59395](https://proandroiddev.com/writing-swift-friendly-kotlin-multiplatform-apis-part-ix-flow-d4b6ada59395)

Considering this Kotlin code:
```
// KOTLIN API CODE
class Clock {
    fun ticker(): Flow<String> = flow {
        while (currentCoroutineContext().isActive) {
            val now: Instant = Clock.System.now()
            val thisTime: LocalTime = now.toLocalDateTime(TimeZone.currentSystemDefault()).time
            emit(thisTime.toString())
            delay(1_000)
        }
    }
}
```
Problems? The generic type will be lost because `Flow` is an interface and will lost generic type `String`. Swift will need to cast (`as!`), each time he will need to access a value. Also, it will be impossible to cancel the flow.

Solution?
- We need to return a class instead of the interface. Generics are handle in classes.
- We will also need to create a wrapper for `Flow`. We will have a single function to start and cancel the flow
- Create an `iosMain` only extension to the wrapper and hide the equivalent function to ObjC
```
// KOTLIN API CODE
class Watch {
    @OptIn(ExperimentalObjCRefinement::class)
    @HiddenFromObjC // Will not be exported to Obj-C and Swift
    fun ticker(): Flow<String> = flow {
        while (currentCoroutineContext().isActive) {
            val now: Instant = Clock.System.now()
            val thisTime: LocalTime = now.toLocalDateTime(TimeZone.currentSystemDefault()).time
            emit(thisTime.toString())
            delay(1_000)
        }
    }
}

class FlowWrapper<out T> internal constructor(private val scope: CoroutineScope,
                                              private val flow: Flow<T & Any>){
    private var job: Job? = null
    private var isCancelled = false

    /**
     *  Cancels the flow
     */
    fun cancel() {
        isCancelled = true
        job?.cancel()
    }

    /**
     * Starts the flow
     * @param onEach callback called on each emission
     * @param onCompletion callback called when flow completes. It will be provided with a non
     * nullable Throwable if it completes abnormally
     */
    fun collect(
        onEach: (T & Any) -> Unit,
        onCompletion: (Throwable?) -> Unit
    ) {
        if (isCancelled) return
        job = scope.launch {
            flow.onEach(onEach).onCompletion { cause: Throwable? -> onCompletion(cause) }.collect()
        }
    }
}

internal fun <T> Flow<T&Any>.wrap(scope: CoroutineScope = MainScope()) = FlowWrapper(scope, this)

// iosMain
@ObjCName("ticker")
fun Watch.wrappedTicker() = this.ticker().wrap()
```

Which will be used in Swift this way:
```
// SWIFT CLIENT CODE
func test() {
    let watch = Watch()
    let handle = watch.ticker()
    handle.collect {
        value in print(value)
    } onCompletion: { throwable in
        print("Complete : \(throwable?.description() ?? "<Success>")")
    }

    DispatchQueue.main.asyncAfter(deadline: .now() + 10){
        print("CANCELLING")
        handle.cancel()
    }
}
```

And even better? Support for Combine! But considering the Combine framework is Swift-only, we will need to write some Swift code.
> First, we extend KotlinThrowable — the exported version of Throwable — to conform to the Error protocol so we can forward any exceptions on our flow. Second, we need to implement a Publisher that will wrap our wrapper to our flow.
> Next, our publisher defines the Associated Types for the Publisher protocol by setting Output to our generic type argument and Failure to KotlinThrowable. Then, we need to implement the method receive [...]

```
import Combine

extension KotlinThrowable: Error {}

class FlowPublisher<T: AnyObject>: Publisher {
    typealias Output = T
    typealias Failure = KotlinThrowable
    
    private let wrappedFlow: FlowWrapper<T>
    
    init(wrappedFlow: FlowWrapper<T>) {
        self.wrappedFlow = wrappedFlow
    }
    
    func receive<S>(subscriber: S) where S : Subscriber, KotlinThrowable == S.Failure, T == S.Input {
        let subscription = FlowSubscription(wrappedFlow: wrappedFlow)
        
        subscriber.receive(subscription: subscription)
        
        wrappedFlow.collect { value in
           let demand: Subscribers.Demand = subscriber.receive(value)
           // Dealing with demand is left as exercise for the reader
        } onCompletion: { throwable in
            subscriber.receive(completion: throwable == nil ? .finished : .failure(throwable!))
        }
    }
    
    class FlowSubscription: Subscription {
        
        private let wrappedFlow: FlowWrapper<T>
        
        init(wrappedFlow: FlowWrapper<T>) {
            self.wrappedFlow = wrappedFlow
        }
        
        func request(_ demand: Subscribers.Demand) {
            // Dealing with demand is left as exercise for the reader
        }
        
        //Progates the cancel 
        func cancel() {
            wrappedFlow.cancel()
        }
    }
}

func flow<T>(_ wrapper: FlowWrapper<T>) -> FlowPublisher<T> {
    return FlowPublisher(wrappedFlow: wrapper)
}

```



## Kotlin/Multiplatform for iOS developers : state & future by Salomon Brys
[https://www.youtube.com/watch?v=j-zEAMcMcjA&t=851s](https://www.youtube.com/watch?v=j-zEAMcMcjA&t=851s)

This presentation is part of KotlinConf' 23

Also how do deploy using SPM. Using KMM-Bridge

Inside the `Package.swift` for SPM, add a dependency to the shared framework and add Swift files to help with bridging.

Check `Multiple Namespace Generation`, see KT-42248 for more informations.

To use `Flows` in Swift code, we can create an `MPFlow` class to encapsulate that flow. (~ 11:00)

## Tools
### Sourcekitten
[GitHub](https://github.com/jpsim/SourceKitten)

It's a tool that translate the `shared.h` header file generated by KMP and output the Swift version of it.

Interesting read:
- [https://towardsdev.com/writing-swift-friendly-kotlin-multiplatform-apis-extra-4c1d7a8d58f2](https://towardsdev.com/writing-swift-friendly-kotlin-multiplatform-apis-extra-4c1d7a8d58f2)
- [https://quiet.github.io/quiet-blog/2018/08/13/Objective-C-Swift-Documentation.html](https://quiet.github.io/quiet-blog/2018/08/13/Objective-C-Swift-Documentation.html)

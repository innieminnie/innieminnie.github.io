---
layout: post
title: "Copy-on-Write(CoW) in Swift"
date: 2021-03-02 01:37:00
categories: Swift Optimization CopyonWrite CoW
---
Copy-on-Write(CoW) in Swift

1. 개요
2. 동작방식
3. 사용예시(Swift의 Collection)
4. 사용예시(Custom Value Types)


---
### 개요

  Copy-on-Write는 Swift의 Performance를 향상시키는 기술 중 하나입니다. 
  Swift에선 주로 <b>구조체(Structure)를 복사할 때 성능을 향상</b>시킬 때 사용되는 기술로, 크기가 큰 값을 복사하는 상황에서 (예를 들어, 10000개의 elements를 담고 있는 array를 다른 변수에 복사해야하는 경우) 빛을 발하는 기술입니다.

---


### 동작방식

  10000개의 elements를 담고 있는 arrayA를 다른 변수 arrayB에서 에 복사해야하는 경우로 설명하자면, <b>실질적인 elements의 복사가 발생하는 시점</b>이 변수 arrayB가 생성되는 순간이 아닌, <b>arrayB가 변경이 발생되는 시점</b>으로 미루는 것입니다. 

  변경 발생 이전까지는 서로 같은 공간(메모리영역)에 배치되어있는 값을 가리키고 있습니다. 이로써 arrayA와 다른부분이 발생하지 않을 때는 굳이 같은 값들을 변수 할당 시에 복사하는 불필요한 상황을 제거하고, arrayB의 변경이 발생함에 따라 arrayA와 arrayB가 달라졌을 때 복사를 진행하게 되어 Copy-on-Write(쓰기 시 복사)라고 합니다.

   Swift의 기본 Collection(Array, Set, Dictionary...) 은 내부적으로 Copy-on-Write로 구현되어있다는 점을 유의해야합니다! 사용자가 정의한 타입에 대해선 Copy-on-Write는 함께 고려해야합니다.

---


### 사용예시(Swift의 Collection)

해당 예시코드는 <https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst#advice-use-copy-on-write-semantics-for-large-values> 를 바탕으로 함을 명시합니다.

```swift
// Copy-on-Write가 활용되지 않은 코드

protocol P {}
struct Node: P {
  var left, right: P?
}

struct Tree {
  var node: P?
  init() { ... }
}
```

​	ㄴ Tree 구조체는 Node구조체(값 타입)을 가지고 있습니다. 또한 Node는 프로토콜 P를 채택하고 있고, 프로퍼티인 left, right 또한 프로토콜 P를 채택합니다. 위와 같을 때 Tree가 복사되어야한다면 Tree의 전체가 복사되어야합니다. Copy-on-Write가 적용되는 부분이 없기 때문입니다. 



```swift
// Array에 내장된 Copy-on-Write(CoW) 활용
struct Tree: P {
  var node: [P?]
  init() {
    node = [thing]
  }
}
```

​	ㄴ 가장 쉽게 copy-on-write를 적용하는 방법은 copy-on-write를 지원하는 자료구조를 활용하는 것입니다. 첫번째 상황에선 left,right로 Node 구조체에 존재하던 요소들이 위의 코드에서 [P?]로 감싸면서 copy-on-write를 내부적으로 사용하고 있는 배열을 활용했습니다. 



​	개선하기 이전의 코드에선 Tree를 복사할 때, Tree구조체의 크기에 따라 달라져 O(n)의 복사비용이 들지만, 개선 이후에는 같은 메모리를 가리키는 새로운 tree를 복사하기에 O(1)의 복사비용을 갖는 효과가 생깁니다.



​	하지만 위의 경우에서는 단순히 Array로 wrapping하는 것은 크게 두 가지 문제점을 갖고 있습니다. 

​	1) Array로 감쌌지만, Array의 기본 제공 메소드인 append()나 count()와 같은 것을 사용할 필요가 없는 경우가 있습니다. 위의 코드에서도 Node의 left와 right 가 같은 타입이기에 Tree내의 node라는 배열을 만들었지만,left와 right만 필요하다면 node배열이 제공하는 메소드의 많은 것들을 낭비하게 됩니다.

​	2) Array는 프로그램의 안정성을 보장하기 위해 지속적으로 check하고, Objective-C와 연동하기 위한 코드를 갖고 있습니다. Array의 크기를 늘려야하는 상황을 대비하기 위해서  지속적으로 runtime checks가 발생하는데 이러한 부분이 속도 저하를 발생시킬 수 있습니다.



  <b>Array와 같은 Swift의 기본 Collection이 copy-on-write를 지원하는 것은 알고 있으면 유용한 정보이지만, Collection의 활용도가 어느정도가 되느냐에 따라 copy-on-write 기능이 다른 측면과 비교해봤을 때 중요도가 낮아질 수 도 있기에 유의해야합니다.</b>

---


### 사용예시(Custom Value Types)
사용자가 정의한 타입에서 CoW를 구현해, 사용예시(Swift의 Collection)에서 발생하던 2가지의 문제점을 개선해보겠습니다.

```swift
final class Ref<T> { 
  var val: T
  init(_ v: T) {val = v}
}

struct Box<T> {
    var ref: Ref<T>
    init(_ x: T) { ref = Ref(x) }

    var value: T {
        get { return ref.val }
        set {
          if !isKnownUniquelyReferenced(&ref) {
            ref = Ref(newValue)
            return
          }
          ref.val = newValue
        }
    }
}
```

​	ㄴ Referece Type인 Class를 만들고 (Ref<T>), Class를 wrapping하는 Value Type의 Struct를 하나 만듭니다(Box<T>).



​	Struct (Box)는 값 타입이므로 다른 변수에 값을 할당할 때 값들이 복사되지만, 구조체 내의 프로퍼티 Class(Ref)는 다른 변수에 할당되어도 값이 저장되어있는 메모리 영역을 공유합니다.



​	BoxA를 복사한 BoxB가 있다고 할 때, BoxA의 값이 변경되면 그 시점에 새로운 ref 인스턴스를 만듭니다. 값을 변경할 때 새로운 Ref 영역을 만들어주기 위해

```swift
 if !isKnownUniquelyReferenced(&ref) {
   //isKnownUniquelyReferenced는 strong reference가 하나일 때 true
            ref = Ref(newValue)
            return
          }
```

가 활용됩니다.



```swift
struct Toy {
	var userAge = 3
}




let toy = Toy()

let kidsBox = Box(value: toy)
var childrenBox = kidsBox 

childrenBox.value.userAge = 5 // 새로운 Ref가 생성되는 시점

```
---


#### 참고:

<https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst#advice-use-copy-on-write-semantics-for-large-values>

<https://marcosantadev.com/copy-write-swift-value-types/>

<https://www.hackingwithswift.com/example-code/language/what-is-copy-on-write>








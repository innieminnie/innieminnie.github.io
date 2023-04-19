## Race Condition: 여러 곳에서 하나의 자원을 두고 경쟁하는 문제상황

비동기 작업이 수행될 때 주의해야할 상황들이 존재하는데, 그 중 대표적인 경우는 Data race, Race Condition이다.<br>

 A가 가지고 있는 것을 B, C, D가 필요로 하는 상황을 가정해보자. A의 자원을 변경하지 않고 조회/확인 용도라면 문제가 발생하지 않는다. 하지만 A의 자원을 변경시켜야 한다면 문제 상황이 발생할 수 있다. <b>B, C, D에서 '동시에' A에 접근</b>하여 자원을 조작할 수 있기 때문이다. 예를 들어 <b>A: 5만원이 들어있는 금고, B,C,D: 금고에 접근할 수 있는 사람들</b>이라고 하자. A의 금고액은 항상 1명만 접근이 가능해야한다. B와 C가 동시에 5만원을 가져가려고 할 때, <b>1명만 접근할 수 있는 제한상태가 보장</b>되지 않으면 5만원이 있는데 10만원(A, B 각각 5만원)이 빠져나가는 오류가 발생할 수 있다.



## Actor

GCD의 DispatchQueue를 활용할 때는 Serial Dispatch Queue나 Dispatch Semaphore를 사용하여 race condition을 해결한다. 공유자원에 접근할 때 lock을 걸어주는 용도이다. Swift 5.5 부터 새롭게 나온 Swift Concurrency에선 Actor 라는 타입을 활용한다.<br>



- Actor는 reference type이다.
  - Actor는 여러 곳에서 필요로하는 자원을 담고 있는 타입이다. 따라서 class와 같은 reference type이어야 자원의 일관성을 유지할 수 있다.
- class type과 차이점은 오직 하나의 task (비동기 작업 단위) 만이 actor의 자원 상태를 변경할 수 있다.
  - Race condition을 막을 수 있다.



 기존의 data race 대응 방식보다 이점은 type 자체가 thread-safe 하다는 점이다. 기존 방식에서는 class의 shared property에 접근할 때를 계속 주의하여 코드를 작성해야한다. Actor type은 자체가 shared property의 접근을 조절하고 있다.<br> 또한 기존방식은 compile time에 property의 data race condition 가능성을 파악할 수 없지만, Actor는 compile time checking이 가능하다.



## Actor 사용 방식

### 문제상황

공연 티켓팅 상황을 가정해보자. 총 공연 티켓은 100개, 티켓팅 대기자 수는 1000명. 먼저 Race Condition을 고려하지 않아 (Actor 사용 X 인 경우) 문제가 발생하는 상황이다. 

```swift
class Enrollment {
  private(set) var availableSeats = 100
  private(set) var success = 0
  private(set) var fail = 0
  
  func enroll() {
    if availableSeats <= 0 {
      fail += 1
    } else {
      success += 1
      availableSeats -= 1
    }
  }
}

let totalApplicants = 1000
let enrollment = Enrollment()
let group = DispatchGroup()

for _ in 0..<totalApplicants {
  DispatchQueue.global().async(group: group) {
    enrollment.enroll()
  }
}

group.notify(queue: .main) {
  print("남은좌석: \(enrollment.availableSeats)")
  print("성공자 수: \(enrollment.success)")
  print("실패자 수: \(enrollment.fail)")
  print("성공 + 실패 총 수: \(enrollment.success + enrollment.fail)")
}
```

DispatchQueue.global().async 문을 통해 enrollment에 비동기 접근을 시도하고 있다. 만약 정상적으로 티켓팅이 진행되었더라면, 마지막 프린트 4줄에서 <b>남은좌석 0, 성공자 수 100, 실패자 수 900, 성공+실패 총 수 1000</b> 으로 항상 일정해야한다.<br>

실제 결과

- 1회
  - 남은좌석: 0 / 성공자 수: 99 / 실패자 수: 869 /  성공 + 실패  총 수: 968
- 2회
  - 남은좌석: 0 / 성공자 수: 102 / 실패자 수: 851 /  성공 + 실패  총 수: 953
- 3회
  - 남은좌석: 0 / 성공자 수: 94 / 실패자 수: 858 /  성공 + 실패  총 수: 952

와 같이 매번 다른 결과가 나타난다. 티켓팅은 1000번 시도되기에, 성공 + 실패가 항상 1000이어야하는데 항상 그 값에 미치지 못한다. 티켓팅 가능 좌석 수(availableSeats)에 동시에 접근해버리는 Race Condition이 고려되지 않았기 때문이다.



### Actor로 해결하기

이번 글의 메인 주제인 Actor로 위 상황을 해결해보자.

```swift
actor Enrollment {
  private(set) var availableSeats = 100
  private(set) var success = 0
  private(set) var fail = 0
  
  func addCount() {
    if availableSeats <= 0 {
      fail += 1
    } else {
      success += 1
      availableSeats -= 1
    }
  }
}
```

actor 키워드로 타입선언을 한다.

Enrollment의 프로퍼티인 availableSeats, success, fail 은 별도의 lock 과정이 필요없이 일관성이 보장된다.<br>

여기서 추가적으로 private(set) 키워드 에 주목해보면, 해당 키워드는 외부에서 값을 변경하는 것은 막고 값을 읽는 것만 가능하다는 것을 의미한다. 따라서 외부에서

```swift
enrollment.availableSeats = 80
```

와 같이 변경을 시도하면<br>

<b>Cannot assign to property: 'availableSeats' setter is inaccessible</b><br>

와 같은 error 문구가 뜬다.<br>



Actor는 private(set)을 선언해줘도 상관없지만, 별도로 사용하지 않아도 된다. 이미 기본적으로 actor의 프로퍼티는 외부읽기가능, 외부변경불가이기 때문이다.<br>



따라서 위와같이 작성해도 되지만 

```swift
var availableSeats = 100
var success = 0
var fail = 0
```

로 변경해주어도 

```swift
enrollment.availableSeats = 80
```

처럼 값 변경을 시도하면<br>

<b>Actor-isolated property 'availableSeats' can not be mutated from a non-isolated context</b> 에러문이 발생한다.<br>

Actor의 availableSeats (isolated property, 고립된 속성)은 외부(non-isolated context, 고립되지않은 문맥)에서 변경될 수 없다. 즉, isolated property는 isolated context인 actor내부에서만 변경이 가능하다는 뜻이다.<br>

이 기본 성질이 Actor 특징의 핵심부분이 된다.<br>



```swift
let totalApplicants = 1000
let enrollment = Enrollment()

Task {
      await withTaskGroup(of: Void.self, body: { taskGroup in
  
          for _ in 0..<totalApplicants {
              taskGroup.addTask {
                  await enrollment.addCount()
              }
          }
      })
  
  print("남은좌석: \(await enrollment.availableSeats)")
  print("성공자 수: \(await enrollment.success)")
  print("실패자 수: \(await enrollment.fail)")
  print("성공 + 실패 총 수: \(await enrollment.success + enrollment.fail)")
}

```

Task 와 TaskGroup을 통해 1000번의 티켓팅 시도를 비동기적으로 시행해보는 부분이다. <br>

이전 문제 상황과 달리 하단의 print 4줄은 항상 같은 값 남은좌석 0 / 성공자  100 / 실패자 900 / 성공+실패 총 1000 이다<br>

race condition 이 발생하지 않는다. <br>



```swift
taskGroup.addTask {
                  await enrollment.addCount()
}
```

부분을 보면 addCount는 async라고 명시하지 않았지만 메소드 호출 시 await 작성해야한다. 동시에 접근이 가능하면 어느 하나가 기다리는 상황이 발생하기 때문이다. Actor type을 활용하는 경우 이 enrollment.addCount() 라는 expression 자체를 async로 본다.<br>



print문을 봐도 마찬가지이다. Enrollment의 isolated-property 값 읽기를 시도할 때 await를 작성해야한다. 이유는 위와 동일하다.



## 정리

새로 생긴 타입 Actor는 비동기처리 시 발생하는 대표적인 문제 상황인 Race-Condition을 해결하기 위해 고안되었다. 기존의 lock, serial queue, semaphore 등의 해결방식에 비해 compile-time에 버그확인이 가능하다는 점, 타입 내에서 별도 처리과정이 불필요하다는 점에서 장점을 가지고 있다.<br>



이전보다 async / await에 대해 조금 더 가까워지고 있는 기분이다. 코드의 가독성을 많이 고려해서 설계하고 develop 중이라고 느껴진다.<br>

#### 참고

<https://developer.apple.com/videos/play/wwdc2021/10133>

<https://medium.com/better-programming/preventing-data-races-using-actors-in-swift-ed3d8a69adf3>


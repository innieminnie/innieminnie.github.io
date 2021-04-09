---
layout: post
title: "TDD와 XCTest 활용하기 "
date: 2020-12-14 23:53:00
categories: TDD XCTest UnitTest
---
TDD와 XCTest
 
1. Test Driven Development
   - TDD의 정의
   - TDD의 의도
   - TDD의 필요성
   - TDD에서 레드단계 (실패한 코드 작성하기) 를 먼저 시행하는 이유
2. XCode에서의 XCTest
   - 연습해보기
   - setUp & tearDown
   - 레드
   - 그린
   - 블루
3. 마무리
   - 더 고민해볼 점



### Test Driven Development

#### TDD의 정의

- 테스트주도개발 ( 짧은 주기의 반복을 통해 테스트가 개발을 이끌어 나가는 형태의 소프트웨어 개발 프로세스)

![tdd_process](https://user-images.githubusercontent.com/46582215/102102475-336cea80-3e6f-11eb-9257-fb3d7875c6d1.png)


1. 레드: 실패하는 테스트 작성
2. 그린: 실패하는 테스트를 통과하는 최소한의 코드 작성 단계 ( 완전한 코드 작성 단계가 아닙니다다)
3. 블루: ''그린''단계의 코드에서 더 나은 완전한 코드로 나아가기 위한 리팩터링 단계. 기존 코드가 변경되어서는 안됩니다. 코드의 정상적 동작을 보장해야합니다.



TDD 작성 시, FIRST원칙을 준수하도록 노력하면 보다 나은 TDD 구현이 가능합니다.

<b>F</b>ast - 테스트는 빠르게 유지가능해야한다.

<b>I</b>ndependent -  테스트는 독립적 실행 단위를 갖는다.

<b>R</b>epeatable - 테스트는 반복적으로 실행될 때마다 결과가 동일해야한다.

<b>S</b>elf-validating - 테스트는 기대하는 결과를 단언(assert)할 수 있어야한다.

<b>T</b>imely - 테스트는 미루지말고 그때 그때 작성해야한다.



#### TDD의 의도

- 결정을 하기 전, 관찰 먼저 합니다.
- 켄트백의 "Clean Code That Works" ( 동작하는 깔끔한 코드를 설계하자.)

#### TDD의 필요성

- 큰 단위의 문제를 작은 단위로 나눌 수 있습니다. 코드의 모듈화를 유도합니다. 
- 지속적인 피드백을 통해 목표를 개선하게 됩니다. ( 실패에 대한 인지를 갖고 있기 때문입니다.)
- 테스트는 훌륭한 스펙 정의 문서가 될 수 있습니다.
- 테스트를 하지 않으면 코드의 리팩토링이 어려워집니다.

#### TDD에서 레드단계 (실패한 코드 작성하기) 를 먼저 시행하는 이유

- 테스트를 고려하는 코드를 작성하게 됩니다.
- 테스트(스펙)에 대한 구체적인 이해를 돕습니다.
- 만들고자 하는 것에 대한 이해 / 명확하고 구체적인 목표 / 빠른 피드백이 가능합니다.



### XCode에서의 XCTest

#### 연습해보기

XCode에서 XCTest를 활용하여 단위테스트를 수행할 수 있습니다. XCTest는 UITest, UnitTest, PerformanceTest 세 종류의 테스트를 지원하지만, UnitTest 구현과정을 대표로 보여드리겠습니다.

[https://leetcode.com/problems/design-parking-system/](https://leetcode.com/problems/design-parking-system/) 의 문제를 샘플로, 풀이 과정을 XCTest를 활용해 TDD를 간단하게 적용해보겠습니다!

- 문제 요구사항 : 주차 시스템 (Parking System) 을 디자인해라. 주차장은 세가지 크기의 주차공간 big, medium, small을 가지며 각 크기 별로 공간이 할당되는 갯수를 갖는다.
- <b>class ParkingSystem 의 init(_ big: Int, _ medium: Int, _ small: Int)</b> : Parking System 인스턴스 생성 메소드. 파라미터는 각 사이즈 주차공간의 할당 갯수를 의미한다.
- <b>class ParkingSystem의 func addCar(_ carType: Int) -> Bool</b> : 현재 Parking system에 주차하려는 차의 carType의 주차공간이 있는지 체크하는 메소드. carType은 big = 1 , medium = 2, small = 3 으로 나타낸다. 주차하려는 차는 오직 해당 carType의 주차공간에만 주차가 가능하다. 가능한 공간이 없으면 return false, 있으면 파킹을 한 후( 남은 공간 하나 감소), return true 

```
example) 

Input
["ParkingSystem", "addCar", "addCar", "addCar", "addCar"]
[[1, 1, 0], [1], [2], [3], [1]]
Output
[null, true, true, false, false]

Explanation)

ParkingSystem parkingSystem = new ParkingSystem(1, 1, 0);
parkingSystem.addCar(1); // return true because there is 1 available slot for a big car
parkingSystem.addCar(2); // return true because there is 1 available slot for a medium car
parkingSystem.addCar(3); // return false because there is no available slot for a small car
parkingSystem.addCar(1); // return false because there is no available slot for a big car. It is already occupied.
```





프로젝트 생성 시, include Tests를 하면 "~Tests폴더" 아래 ~Tests.swift파일을 확인할 수 있습니다. 

Xcode 단위테스트들은 XCTestCase 하위클래스 내에 포함되어 있습니다.

```swift
// DesignParkingSystemTests.swift 

import XCTest
@testable import DesignParkingSystem

class DesignParkingSystemTests: XCTestCase {
    override func setUpWithError() throws {
       // Put setup code here. This method is called before the invocation of each test method in the class.
      super.setUp()
    }
    override func tearDownWithError() throws {
      super.tearDown()
       // Put teardown code here. This method is called after the invocation of each test method in the class.
    }

    func testExample() throws {
    }

    func testPerformanceExample() throws {
        self.measure {
        }
    }

}
```



#### setUpWithError & tearDownWithError

<b>setUp</b>은 XCTestCase 각각의 test가 실행하기 이전에 호출되며 <b>tearDown</b>은 각각의 test 실행이 끝나면 호출됩니다.

(Xcode 11.4 이전에는 testFile생성시, setUp / tearDown이 기본값이었지만, 이후에는 해당 과정 중 error발생 시에 대한 처리가 추가된 setUpWithError / tearDownWithError로 대체되었습니다.setUpWithError / tearDownWithError 는 각각 에러처리 작업을 한 후 , setUp / tearDown를 이전과 동일하게 호출합니다.) 



setUp메소드에는 주로 테스트 케이스의 모든 테스트에 적용할 인스턴스를 생성합니다(shared properties). 문제를 활용하여 class DesignParkingSystem타입을 먼저 개략적으로 구상합니다.

```swift
//ParkingSystem.swift

import Foundation
class ParkingSystem {
  
		private var big: Int
    private var medium: Int
    private var small: Int

    init(_ big: Int, _ medium: Int, _ small: Int) {
        self.big = big
        self.medium = medium
        self.small = small
    }
    
    func addCar(_ carType: Int) -> Bool {
       return false //타입반환형을 맞춰주기 위한 일시적인 코드일뿐, 완성된 코드가 아닙니다.
    }
}

```

이후 기존 DesignParkingSystemTests의 setUp에 문제의 example에 나와있는 parkingSystem을 기준으로 initialize합니다. 아래의 코드에서 parkingSystemMock는 주차공간big 1개, 주차공간 medium 1개, 주차공간 small 0개를 갖습니다. 

```swift
class DesignParkingSystemTests: XCTestCase {
    override func setUpWithError() throws {
        let parkingSystemMock = ParkingSystem(1, 1, 0)
        super.setUp()
    }
```



#### 레드

class DesignParkingSystemTests의 func testExample() throws {} 에는 테스트코드를 작성합니다. 메소드명은 test로 시작해야하며, test이후 메소드명은 테스트하고자하는 내용에 맞게 수정합니다. 해당 과정에서는 testSuccessAddingCarInParkingSystem()으로 지었습니다. 또한 테스팅을 위해 XCTAssertEqual등을 비롯한 상황에 맞는 다양한 메소드를 활용합니다. 문제에서는 addCar 메소드에 대한 테스트를 할 것입니다.

![failed_test_case](https://user-images.githubusercontent.com/46582215/102102299-f9034d80-3e6e-11eb-95b7-75fa224d5e6f.png)

테스트는 addCar(1) 시, 성공해야하지만(expected = true) , false를 return 하고 있습니다(actual). 이 부분이 앞서 말씀드린 TDD의 "레드" 에 해당합니다.  addCar 메소드에는 구체적인 함수내용을 작성하지 않은 채, 우선 return false를 하고 있습니다. 해당 부분이 실패코드를 작성한 것과 같습니다. 이제 addCar를 문제에 맞게 작성해보겠습니다.



#### 그린

```swift
func addCar(_ carType: Int) -> Bool {
        switch carType {
        case 1:
            if self.big - 1 >= 0 {
                self.big -= 1
                return true
            } else {
                return false
            }
        case 2:
            if self.medium - 1 >= 0 {
                self.medium -= 1
                return true
            } else {
                return false
            }
        case 3:
            if self.small - 1 >= 0 {
                self.small -= 1
                return true
            } else {
                return false
            }
        default:
            return false
        }
    }
```

코드 수정 후, test를 재실행하면

![success_test_case](https://user-images.githubusercontent.com/46582215/102102656-6ca55a80-3e6f-11eb-8287-100cd4c1d6cc.png)

와 같이 메소드명 옆에 초록색 체크표시와 함께 성공했다는 메시지가 콘솔창에 출력됩니다

해당 부분이 TDD의 "그린" 에 해당합니다.



#### 블루

TDD의 "블루" 단계로,  addCar메소드를 메소드의 실행내용은 변형시키지 않은 채 수정해보겠습니다.

![refactoring_code](https://user-images.githubusercontent.com/46582215/102102545-48e21480-3e6f-11eb-80b9-9382a90d8949.png)

이후 테스트케이스 여러개 재실행 시, 전부 정상적으로 작동하는 것을 확인할 수 있습니다.



### 마무리

이로써 TDD의 레드 > 그린 > 블루 의 과정을 XCTest를 활용한 유닛테스트로 간단하게 나타내보았습니다. 위에서는 보편적으로 사용되는 XCTAssertEqual메소드의 사용을 보여주었지만, 해당 과정은 true, false를 return하는 것을 확인해야하는 것으로 보아  XCTAssertEqual보다는 <b>XCTAssertTrue(actual, "it should be true")</b> 메소드를 사용하는 것이 더 적합해 보입니다. 이외에도 다양한 상황에 적용해볼 수 있는 XCTAssert 관련 메소드가 많으니 맞게 활용하면 좋을 것 같습니다!



#### 더 고민해볼 점

위의 과정은 test 메소드를 여러 개 생성하면 전부 단일적이라 각 test가 시작되는 시점에 새로운 인스턴스가 할당됩니다. 그래서 test의 내용이 전부 독립적일 때는 적용 가능하지만, 예를 들어 addCar 메소드가 처음에는 파라미터로 받는 carType에 대한 공간의 여유가 있어 true를 반환하더라도 지속적으로 메소드를 실행하면 공간이 부족해 false가 발생하는 경우를 test해보고 싶을 때, 단순히 하나의 test안에 XCTAssert~문을 여러 번 실행해야하는지, 아니면 test를 분할해도 서로의 상황을 연결지을 수 있는지  더 학습할 예정입니다.

#### 참고
[https://nshipster.com/xctestcase/](https://nshipster.com/xctestcase/)
[https://zeddios.tistory.com/991](https://zeddios.tistory.com/991) (리나가 공유해줘서 도움이 됐어요)
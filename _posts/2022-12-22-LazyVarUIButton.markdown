---
layout: post
title: "UIButton을 lazy var로 선언해야 하는 이유"
date: 2022-12-21
categories: UIButton Initialization LazyVar
---
<br>

[일반적인 UI Component 정의](#일반적인-ui-component-정의)<br>
[private let 으로 UIButton을 선언하면](#private-let-으로-uibutton을-선언하면)<br>
[Lazy var button여야 하는 이유](#lazy-var-button여야-하는-이유)<br>
[Swift Initialization](#swift-initialization)<br>
[Summary](#summary)<br>

---
# 일반적인 UI Component 정의
코드로 화면에 배치할 UITextField, UILabel, UIButton 등과 같은 UIComponents 작성 시, 아래와 같이 정의한다.

```swift
 private lazy var addButton: UIButton = {
    let button = UIButton()
    button.translatesAutoresizingMaskIntoConstraints = false
    button.configureAbleMode(title: "추가")
    button.applyCornerRadius(24)
    button.addTarget(self, action: #selector(tappedAddButton(sender:)), for: .touchUpInside)
    
    return button
  }()
```

여기서 <b>lazy var</b>를 굳이 사용하는 이유가 뭘까 싶었다. UIKit에 정의된 UI Component Class를 상속받아 정의하기에, let으로 선언해도 일반적인 property 설정은 대게 다 가능하기 때문이다.

---
# Private let 으로 UIButton을 선언하면

그래서 UI Component를 선언할 때 일부러 <b>private let</b>으로 전부 해봤다. 그런데 UIButton에서만 경고문이 발생했다.

<center><img src="/assets/img/2022-12-22-uibutton1.png" width = 500></center>

addTarget의 self를 HomeViewController.self (해당 uibutton이 로 변경해서 warning을 없애랜다. 그래서 바로 Fix 누름.
<br>
바꾸래서 바꿨더니 해당 button을 누르면 crash가 발생한다.
<center><img src="/assets/img/2022-12-22-uibutton2.png" width = 500></center>

\#selector에 작성한 메소드가 static method가 아닌 HomeViewController의 instance method이기 때문이다.  

```swift
@objc private static func tappedAddButton(sender: UIButton) { 
    button 누르면 실행할 코드 작성
}
```

이렇게 고치면 실행은 되지만 static func이기에 제한적이다.
static func 안에서는 instance member를 사용할 수 없기 때문. static func에선 static member/class member만 있어야한다. 하지만 내가 작성한 tappedAddButton는 버튼을 눌러서 다른 ViewController의 인스턴스 생성 및 present 하는 작업 (instance member가 존재한다.)이 들어가야 하기에 static func면 안된다.

해당 선언 부분을 <b> private lazy var addButton</b> 으로 바꾸면 warning 없고, selector 내의 action 관련 메소드(tappedAddButton(sender: UIButton))도 잘 찾는다.

---
# Lazy var button여야 하는 이유
lazy var와 let의 차이는 <b>초기화 시점</b>이다. lazy var는 초기화를 실제 해당 변수가 사용되는 시점으로 지연시킨다. 일반적으로 button에 대한 action은 Target-Action 패턴을 통해 정의한다. 여기서 action은 파일 내에서 정의된 @objc 메소드일 확률이 높다.

<b>warning이 발생한 이유:</b> let일 경우, addTarget의 self가 정의 완료되지 않았다. 모든 저장 프로퍼티가 준비되어야 self에 접근할 수 있다.

lazy var를 사용하면, button이 사용되기 직전에 정의 코드가 실행되어 영역을 할당받기에, self가 HomeViewController의 인스턴스임을 파악하고 self에 접근할 수 있다.

다른 UI Component들은 let이어도 무관하던 이유는 자기 자신에 대한 속성 설정만 있었기에 ViewController가 초기화되는 점이 필요치 않았다. 이와 달리 button들은 보통 해당 button에 대한 action이 들어가있기 마련이다. Button을 그냥 디스플레이용으로만 쓰는 경우는 거의 없기 때문이다. 그래서 UIButton처럼 UIControl을 상속하는 class의 경우는 좀 더 유의할 필요가 있다.

---
# Swift Initialization
참고한 스택오버플로우에서 swift의 initialization phase1 과 phase2를 설명해주고 있기에 initailization 페이지에 가서 확인해보았다.

## Two Phase Initialization
Class 초기화는 2번의 과정에 걸쳐 진행된다.

<b>Phase 1</b> - 각 저장 프로퍼티는 class가 설정한 초깃값이 할당된다.
<b>Phase 2</b> - Phase 1을 거친 저장 프로퍼티들을 커스터마이징 할 수 있다 (새 인스턴스가 프로퍼티를 사용하기 전에).

이 과정은 initialization(초기화)의 안정성을 높인다. Property 값들이 초기화 되기 이전에 접근되는 것을 막아준다. 또한 다른 initializer에 의해 다른 값이 setting 되는 것으로부터 예방한다.

## Phase 1 / Phase 2 구체적인 과정
### Phase 1
- designated / convenience initializer 가 class에 호출된다.
- 해당 class의 새 instance를 위한 메모리가 할당된다. 메모리는 아직 초기화되지 않았다. 영역만 할당된 상태.
- designated initializer 가 class의 모든 저장프로퍼티들이 값이 있는지 체크한다. 그러면 저장프로퍼티들을 위한 메모리가 초기화된다.
- designated initializer는 저장 프로퍼티들에 대해 같은 일(메모리 확보)을 수행하도록 superclass initializer에게 위임한다.
- 이 과정 (class inheritance chain)은  chain의 끝부분에 다달을 때까지 계속된다.
- final class까지 과정을 다하면, 모든 저장프로퍼티들은 값을 지니고 있음이 보장된다. 인스턴스의 메모리는 완전히 초기화 완료된다.

### Phase 2
- Phase 1 진행방향의 역방향으로 다시 chain을 타고 내려온다. 내려오면서 각각의 designated initializer 는 인스턴스를 customizing할 옵션사항을 지니고 있다. Initializer는 이제 self에 접근 가능하고, 프로퍼티를 수정하거나 / instance method를 호출하거나 할 수 있다. 
- chain에 있는 convenience initializer 또한 인스턴스를 customizing 하거나 self에 접근하여 원하는 작업을 수행할 수 있다.


> Phase 1은 일단 저장프로퍼티들의 값을 확인하면서 메모리 영역을 확보하는 작업 수행<br><br>Phase 2는 조상 class부터 내려오면서 designated / convenience initializer 작업 진행하면서 customizing 가능.

---
## Summary
프로퍼티의 기본값을 정의할 때, 모든 저장 프로퍼티들의 초기화가 완전히 이루어졌는지가 중요하다. 생각해보면 글에서 다룬 UIButton의 addTarget부분의 self를 파악하지 못하는 문제점과 비슷하게 프로퍼티 초깃값 설정 시 인스턴스의 다른 저장프로퍼티를 호출할 수 없다.

```swift
private let buttonA: UIButton = {
    let button = UIButton()
    self.buttonB.functionB() // 오류 발생

    return button
}
``` 

이것도 private lazy var 로 변경하면 컴파일은 된다. 근데 런타임 시 buttonB가 초기화 완료되었음을 보장하기 어려울 듯.

---
### 참고
- [https://stackoverflow.com/questions/71560311/xcode-13-3-warning-self-refers-to-the-method-object-self-which-may-be-u](https://stackoverflow.com/questions/71560311/xcode-13-3-warning-self-refers-to-the-method-object-self-which-may-be-u)

- [https://docs.swift.org/swift-book/LanguageGuide/Initialization.html](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html)

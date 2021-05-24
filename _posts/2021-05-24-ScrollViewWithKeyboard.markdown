---
layout: post
title: "키보드 가림 현상 해결하기"
date: 2021-05-24
categories: Keyboard UIResponder UITextFieldDelegate
---
---

## 목차
1. [완성화면](#완성화면)
1. [오토레이아웃 설정](#오토레이아웃-설정)
1. [NotificationCenter에 키보드 동작 등록하기](#notificationcenter에-키보드-동작-등록하기)
1. [UITextFieldDelegate 관련 부분](#uitextfielddelegate-관련-부분)
1. [NotificationCenter의 post 이후 실행 메소드 작성](#notificationcenter의-post-이후-실행-메소드-작성)
    - Notification.userInfo 이란?
    - UIEdgeInsets 이란?
    - contentInset & verticalScrollIndicatorInsets
1. [ScrollView에서의 touchesBegan vs UITapGestureRecognizer](#scrollview에서의-touchesbegan-vs-uitapgesturerecognizer)
<br>

키보드의 입력창이 화면하단에 있을 때, 입력 시 키보드 위치와 겹쳐서 입력창이 가려지는 문제 현상을 개선해보겠습니다.

---
### 완성화면
![2021-05-24-scrollviewkeyboard01](/assets/img/2021-05-24-scrollviewkeyboard01.gif)

보시는 바와 같이 아래에 있는 TextField를 입력을 위해 tap했을 때, 키보드가 올라가고 TextField의 위치가 올라오는 것을 확인할 수 있습니다.

---
### 오토레이아웃 설정
![2021-05-24-scrollviewkeyboard02.png](/assets/img/2021-05-24-scrollviewkeyboard02.png)

우선 ScrollView위에 ContentView,<br>
ContentView 내부에 textfield들을 여러개 묶은 Vertical StackView를 배치했고, <br>
StackView와 ContentView, ContentView와 ScrollView의 Content Layout Guide, ContentView와 ScrollView의 Frame Layout Guide 사이의 제약조건을 설정했습니다.

---
### NotificationCenter에 키보드 동작 등록하기

```swift
class ViewController: UIViewController {
    @IBOutlet weak var scrollView: UIScrollView!
    @IBOutlet var textFields: [UITextField]!
    
    weak var activeField: UITextField?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        for textField in textFields {
            textField.delegate = self
        }
        
        NotificationCenter.default.addObserver(self, selector: #selector(keyboardAppear), name: UIResponder.keyboardDidShowNotification, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(keyboardDisappear), name: UIResponder.keyboardWillHideNotification, object: nil)
    }
    
    @objc func viewTapped() {
        if let activatedField = activeField {
            activatedField.resignFirstResponder()
        }
    }

    @objc func keyboardAppear(notification: Notification) {
       // UIResponder.keyboardDidShowNotification 을 NotificationCenter에서 감지했을 때 수행할 메소드 내용 (키보드가 나타났다.)
    }
    
    @objc func keyboardDisappear(notification: Notification) {
       // UUIResponder.keyboardWillHideNotification 을 NotificationCenter에서 감지했을 때 수행할 메소드 내용 (키보드가 사라질 예정이다.)
    }
}
```

1. UITextField가 여러개이므로 <b>activeField</b> 변수를 통해 <b>현재 tap한 UITextField</b>를 지정할 수 있도록 합니다.

1. <b>UIResponder.keyboardDidShowNotification</b> 와 <b>UIResponder.keyboardWillHideNotification</b> 를 NotificationCenter.default에 addObserver로 등록하여 반응을 감지했을 때 각각 <b>keyboardAppear</b> 와 <b>keyboardDisappear</b> 메소드를 수행할 수 있도록 합니다. 
![2021-05-24-scrollviewkeyboard03](/assets/img/2021-05-24-scrollviewkeyboard03.png)<br>
![2021-05-24-scrollviewkeyboard04](/assets/img/2021-05-24-scrollviewkeyboard04.png)

---
### UITextFieldDelegate 관련 부분
이제 <b>activeField가 설정</b>되는 부분과 <b>키보드의 return키를 눌렀을 때 textField가 입력을 종료</b>하는 부분을 UITextFieldDelegate 프로토콜을 채택하여 관련 메소드를 작성해보겠습니다.

```swift
extension ViewController: UITextFieldDelegate {
    func textFieldDidEndEditing(_ textField: UITextField) {
        activeField = nil
    }
    
    func textFieldDidBeginEditing(_ textField: UITextField) {
        activeField = textField
    }
    
    func textFieldShouldReturn(_ textField: UITextField) -> Bool {
        textField.resignFirstResponder()
        return true
    }
}
```
---
### NotificationCenter의 post 이후 실행 메소드 작성

```swift
extension ViewController {
    @objc func keyboardAppear(notification: Notification) {
        guard let activeField = activeField,
              let userInfo = notification.userInfo,
              let keyboardFrame = userInfo[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect else {
            return
        }
        
        let contentInsets = UIEdgeInsets(top: 0.0, left: 0.0, bottom: keyboardFrame.height + 10, right: 0.0)
        scrollView.contentInset = contentInsets
        scrollView.verticalScrollIndicatorInsets = contentInsets
        let activeRect = activeField.convert(activeField.bounds, to: scrollView)
        scrollView.scrollRectToVisible(activeRect, animated: true)
    }
```
1. activeField 와 keyboardFrame을 설정합니다.

    #### Notification.userInfo 이란?
    - NotificationCenter에 Notification이 등록될 때, Notification과 관련된 정보를 담고 있으며, <b>[AnyHashable : Any]? { get }</b> 구조의 딕셔너리입니다.

    - <b>userInfo[UIResponder.keyboardFrameEndUserInfoKey]</b> 는 화면 속 키보드 좌표에서 (키보드가 올라올 상황) ending frame rectangle (테두리) 을 CGRect 타입으로 지니고 있습니다.
    <br>
1. scrollView.contentInset & scrollIndicatorInsets를 설정하고 UITextField.convert() 와 UIScrollView.scrollRectToVisible() 을 활용하여 위치를 조정합니다.
    #### UIEdgeInsets이란?
    - rectangle에 파라미터(top, left, bottom, right)으로 설정한 거리만큼 사이즈 조정을 가능하게 합니다. UIEdgeInsets.zero는 top, left, bottom, right 모두 0으로 설정한 것과 같습니다.
    #### contentInset & verticalScrollIndicatorInsets
    ![2021-05-24-scrollviewkeyboard05](/assets/img/2021-05-24-scrollviewkeyboard05.png)
    - 사용자가 설정한 거리만큼(UIEdgeInsets) scroll view의 가장자리로부터 scroll view내의 content view와의 inset(간격)을 설정할 수 있습니다. (해당 과정에선 UIScrollView.contentInset을 활용하지만, safeArea 또한 contentInset이 있습니다.)
        ![2021-05-24-scrollviewkeyboard10](/assets/img/2021-05-24-scrollviewkeyboard10.png)
        scrollview의 contentInset의 bottom값을 조정해보겠습니다.<br>ThirdTextField를 tap했을 때입니다.<br>
          
        |UIEdgeInsets.zero|UIEdgeInsets(top: 0.0, left: 0.0, bottom: keyboardFrame.height, right: 0.0)|UIEdgeInsets(top: 0.0, left: 0.0, bottom: keyboardFrame.height + 10, right: 0.0)|
        |-|-|-|
        |![2021-05-24-scrollviewkeyboard07](/assets/img/2021-05-24-scrollviewkeyboard07.png)|![2021-05-24-scrollviewkeyboard08](/assets/img/2021-05-24-scrollviewkeyboard08.png)|![2021-05-24-scrollviewkeyboard09](/assets/img/2021-05-24-scrollviewkeyboard09.png)|
          
        <br>
    ![2021-05-24-scrollviewkeyboard06](/assets/img/2021-05-24-scrollviewkeyboard06.png)
    기존의 scrollIndicatorInsets가 deprecated 되었고 verticalScrollIndicatorInsets와 horizontalScrollIndicatorInsets로 분리되었습니다!<br>
    scrollview의 contentInset이 변경됨에 따라 scrollIndicatorInsets 또한 영향을 받는데요, contentInset의 변화로 추가되는 부분에도 scrollview의 indicator도 표시됩니다. scroll이 안되는 부분에도 indicator를 표시하는 것은 맞지 않으니 contentInset에 따라 scrollIndicatorInsets 또한 맞춰줍니다.<br>
    다음으로 
    ```swift
    let activeRect = activeField.convert(activeField.bounds, to: scrollView)
        scrollView.scrollRectToVisible(activeRect, animated: true)
    ```
    부분입니다.
    <b>UIView.convert(_:to:)</b> 를 통해 좌표변환을 활용합니다. ㅎㅎㅎ 여기서부터 머리가 살짝 아팠는데요! 이 function은 CGRect로 사각형에 대한 좌표계를 받아서 다른 view의 좌표계로 전환합니다!UIView의 bounds는 각 view마다 좌표가 다르기 때문입니다.<br>
    activeField.bounds 를 통해 TextField 사각형에 대한 좌표계 값을 scrollView의 좌표계를 기준으로 전환합니다.<br>
    제 식으로 다시 생각해 봤을 때, <b>서로 다른 view 간의 거리/위치 관계의 sync를 맞춰주는 느낌</b>이었습니다. <b>activeRect</b>를 통해 activeField의 좌표계를 <b>scrollView에 맞춰 재설정</b>하고, 이후에 해당 activeRect의 위치에 따라 <b>scrollView의 scrollRectToVisible</b>을 실행 가능하게 하기 위해서죠!<br>
    UIEdgeInset부터 조금 몽롱해졌는데 나중에 다시 한 번 bounds, frame, inset 등 화면 관련해서 포스팅을 다시 해봐야겠어요!<br>

1. keyboard가 이제 내려갈 때엔 contentInsets를 처음으로 되돌립니다. 
```swift    
        @objc func keyboardDisappear(notification: Notification) {
            let contentInsets = UIEdgeInsets.zero
            scrollView.contentInset = contentInsets
            scrollView.verticalScrollIndicatorInsets = contentInsets
        }
    }
```

---
### ScrollView에서의 touchesBegan vs UITapGestureRecognizer
지금까지 키보드 가림 현상에 대한 구현과정과 과정 속에서 학습한 내용을 정리해봤는데요, 구현 중 touchesBegan 과 UITapGestureRecognizer의 차이점에 대해서도 알게되어 추가해보겠습니다.

위의 내용은 키보드의 return 키를 눌렀을 때 키보드가 내려가고 scrollView도 원래의 위치로 돌아가는데 return키를 눌렀을 때뿐만 아니라 <b>키보드 이외의 화면을 tap했을 때</b>에도 올라와있는 키보드가 내려가도록 하고 싶었습니다.

단순하게 touchesBegan을 먼저 활용해보았습니다.
```swift
override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
    if let activatedField = activeField {
            activatedField.resignFirstResponder()
    }
}
```

이 부분을 작성한 이유! 작동이 안되더군요!
스택오버플로우를 서치해 본 결과, ScrollView에선 UIGestureRecognizer를 활용하는 사례가 많았습니다.

```swift
override func viewDidLoad() {
    //... 이전 작성 부분 ...

    let tapGestureRecognizer = UITapGestureRecognizer(target: self, action: #selector(viewTapped))
    tapGestureRecognizer.numberOfTapsRequired = 1
    tapGestureRecognizer.isEnabled = true
    scrollView.addGestureRecognizer(tapGestureRecognizer)
}
```
```swift
@objc func viewTapped() {
    if let activatedField = activeField {
        activatedField.resignFirstResponder()
    }
}
```
로 수정해주니 잘 동작하였습니다.

이유는~!
ScrollView는 유저가 scroll 하는 것을 목적으로 하는 UIView입니다. 따라서 유저의 터치가 '스크롤'을 위한 것인지 '단순히 잠깐 터치'하는 것인지 구분을 해야하는데 <b>touchesBegan</b> 은 인지하지 못합니다.

그래서 ScrollView를 서브클래싱 해주어서
>
    1. touchesShouldBegin:withEvent:inContentView:
    2. pagingEnabled
    3. touchesShouldCancelInContentView:

고 같은 function을 override 하여 해결할 수 있습니다.

하지만 해당 구현물에서는 ScrollView에 대해 서브클래싱을 해주지 않았기에 <b>UITapGestureRecognizer</b> 로 탭한 경우를 구분해주었습니다 :)

---
#### 참고
https://spin.atomicobject.com/2020/03/23/uiscrollview-content-layout-guides/<br>
https://stackoverflow.com/questions/28873530/uiscrollview-and-touchesbegan-touchesended-etc/38589220<br>
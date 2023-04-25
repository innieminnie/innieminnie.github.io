---
layout: post
title: "[CustomUI] TagBubbleView"
date: 2023-04-24
categories: CAShapeLayer UIBezierPath UIAnimation animateKeyframes
---
---

## Pop해서 나타나는 / 드래그앤드롭으로 위치변경가능한 말풍선

|Tap 했을 때 pop| Drag & Drop으로 위치 이동|
|-|-|-|
|<img src="/assets/img/2023-04-24-screen1.gif" width = 300>|<img src="/assets/img/2023-04-24-screen2.gif" width = 300>|


## KeyPoint
1. CAShapeLayer & UIBezierPath
1. PopAnimation
    - animateKeyframes
    - CGAffineTransform
1. UILongPressGestureRecognizer

### 1. CAShapeLayer & UIBezierPath
- CALayer<br>
  CALayer는 이미지 기반의 콘텐츠를 관리하고 해당 콘텐츠에서 애니메이션을 수행가능하게 하는 객체이다. View에서 여러 시각적 효과를 제공할 때 layer를 활용한다. UIView는 view 내에 layer를 가지고 있어 view.layer를 통해 시각효과를 구현할 수도 있고, CALayer 자체가 독립적이기에 CALayer 객체를 독립적으로 선언하여 활용할 수도 있다.<br><br>

- CAShapeLayer<br>
  CALayer는 효과를 나타내고자 하는 구체적인 대상에 따라 CAShapeLayer, CATextLayer, CAGradientLayer로 나눌 수 있다. 이번엔 CAShapeLayer를 다룬다.<br>

  CAShapeLayer는 도형을 표현할 때 주로 사용한다. fillColor / stokeColor / lineJoin 등과 같이 layer에 그려지는 도형의 스타일을 다루는 properties를 가지고 있다.<br><br>

- UIBezierPath<br>
  CAShapeLayer에 도형을 표현하려면 path를 그려야한다. path의 선분을 그리기 위해 UIBezierPath를 활용한다. UIBezierPath로 원하는 도형의 path를 그려준 후, CAShapeLayer의 path(CGPath)에 UIBezierPath의 cgPath를 대입한다.<br><br>

> UIBezierPath 와 CGMutablePath ? <br>
  여러 예시코드를 살펴보다 UIBezierPath를 사용하는 곳도 있고 CGMutablePath를 사용하는 곳도 있다. layer의 path는 CGPath이기에 CGMutablePath를 직접 할당하는 것이 더 적합한가 생각이 들었다.<br><br>
  https://stackoverflow.com/questions/25457597/what-is-the-difference-between-cgpath-and-uibezierpath 에 따르면,<br>
  CGPath는 CoreGraphics 라이브러리 소속이고 UIBezierPath는 UIKit 라이브러리 소속이다. UIBezierPath는 CG의 wrapper인 objc 객체이다.  그러므로 CGPath가 iOS 아키텍처 구조 상 보다 하위층에 속한다. 그래서 CoreGraphics에서 핸들링 가능하고 정교한 작업 수행이 가능하다. 비교적 상위층에 있는 UIBezierPath의 property 접근을 통해 기본적 선분그리기관련 작업이 가능하다.<br><br>

TagBubbleView는 UIView의 위/아래 에 삼각형 shape을 추가해서 말풍선의 꼬리 부분을 만들었다. 위쪽 삼각형 관련 코드이다.

```swift  
  private let upperShape = CAShapeLayer()
  private let upperPath = UIBezierPath()
  
  private func addUpperArrow() {
    upperPath.move(to: CGPoint(x: bounds.midX - 10, y: 1))
    upperPath.addLine(to: CGPoint(x: bounds.midX, y: -10))
    upperPath.addLine(to: CGPoint(x: bounds.midX + 10, y: 1))
    upperPath.close()
    
    upperShape.path = upperPath.cgPath
    upperShape.fillColor = UIColor.systemGray.cgColor
    self.layer.addSublayer(upperShape)
  }
```
UIBezierPath의 move로 도형그리기 시작하는 위치정해서 addLine으로 그린다. 마지막 close() 로 처음 시작 위치로 돌아온다.<br>
CAShapeLayer의 path에 UIBezierPath의 cgPath를 대입한다.<br>
CALayer의 속성을 통해 도형의 스타일을 정의한다. 여기선 도형의 색상을 회색으로 설정했다.<br><br>
TabBubbleView의 layer에 sublayer로 upperShape을 추가한다<br>

아래 모양 삼각형도 path가 그려지는 위치만 바꿔서 동일한 방식으로 만든다. 화면의 centerY 기준으로 했을 때 TagBubbleView가 상단에 위치하면 아래모양삼각형이 활성화되고 반대일 경우에는 윗모양삼각형이 활성화된다.(뒤에 드래그 설명할 때 구현)<br>

### 2. PopAnimation
인스타그램의 사진의 해시태그를 확인할 때처럼 이미지를 탭할 때 TagBubbleView가 bouncing하는 효과를 나타냈다. TagBubbleView가 커졌다 작아졌다 다시 살짝 커진다.<br>

- animateKeyframes<br>
    keyframe이 개인적으로 편한 이유는 UIView.animate보다 코드가 간결하게 느껴진다. UIView.animate는 연속적으로 실행하는 animation이 있을 때 (동시작용X, 애니메이션의 순서 존재) completionHandler를 활용해야하는데 순서가 많아지면 completionHandler로 덩달아 많아진다...<br><br>

    사진이미지가 탭될때마다 TagBubbleView의 isHidden이 토글되는데, isHidden이 false 될 때 애니메이션이 실행된다.
```swift
  override var isHidden: Bool {
    didSet {
      if !isHidden {
        UIView.animateKeyframes(withDuration: 0.5, delay: 0) {
          UIView.addKeyframe(withRelativeStartTime: 0, relativeDuration: 1 / 3) {
            self.transform = CGAffineTransformMakeScale(1.2, 1.2)
          }
          
          UIView.addKeyframe(withRelativeStartTime: 1 / 3, relativeDuration: 1 / 3) {
            self.transform = CGAffineTransformMakeScale(0.8, 0.8)
          }
          
          UIView.addKeyframe(withRelativeStartTime: 2 / 3, relativeDuration: 1 / 3) {
            self.transform = CGAffineTransformIdentity
          }
        }
      }
    }
}
```
- CGAffineTransform<br>
  view의 변화방식을 구체화한다. Scale(확장) / Rotate(회전) / Translate(이동) 이 가능하다. 여기선 커졌다 작아졌다니까 <b>CGAffineTransformMakeScale</b>을 사용했다. x, y 각각 커지는 대비만큼 작성한다. (1이 기본)<br>
  마지막 <b>CGAffineTransformIdentity</b>는 저장된 초기상태를 불러와 원상복귀시 사용한다.<br><br>
- animateKeyframes<br>
  클로저 내의 내용을 진행하는데 걸리는 전체시간(withDuration)과 애니메이션 시작 전 지연시간(delay) 설정<br><br>
- addKeyframe<br>
  애니메이션의 동작 순서에 따라 순차적으로 추가해준다. 주의 점은 파라미터다. addKeyframe은 animateKeyframes에서 설정한 duration에 대해 <b>상대적인 시작시간</b>을 기준으로 한다. withRelativeStartTime은 전체시간 대비 어느정도 지났을 때 시작하는지, relativeDuration은 이 시작시간을 기준으로 전체시간 대비 어느정도의 기간동안 진행해야하는지 적는다. <br><br>
  
### 3. UILongPressGestureRecognizer
TagBubbleView를 길게 눌러서 드래그 앤 드롭을 가능케한다.
UILongPressGestureRecognizer의 state에 따른 처리가 가능하다.

```swift
 let longPressGesture = UILongPressGestureRecognizer(target: self, action: #selector(moveBubbleLocation(_:)))
 self.addGestureRecognizer(longPressGesture)

  @objc private func moveBubbleLocation(_ recognizer: UILongPressGestureRecognizer) {
    switch recognizer.state {
    case .began, .changed:
      var center = self.center
      guard let superView = superview else { return }
      
      center.x = recognizer.location(in: superView).x
      center.y = recognizer.location(in: superView).y
      self.center = center
      
      if center.y < superView.frame.midY {
        upperShape.isHidden = true
        lowerShape.isHidden = false
      } else {
        upperShape.isHidden = false
        lowerShape.isHidden = true
      }
      
    default:
      break
    }
  } 
```

드래그 하고 있는 상태에선 state가 changed이다. superview의 좌표를 기준으로 tagBubbleView의 center를 조정한다.<br><br>
그리고 superview의 centerY보다 위/아래 인지에 따라 hidden시킬 삼각형 모양을 정한다.

[깃허브 전체코드](https://github.com/innieminnie/test2/blob/main/Test2/TagBubble/TagBubbleView.swift)
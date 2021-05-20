---
layout: post
title: "ScrollView 활용하기"
date: 2021-05-20
categories: ScrollView AutoLayout
---
---

## 목차
1. [기본 ScrollView 구현방식](#기본-scrollview-구현방식)
1. [Content Layout Guide 와 Frame Layout Guide](#content-layout-guide-와-frame-layout-guide)
1. [활용하기: 키보드 올라올 때 ScrollView의 위치 조정하기](#활용하기-키보드-올라올-때-scrollview의-위치-조정하기)

---
## 기본 ScrollView 구현방식
[Auto Layout Guide_Working with Scroll Views](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/WorkingwithScrollViews.html#//apple_ref/doc/uid/TP40010853-CH24-SW1) 에서 ScrollView 활용 절차에 대해서 소개하고 있는데요, 이에 대해 적어보겠습니다.

1. Scene위에 scroll view를 추가해라.
![2021-05-20-scrollview01](/assets/img/2021-05-20-scrollview01.png)
1. Scroll view의 "크기 와 위치" 를 정의하기 위해 제약조건을 추가해라.
    - ScrollView를 SuperView에 대해 0/0/0/0 으로 제약조건 설정
![2021-05-20-scrollview02](/assets/img/2021-05-20-scrollview02.png)
![2021-05-20-scrollview03](/assets/img/2021-05-20-scrollview03.png)
1. Scroll view위에 view를 추가해라. 해당 뷰에 "Content View" 라고 이름을 지어라
![2021-05-20-scrollview04](/assets/img/2021-05-20-scrollview04.png)
    >Content View 라고 이름 짓는 것은 View 객체를 네이밍으로 구분짓기 위함이며, Content View 이름 자체가 갖는 의미는 없습니다 :)
1. Content View 의 top/bottom/leading/trailing을 scroll view의 edge에 맞춰라. Content View는 이제 Scroll View의 content area(스크롤 되어 콘텐츠가 나타나는 영역)에 해당한다.
    - ContentView를 ScrollView(SuperView)에 대해 0/0/0/0 으로 제약조건 설정
![2021-05-20-scrollview05](/assets/img/2021-05-20-scrollview05.png) 
    >!REMEMBER! 
    Content view는 현재 고정된 크기를 갖지 않는다. 내부의 구성요소에 따라 늘어날 수 있다.  
1. ✅ (Optional) 수직스크롤만 가능하게하기 - content view’s width와 scroll view’s width를 같게 해라.
![2021-05-20-scrollview06](/assets/img/2021-05-20-scrollview06.png)  
1. (Optional) 수평스크롤만 가능하게하기 - content view’s height와 scroll view’s height를 같게 해라.<br><br>
1. ContentView안에 원하는 content(예를 들어 UILabel) 을 배치시켜라. 안의 구성요소들의 layout와 intrinsic content size에 따라 ContentView의 크기가 정해질것이다.
![2021-05-20-scrollview07](/assets/img/2021-05-20-scrollview07.png)
![2021-05-20-scrollview08](/assets/img/2021-05-20-scrollview08.png)  
![2021-05-17-01](/assets/img/2021-05-20-01.png)

스크롤 한번 봅시다!
![2021-05-17-02](/assets/img/2021-05-20-02.gif)

---
## Content Layout Guide 와 Frame Layout Guide

스크롤뷰가 어렵게 느껴지는 이유는 제약조건을 다 설정해준 것 같은데도 오토레이아웃의 "빨간줄", 제약조건에 대한 error (Unsatisfiable, Ambiguous) 가 발생하기 때문입니다.

이는 스크롤뷰는 Content Layout Guide와 Frame Layout Guide 를 갖고 있다는 특징을 살펴볼 필요가 있습니다. 

- [Content Layout Guide](https://developer.apple.com/documentation/uikit/uiscrollview/2865870-contentlayoutguide)
    scroll view의 변형되지 않는 content rectangle 기반으로 하는 layout guide입니다. scroll view 내부 content area와 연관된 Auto Layout 설정시 이용합니다. 

- [Frame Layout Guide](https://developer.apple.com/documentation/uikit/uiscrollview/2865772-framelayoutguide) 
    scroll view의 변형되지 않는 frame rectangle 기반으로 하는 layout guide입니다. scroll view의 frame rectangle와 연관된 Auto Layout 설정시 이용합니다. 

두 layout guide에 대해 모두 설정해주어야 제약조건관련 error문이 발생하지 않습니다.

위의 진행과정에서 <b>Content Layout Guide 와 Frame Layout Guide</b>을 직접적으로 사용하진 않았는데요. 두 Layout Guide는 XCode11 이후 생긴 것으로 이전에는 위에서 보여드린바와 같이 Layout Guide를 활용하지 않고 스크롤영역 / 스크롤방향 각각에 대한 제약조건을 줘야한다는 것을 미리 알고 있어야 했습니다. XCode11 이후엔 ScrollView 생성시 Content Layout Guide 와 Frame Layout Guide가 자동으로 표기되는 것을 Storyboard에서 확인할 수 있습니다. Content Layout Guide 와 Frame Layout Guide를 활용하는 방법을 밑에 보여드리겠습니다. 위와 같은 결과를 가져오지만 방식만 다른겁니다!

위에서 진행한 과정에서,
- 4번과정에서 ContentView를 ScrollView(SuperView)에 대해 0/0/0/0 으로 제약조건 설정 -> ContentView가 ScrollView의 <b>content layout guide</b>에 대해 top/bottom/leading/trailing 설정

- 5번과정에서 ContentView의 width와 scroll view’s width를 같게 설정 -> ContentView의 width를 scroll view의 <b>frame layout guide</b>에 대해 width 를 동일하게 설정

![2021-05-20-scrollview09](/assets/img/2021-05-20-scrollview09.png) 
(제약조건 다섯가지가 LayoutGuide 기반으로 바뀐것을 유의하세요!)

로 설정할 경우, 동일한 결과를 갖습니다.

#### ❓왜 Layout Guide를 두 가지로 분리해서 제공할까요❓

ScrollView는 content 내부에 담긴 내용이 기기의 한 화면에 담지 못할 때, <b>유저가 스크롤을 통해 모든 정보를 보는 것을 가능케 한다</b>는 것에 의미를 갖습니다.<br> 
이를 위해서 content 내부에 모든 내용이 담기도록 content layout guide / frame layout guide를 각각 따로 설정함에 따라, <b>content layout guide</b>로 content에 대한 제약조건에 집중하고(<b>스크롤가능영역</b>)<br> 
<b>frame layout guide</b>를 통해 <b>스크롤방향</b>과 같은 frame 관련 설정에 집중할 수 있도록 합니다.
 
 ---
## 활용하기: 키보드 올라올 때 ScrollView의 위치 조정하기
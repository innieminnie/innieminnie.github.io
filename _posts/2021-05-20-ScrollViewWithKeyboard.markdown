---
layout: post
title: "키보드 올라올 때 ScrollView의 위치 조정하기"
date: 2021-05-20
categories: ScrollView AutoLayout
---
---

iOS 개발을 하다보면 자주 마주치게되는 부분인데요,
UITextField나 UITextView가 화면의 하단에 있을 경우, 입력을 위해 tap했을 때 키보드가 올라오면서 입력창 부분이 가려지는 현상이 발생합니다.<br>
해당 상황을 스크롤뷰를 활용해서 키보드가 올라갔을 때 입력창(UITextField) 아래에 키보드가 위치할 수 있도록하여 키보드에 의해 입력창이 가려지는 현상을 해결해보겠습니다.<br>
또한 (1) 입력창 이외의 부분을 터치했을 때 혹은 (2) return 키를 입력하였을 때 키보드가 내려가도록 구현해보겠습니다.<br>
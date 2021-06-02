---
layout: post
title: "Localization 지역화"
date: 2021-06-02
categories: Localization Internalization
---
---

## 목차
1. [Internationalization 과 Localization](#internationalization-과-localization)
    - 1. Localizable.strings (Localization 파일) 추가하기
    - 2. 언어 추가하기
    - 3. String content 지역화 (Localize 하기)
    - 4. 지역/언어 설정을 통해 확인하기
1. [Storyboard Localization](#storyboard-localization)
1. [Numbers Dates Times](#numbers-dates-times)
1. [더 공부해 볼 내용](#더-공부해-볼-내용)

---
### Internationalization 과 Localization
Localization(지역화)이란, 기기가 <b>설정된 언어/지역</b>에 따라<br> 앱이 나타내는 <b>언어나 표현방식</b>을 달리하기 위해<br> Xcode에서 지원하고 있는 <b>User Experience를 향상시키는 기능</b>입니다.

앱을 Localization을 적용하기 이전에, 우선 Internationalization(국제화)를 적용해야합니다. Internationalization은 앱이 다른 지역/언어/문화 등에 적용할 수 있는 상태를 만들어주는 과정입니다. Internationalization 과정을 통해 단순히 앱이 지역언어로의 '번역'이 가능하게 할 뿐 아니라 다양한 시간대, 다양한 숫자표기형식 및 날짜 형식, 다양한 측정단위 등을 지원할 수 있는 상태로 만들어줍니다.

![2021-06-02-internationalizationandlocalization](/assets/img/2021-06-02-internationalizationandlocalization.png)

과정을 간단히 요약하자면

1. 개발과정에서 User Interface와 Code 에 대해 국제화 작업을 진행합니다.
1. pseuolocalizations와 타 지역 설정 과정을 통해 앱을 테스트합니다.
1. localize에 대한 준비가 완료되면, standard file formats를 사용하여 localizable text를 export하고, 여러 언어로 번역될 수 있도록 localization team에 제출합니다.
1. 번역이 완료되면 localization을 프로젝트에 import 한 후, 지원하는 언어 및 지역에 대해 앱을 테스트합니다.

이제 Xcode Project를 활용하여 과정을 진행해보겠습니다.

---
### 1.Localizable.strings (Localization 파일) 추가하기
XCode는 해당 파일을 자동적으로 translation file로 인식하여 번역 기능을 돕습니다. 하나의 파일 당, 하나의 언어를 지원하고 key-value 형식을 통해 파일을 작성합니다. <b> File > New > File </b> 에서 <b>Strings File</b>을 선택하고 <b>Localizable.strings</b>로 파일 이름을 설정한 뒤, 오른쪽 패널에서 <b>Localized...</b>버튼을 클릭합니다.

1            |  2
:-------------------------:|:-------------------------:
![2021-06-02-localization01](/assets/img/2021-06-02-localization01.png)  |  ![2021-06-02-localization02](/assets/img/2021-06-02-localization02.png)

### 2.언어 추가하기
Project의 settings에서 설정합니다.
![2021-06-02-localization03](/assets/img/2021-06-02-localization03.png)
![2021-06-02-localization04](/assets/img/2021-06-02-localization04.png)
![2021-06-02-localization05](/assets/img/2021-06-02-localization05.png)

추가한 언어에 따라 파일은 localize 가 가능하다라는 과정을 설정해주었습니다. 언어별로 파일이 Localizable.strings 하위에 생성되는 것을 알 수 있습니다.

![2021-06-02-localization06.png](/assets/img/2021-06-02-localization06.png)

MainStoryboard 파일 또한 기본 스토리보드에 대응하여 언어별로 strings 포맷으로 추가되었어요!

### 3.String content 지역화 (Localize 하기)
먼저 ViewController와 Storyboard에 간단한 UI 구성을 해보겠습니다.
![2021-06-02-localization07](/assets/img/2021-06-02-localization07.png)<br>
<b>NSLocalizableString</b>을 활용하여 "GREETING", "INTRODUCTION", "CLICK" 과 같은 key 값을 넣습니다.

이제 Localizable.strings의 하위 폴더에 언어별로 string에 대한 값을 설정합니다.
![2021-06-02-localization08](/assets/img/2021-06-02-localization08.png)

### 4. 지역/언어 설정을 통해 확인하기
<b>Product > Scheme > Edit Scheme... </b> 의 <br><b>Run > Options > App Language / AppRegion</b> 에서 설정이 가능합니다.<br>
![2021-06-02-localization09](/assets/img/2021-06-02-localization09.png)

---
### Storyboard Localization
지역화 작업을 하다보면, storyboard의 요소에 직접 접근해야하는 경우가 있습니다. 그때 Main.Storyboard의 하위파일에서 작업을 진행하면 되는데요,
![2021-06-02-localization06.png](/assets/img/2021-06-02-localization06.png)

Main.string(Korean)을 예로 들어보겠습니다.

먼저 Storyboard에 UITextField를 추가해보겠습니다.
![2021-06-02-localization10](/assets/img/2021-06-02-localization10.png)

해당 UIComponent를 선택하면 옆에 <b>Object ID</b>를 확인할 수 있습니다. Object ID를 활용해서 원하는 값을 설정합니다.

![2021-06-02-localization11](/assets/img/2021-06-02-localization11.png)

코드에서 작성할 필요가 없지만, 지역화가 필요한 경우에는 해당 파일에서 작성해주면 되겠네요!
 
그러면 Localizable.strings 파일과 Main.strings에 공통요소에 대해서 다르게 설정해주면 어떻게 될지 확인해보았습니다. 위에서 greetingLabel.text를 "안녕하세요" 라고 설정해주었는데, Main.strings(Korean) 파일에 다른 값을 설정해보았습니다.
![2021-06-02-localization12](/assets/img/2021-06-02-localization12.png)
![2021-06-02-localization13](/assets/img/2021-06-02-localization13.png)

Localizable.strings 파일의 설정값이 우선적으로 작동합니다. ViewController에서 text 셋팅과정을 주석처리 했을 때 비로소 '안녕하지않아요' 로 나타나는 것을 확인할 수 있었습니다.

---
### Numbers Dates Times 
#### Numbers
- NumberFormatter를 활용합니다.
![2021-06-02-localization14](/assets/img/2021-06-02-localization14.png)

먼저 PresetNumberStyle을 통해 Decimal 값을 US -> Italy 로 전환해보겠습니다.
![2021-06-02-localization15](/assets/img/2021-06-02-localization15.png)
![2021-06-02-localization16](/assets/img/2021-06-02-localization16.png)

Italy어로 설정하고 run 시키니  number format이 변경된걸 확인할 수 있었습니다!

#### Dates & Times
- DateFormatter를 활용합니다.
![2021-06-02-localization17](/assets/img/2021-06-02-localization17.png)
![2021-06-02-localization18](/assets/img/2021-06-02-localization18.png)

DateFormatter의 클래스메서드 localizedString 의 설정값에 따라 다양하게 표현가능합니다.

---
### 더 공부해 볼 내용
이번에는 NSLocalizedString을 활용해서 UI에 표시되는 내용들을 지역별로 설정한 값들에 맞게 표현하는 것에 대해서 다뤄봤는데요, <b>RTL Language</b>도 지원가능하고, <b>Localization File을 Export/Import</b> 과정 또한 필요합니다. 이에 대해선 다음 포스팅에 작성해보겠습니다 :)

---
참고:<br>
<https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPInternational/Introduction/Introduction.html><br>
<https://medium.com/swlh/app-localization-in-swift-ios-swift-guide-baa2c2e4298e><br>
<https://medium.com/doyeona/localizations-in-swift-ff3e580e3605><br>

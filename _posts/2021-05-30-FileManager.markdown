---
layout: post
title: "iOS File System과 FileManager 활용하기"
date: 2021-05-24
categories: FileSystem FileManager
---
---

## 목차
1. [iOS File System Basic](#ios-file-system-basic)
1. [FileManager](#filemanager)
1. [File 접근하기](#file-접근하기)
1. [File Existence 확인하기 및 File 생성하기](#file-existence-확인하기-및-file-생성하기)
1. [File 읽기](#file-읽기)
1. [File 삭제](#file-삭제)
---
### iOS File System Basic
유저 및 시스템의 데이터 파일을 영구적으로 저장하는 것은 프로그래밍에서 매우 중요한 영역 중 하나입니다. '파일'과 '디렉토리'의 계층적 구조를 통해서 파일을 관리합니다.<br>
먼저 iOS의 File System 구조에 대해 간단히 살펴보겠습니다.
![2021-05-30-iosfilesystem](/assets/img/2021-05-30-iosfilesystem.png)
#### Sandbox?
'보안' 목적을 위해 하나의 앱은 하나의 sandbox 디렉토리를 갖습니다. 앱을 설치할 때 자동적으로 이 영역 내에 기본적인 디렉토리들이 생성됩니다.

샌드박스 내의 주요 디렉토리가 담는 파일의 내용을 정리해보았습니다.

- <b>Bundle Container</b>의 <i>AppName.app</i>: 앱의 bundle (앱 자체와 앱 관련 자원들) 저장<br>
- <b>Data Container</b>: 앱과 유저를 위한 data<br>
    - <b>Documents</b>: 
        유저가 생성한 콘텐츠 저장. 디렉토리의 내용은 유저가 file sharing을 할 수 있도록 설정할 수 있습니다. 따라서 해당 디렉토리 내에는 유저에게 보여질 수 있는 파일들을 담도록 하는 것을 권장합니다. 또한 iTunes/iCloud 백업이 가능합니다.<br>
        - <b>Documents / Inbox</b>: 
            앱 외부에서 앱에게 열람 권한을 요청하는 파일에 대해 저장합니다. 해당 디렉토리에서 파일의 읽기 및 삭제는 가능하지만, 새로운 파일을 생성하거나 기존 파일에 쓰기 작업을 진행할 순 없습니다. 예를 들어, 외부의 동영상 플레이어 프로그램에서 동영상 파일을 받을 때(동영상에 대한 수정작업은 불가능하다고 제한할 때) 해당 파일은 Inbox 디렉토리에 저장하는 것이 적합하다고 볼 수 있습니다.<br>
    - <b>Library</b>: 
        유저의 데이터 파일이 아닌 파일에 대해서 최상위 계층의 디렉토리입니다. 주로 유저에게 파일을 노출시키고 싶지 않을 때 사용합니다. Library / Application Support 나 Library / Cache 가 주로 사용되며, 직접 custom하게 하위 디렉토리를 생성할 수도 있습니다. Cache 하위 디렉토리를 제외하곤 iTunes/iCloud 백업이 가능합니다.
        - <b>Library / Application Support</b>: 
            앱이 실행하는데 사용되지만, 사용자에게 숨겨져야하는 파일 (데이터파일, 구성파일, 템플릿 및 앱번들에서 수정된 리소스)
         - <b>Library / Caches</b>: 
            임시데이터보다 오래 지속되어야하지만 support 데이터가 아닌 모든 데이터 (DB캐시파일, 일시적으로 다운로드 할 수 있는 Contents)
    - <b>tmp</b>:
        앱 런칭 사이에 영구적으로 저장할 필요가 없는 임시파일을 작성합니다. 더 이상 필요하지 않으면 앱에서 해당 디렉토리의 파일을 삭제해야할 필요가 있으며 앱이 실행되지 않을 때(not running), 시스템에 의해 디렉토리가 비워집니다. iTunes/iCloud 백업이 불가능합니다.
- <b>iCloud container</b>: 유저의 iCloud의 접근이 필요할 경우, runtime에 접근 요청

---
### FileManager
iOS의 파일시스템에 접근하기 위해 편리한 FileManager API를 제공합니다. FileManager를 통해 device의 앞서 설명한 iOS File System의 디렉토리에서 파일접근, 파일수정, 파일저장, 파일 삭제가 가능합니다.

iOS의 파일들은 <i>'path'</i>(경로)를 가지고 있고 해당 경로를 담고 있는 <i>URL</i> 을 활용합니다.

---
### File 접근하기
예를 들어, 앱의 Directory폴더의 abc.txt파일에 접근해야한다면
```swift
let fileURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0].appendingPathComponent("abc.txt")

```
와 같이 작성할 수 있습니다. 
- <b>첫번째 파라미터  FileManager.SearchPathDirectory</b>: 
    찾고자 하는 standard directory의 종류 (Documents, Caches, etc...)
- <b>두번째 파라미터 FileManager.SearchPathDomainMask</b>: 
    찾고자 하는 범위가 user domain이냐, local domain이냐를 정합니다. (예를 들어, .applicationDirectory 라면 user domain 일 경우, ~/Applications이고 local domain일 경우 /Applications가 됩니다.)
- FileManager의 urls(for:in:) 은 찾고자하는 경로에 대해 여러 파일/디렉토리가 존재할 수 있기에 [URL]을 반환합니다.따라서 array의 index로 접근하여 해당 url에 <b>appendingPathComponent("abc.txt")</b> 와 같이 더해줍니다.

(코드를 작성한 후, Simulator 실행을 통해 작업을 할 때 해당 파일의 위치를 직접 확인하고 싶다면 <b>path.absoluteString</b> 을 확인해보세요!)

---
### File Existence 확인하기 및 File 생성하기
위에 작성한 fileURL를 기준으로 특정 경로에 파일이 존재하는지 확인하고, 없으면 파일을 생성하는 코드를 작성해보겠습니다.

```swift
guard fileManager.fileExists(atPath: fileURL.path) else {
    fileManager.createFile(atPath: fileURL.path, contents: nil, attributes: nil)
    return
}
```

---
### File 읽기
File에 담긴 내용을 불러오는 형식에 따라 달리 작성할 수 있습니다.<br>
String으로 읽어오고 싶다면,
```swift
do {
    let stringData = try String(contentsOf: fileURL)
} catch {
    print(error.localizedDescription)
}
```

Data타입으로 읽어오고싶다면,
```swift
do {
    let savedData = try Data(contentsOf: fileURL)
} catch {
    print(error.localizedDescription)
}
```

---
### File 쓰기
String을 쓰고 싶다면,
```swift
let newContent = "def"

do {
    try newContent.write(to: fileURL)
} catch {
    print(error.localizedDescription)
}
```

Data타입을 쓰고 싶다면,
```swift
do {
    let newData = try newContent.data(using: .utf8)
    try newData.write(to: fileURL)
} catch {
    print(error.localizedDescription)
}
```

---
### File 삭제
```swift
try FileManager.default.removeItem(at: fileURL)
```

---
참고:<br>
https://learnappmaking.com/filemanager-files-swift-how-to/#reading-a-file-with-swift<br>
https://nshipster.com/filemanager/
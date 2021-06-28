---
layout: post
title: "MVVM in iOS"
date: 2021-06-29 01:16:00
categories: MVVM binding DesignPattern
---
---

## 목차
1. [MVC패턴 이해하기](#mvc패턴-이해하기)
    - Traditional MVC와 Cocoa Framework의 MVC
2. [MVVM패턴 살펴보기](#mvvm패턴-살펴보기)
    - Data binding과 Observable
3. [ViewModel 만들기](#viewmodel-만들기)
4. [Data Binding 과 Observable](#data-binding-과-observable)
5. [Observable 적용해보기](#observable-적용해보기)
6. [마무리](#마무리)

---
### MVC패턴 이해하기
- Traditional MVC와 Cocoa FrameWork의 MVC

전통적인 MVC 디자인패턴과 애플의 코코아 프레임워크에서 제시하는 MVC 디자인패턴은 유사하지만 약간의 차이가 있습니다. MVVM 디자인패턴을 알아보기 앞서, MVC패턴과 MVVM패턴의 등장배경을 간단히 알아보겠습니다. (기본적 Model, View, Controller의 의미는 안다는 전제하에 글을 진행합니다.)

<center><img src="/assets/img/2021-06-29-mvvm1.png" width="500"></center>

 위의 사진이 '전통적' MVC / 아래 사진이 '애플'이 제시하는 MVC 패턴입니다.
공통적으로 Controller가 전체적인 비즈니스 로직을 담당하고 있습니다.

 우선 전통적 MVC에서 주목할점은, View와 Model의 의존관계입니다.<br> View는 Model의 상태를 확인하기 위한 요청을 전성하고, Model은 View에게 자신의 상태를 notify(알림) 합니다. <b>(1) View와 Model의 의존관계</b>가 있을 뿐 아니라, Controller는 View에 입력된 User Action을 처리 및 뷰를 업데이트하기 위한 비즈니스 로직이 존재하고 해당 과정에서 Model을 업데이트하는 과정이 존재하기에, <b>(2) View와 Controller의 의존관계</b>와 <b>(3) Controller와 Model의 의존관계</b> 또한 존재하게 됩니다.

 복잡한 의존관계를 정돈하기 위해 애플에서 제시한 MVC는 이전보다 View와 Model의 의존관계를 끊어낸 설계를 제시했습니다. 아래 사진에서 보시다 시피 View와 Controller간의 관계는 유사하지만, 이전과 달리 <b>View가 Model의 상태를 확인하기 위해선 Controller를 거쳐가야합니다.</b> 또한 <b>Model의 업데이트와 Model의 상태확인은 오롯이 Controller와 Model 사이에서만 소통합니다.</b><br> Controller가 Model과 View사이에서 <b>중재자</b> 역할을 하고 있습니다.

 하지만 두 MVC 방식 모두 도메인의 크기가 커질수록 생기는 문제점이 있습니다. 동작 수행에 있어 항상 Controller를 거쳐가기 때문에, 처리해야하는 로직이 많아지고 복잡해질수록 Controller의 책임이 점점 커지게 됩니다. Controller에 의존적인 설계가 되어버리고 ViewController(Controller의 내용이 들어가는 부분) 에서 View 또한 같이 다뤄져 버리기에 UI 독립적이지 못해 테스트에 있어 용이하지 못합니다.

---
### MVVM패턴 살펴보기
- Data binding과 Observable

 MVVM패턴은 위에서 이야기한 MVC패턴에 대한 개선책으로 나온 디자인패턴입니다. 먼저 그림으로 살펴보겠습니다.

 <center><img src="/assets/img/2021-06-29-mvvm2.png" width="500"></center>(출처: 위키)<br>

 주요 차이점은 Controller 대신 <b>ViewModel</b>이 등장한다는 점과,<br> View와 View Model 사이에 <b>Data Binding</b>이 있다는 점 입니다.

 iOS Framework를 기준으로 말씀드리면, 

 - ViewController는 View의 역할을 담당합니다. ViewController 파일의 코드 내에는 View 관련 내용만 담습니다(View를 표시하거나, User Action에 대한 내용).<br>

 - View는 ViewModel을 인지하고 있습니다. View에서 발생하는 User Action에 따라 ViewModel에게 상황을 전달합니다.<br>

 - 반대로 ViewModel은 View에 대해 모릅니다. 하지만 자신이 담당하는 Model을 인지하고 있습니다. View에서 상황을 전달하면, ViewModel은 상황 발생 시, 상황에 맞게 Model의 정보를 얻어오거나 / Model을 업데이트하거나 등, 자신이 인지하고 있는 Model에 대한 처리를 수행합니다.<br>

 - <b>!!! ViewModel이 Model을 처리하는 과정에서 View가 바뀌어야하는 상황이 온다? -> ViewModel이 View에게 해당 상황을 직접 알려주지 않습니다!!!</b> ViewModel은 View를 인지하고 있지 않기 때문이죠<br>

 - 대신에 View는 자신이 변할 타이밍을 파악하기 위해 ViewModel을 감시하고 있습니다 (Observable 활용). Binding을 통해 감시하는 체제를 설정하고, ViewModel의 변화가 발생하면 이를 감지해서 자기 자신(View)를 업데이트합니다.(Observable 와 Binding에 대해선 밑에서 자세히 말씀드리겠습니다.)<br>

 - Model은 이전 MVC에서와 마찬가지로 데이터 자체를 담습니다. View와의 직접적 연결고리는 없습니다.<br>

---
### ViewModel 만들기
ViewModel에 대해 주관적으로 해석해보자면, Model에 대한 Wrapper 느낌입니다. Model이 스스로 행위를 수행하는 것보다, ViewModel을 통해 행위를 수행하고 ViewModel이 행동하기 위해 소스를 제공하는 느낌이었습니다.

예를 들어, Todo 라는 Model과 해당 Todo에 대한 ViewModel을 설정해보겠습니다. (가정 상황: Todo 목록을 테이블뷰에 표현한다.)

```swift
import Foundation

struct TodoList { // Model
    let todos: [Todo]
}

struct TodoListViewModel { //ViewModel
    let todos: [Todo]

    func todosCount() -> Int {
        return self.todos.count
    }

     func todoItem(at index: Int) -> TodoViewModel {
        let todo = self.todos[index]
        return TodoViewModel(todo)
    }

}
```
```swift
struct Todo: Decodable { // Model
    let title: String
    let description: String
}

struct TodoViewModel { // ViewModel
    private let todo: Todo

    init(_ todo: Todo) {
        self.todo = todo
    }

    var title: String {
        return self.todo.title
    }
    
    var desription: String {
       return self.todo.description
    }

}
```
Model은 필요한 최소한의 정보만 지니고 있고, Model의 값을 호출하거나 배열의 갯수를 세거나 등 Model을 Control하는 작업은 ViewModel에 정의되어있습니다. 이를 UITableView에 표시하는 부분에 대해서 간단하게 코드로 보여드리겠습니다.

TodoTableViewController의 fetchData 또한 MVVM을 엄격히 적용시킨다면, 리팩토링의 여지가 있음을 밝힙니다! 여기선 fetchData 보다 Todo / TodoViewModel , TodoList / TodoListViewModel 의 역할에 초점을 맞췄습니다.
```swift
class TodoTableViewController: UITableViewController {
    private var todoListVM: TodoListViewModel! // 뷰는 뷰모델을 인지한다.
    
    override func viewDidLoad() {
        super.viewDidLoad()
        fetchData()
    }
    
    private func fetchData() {
        let url = // 데이터 fetch할 url 주소
        
        // getTodos라는 네트워킹을 통한 data 추출 메소드 있다고 가정
        // 호출 데이터의 Decoding 결과 = [Todo]
        // 해당 호출에 대한 completionHandler 
        WebService().getTodos(url: url) { todos in
            if let todos = todos {
                //fetching한 데이터를 담는 뷰모델 생성
                self.todoListVM = TodoListViewModel(todos: todos)
                // 뷰 업데이트
                DispatchQueue.main.async {
                    self.tableView.reloadData()
                }
            }
        }
    }

    // 표시할 내용의 갯수에 대해 "뷰모델"을 통해 접근합니다.
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return todoListVM.numberOfRowsInSection(section)
    }
    
    // cell의 생성에 대해 "뷰모델"을 통해 접근합니다.
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        guard let cell = tableView.dequeueReusableCell(withIdentifier: "TodoTableViewCell", for: indexPath) as? TodoTableViewCell else {
            fatalError("TodoTableViewCell is not found")
        }
        
        let todoVM = self.todoListVM.todoItem(at: indexPath.row)
        cell.titleLabel.text = todoVM.title
        cell.descriptionLabel.text = todoVM.desription
        return cell
    }
}

```

---
### Data Binding 과 Observable
 보통 MVVM 패턴을 이야기하면 Data Binding과 Observable이 같이 동반되어서 나타납니다. 하지만 위에선 구체적으로 Data Binding을 설정해주는 부분이 없어보입니다. ViewModel이 Model을 컨트롤 하는건 나타나있지만, 결국 View를 업데이트 해주는 부분은 네트워크 호출 후, 결괏값에 따라 tableView.reloadData() 하는 부분 입니다.<br>

 그렇다고 위의 로직이 MVVM 패턴이 아니라고 말하긴 어렵습니다. 왜냐하면 Model과 ViewModel, 그리고 ViewController(View영역) 을 분리해주었고 각 영역간의 의존관계 (ViewModel은 Model을 인지하고, View는 ViewModel을 인지한다.) 가 분명합니다.<br>

 위의 코드에서 binding이 따로 필요없었던 이유는, View가 변하지 않기 때문입니다. Data Binding은 View가 자신이 변화하기 위해서 감지해야할 필요가 있는 ViewModel의 요소를 감지대상으로 설정해놓고, 해당 요소에 변화가 생기면 스스로 변화합니다. 즉 위에선 단순히 네트워킹 요청을 통해 TodoList를 받으면 이를 표현하고 끝이지만, 예를 들어 아이템 내용이 변하거나, 아이템의 갯수가 변하거나 등 변화요인이 있다면 data binding과 observable을 활용하는 의미가 있습니다.

 ---
 ### class Observable 만들기
 Observable은 '관찰가능한' 이라는 의미입니다. 말 그대로 관찰 대상이 되는 것(감지되는 값)에 대해 설정할 것들을 작성하면 됩니다. 

 ```swift
 class Observable<T> {
     var value: T {
         didSet {
             listener?(value)
         }
     }

     init(_ value: T?) {
         self.value = value
     }

     private var listener: ((T) -> Void)?

     func bind(_ listener: @escaping (T) -> Void) {
         listener(value)
         self.listener = listener
     }
 }
 ```

 <https://medium.com/flawless-app-stories/data-binding-in-mvvm-on-ios-714eb15e3913> 코드 참조

- value: 감지할 대상 설정 
- bind: View에서 Binding을 하기 위한 부분 ( 메소드 내에선 내부의 listener 설정)
- listener: value가 설정/변경됨에 따라 호출되는 클로저 (View의 변화 내용 담는 부분)

---
### Observable 적용해보기
```swift
struct Todo: Decodable { // Model
    let title: String
    let description: String
}

struct TodoViewModel { // ViewModel
    private var todo: Observable <Todo> = Observable(Todo(title: "", description: ""))

    init(_ todo: Todo) {
        self.todo = todo
    }

    var title: String {
        return self.todo.title
    }
    
    var desription: String {
       return self.todo.description
    }

}

// ...
// bind하는 부분 (listener에 대한 구현
todoViewModel.todo.bind { [weak self] _ in
    DispatchQueue.main.async {
        self.updateTodoView() // view의 표시내용 업데이트 메소드라 가정
    }
}
```

---
### 마무리
저의 개인적인 오랜..숙원..이었던 MVVM을 이해하는 단계에 도달한 것 같아 기분이 죠습니다! MVVM에 초점을 맞춰어 설명하면서 Observable을 직접 구현한 코드를 통해 binding의 동작원리를 살펴보았는데요, 요즘 핫한 RxSwift라던지, Combine Framework 와 같은 것들이 좀 더 '반응적(Reactive)'으로 User의 Action에 대응하기 위해 등장했고 이런 Reactive Programming에 있어서 MVVM패턴과 합이 잘 맞는다고 합니다. UI가 독립적으로 자신이 변할 포인트를 감지해서 착착 변하니까요! ViewModel의 도입에 따라 ViewController가 다루는 범위 또한 확 줄어들었죠. 굳굳 :)

---
참고:<br>
<https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html><br>
<https://medium.com/flawless-app-stories/data-binding-in-mvvm-on-ios-714eb15e3913><br>
<https://www.youtube.com/watch?v=iI0LabCYZJo&t=17s><br>

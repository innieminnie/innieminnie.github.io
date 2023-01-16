---
layout: post
title: "TableViewDataSource 재사용성 높이기"
date: 2023-01-15
categories: Reusable UIComponent TableViewDataSource
---
---

### 목차
- [프로젝트의 두 개의 유사한 화면](#프로젝트의-두-개의-유사한-화면)
- [WorkoutSelectionVC 운동종목추가역할 먼저 수정하기](#workoutselectionvc-운동종목추가역할-먼저-수정하기)
- [WorkoutListVC 운동종목목록도 바꿔주기](#workoutlistvc-운동종목목록도-바꿔주기)
- [Protocol과 Extension 활용해서 공통데이터 한 번 더 묶어보기](#protocol과-extension-활용해서-공통데이터-한-번-더-묶어보기)
- [Summary](#summary)

---
# 프로젝트의 두 개의 유사한 화면
최근 배포한 앱 내에서 유사한 화면 두 개가 있다.

|운동 종목 추가하는 뷰컨|운동 목록 표현하는 뷰컨|
|-|-|-|
|<img src="/assets/img/2023-01-15-page1.png" width = 400><|<img src="/assets/img/2023-01-15-page2.png" width = 400>|

같은 데이터를 표시하는 TableView를 사용하고 있고, cell을 tap할 경우에 행하는 액션에서 차이가 있다. 두 화면에 대해 비교한 걸 정리해보자면,

<b>공통점</b>:  TableView에 표시하는 데이터 목록<br>
<b>차이점</b>:
  - 운동종목추가뷰컨은, cell의 multiselection이 가능하다.(선택한 종목을 운동계획에 추가한다.)<br>운동목록뷰컨은, 1개의 cell만 selectable(선택한 운동종목에 대한 설정값을 보여준다.)<br><br> 

  - 운동종목추가뷰컨은, 검색기능이 없다.<br>운동목록뷰컨은, 운동검색이 가능하다.<br><br> 

나는 이걸 각 뷰컨마다 tableview만들고/datasource랑 delegate 만들고 하고 있었다<br>
표시하는 데이터가 똑같다보니 각 ViewController가 채택하고있는 UITableViewDataSource 관련 부분의 코드 또한 중복이 많았다.<br> 그래서 합치기도전.

---
# WorkoutSelectionVC 운동종목추가역할 먼저 수정하기

공통으로 사용하는 WorkoutTableView 먼저 만든 후, 각각의 뷰컨트롤러에서 생성하는 UITableView를 WorkoutTableView로 변경했다.<br>
WorkoutTableView는 간단한 이니셜라이저와 등록할 Cell ( WorkoutTableViewCell ) 로 통일시켰다.

```swift
class WorkoutListTableView: UITableView {
   init() {
     super.init(frame: .zero, style: .plain)
     self.translatesAutoresizingMaskIntoConstraints = false

     let nib = UINib(nibName: "WorkoutTableViewCell", bundle: nil)
     self.register(nib, forCellReuseIdentifier: WorkoutTableViewCell.identifier)
     self.separatorStyle = .none
   }

   required init?(coder: NSCoder) {
     fatalError("init(coder:) has not been implemented")
   }
 }
 ```

UITableViewDataSource 를 커스텀하게 생성해서 작성했다. 기존 WorkoutListViewController에 데이터소스 관련 프로토콜에 있는 부분을 복붙했다.

```swift
class WorkoutListDataSource: NSObject, UITableViewDataSource {
   func numberOfSections(in tableView: UITableView) -> Int {
    //section 수 (필수 X)
   }

   func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    //하나의 section 당 row 수 (필수작성메서드)
   }

   func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
     //사용할 cell 과 초기 셋팅 작업 (필수작성메서드)
     return cell
   }

  // 이외 선택적으로 프로토콜의 메서드 작성 ...
 }
 ```
<br>
그리고 나서 WorkoutSelectionVC에서 datasource를 설정해준다.

```swift
private var dataSource: WorkoutListDataSource!

 private func setUpListTableView() {
    self.dataSource = WorkoutListDataSource()
    
    workoutListTableView.dataSource = dataSource
    workoutListTableView.delegate = self
    workoutListTableView.allowsMultipleSelection = true
  }
```

---
# WorkoutListVC 운동종목목록도 바꿔주기
WorkoutListVC도 똑같은 데이터를 사용하기에, 위에서 만든 WorkoutListTableViewDataSource를 똑같이 설정해주면 된다.<br>
그런데 여기서는 "검색" 기능도 들어있다.<br>

내 프로젝트에선 해당 VC에서 '검색을 하고 있는 경우' 에는 검색결과에 따라 section / row 가 달라져서 해당 VC에서도 조건에 따른 분기가 생긴다.<br> 따라서 DataSource부분에 조건탐색 과정이 들어가야하므로 해당 UITableViewDataSource가 검색중인지 감지해야한다.<br>
- 기존
  기존에는 WorkoutListVC 에서 아래와 같이 isSearching을 numberOfRowsInSection, cellForRowAt과 같은 곳에서 확인했다.
  ```swift
  private var isSearching: Bool {
      let isSearchBarActive = self.navigationItem.searchController?.isActive ?? false
      let isSearchBarEmpty = self.navigationItem.searchController?.searchBar.text?.isEmpty ?? false
      return isSearchBarActive && !isSearchBarEmpty
  }
  ```
<br>

- 변경
  isSearching에 대한 정보를 WorkoutListTableViewDataSource가 알고 있어야하므로, UISearchBarDelegate 프로토콜을 채택하여 해당 부분에서 isSearching 스위칭이 발생하도록 개선했다.

  ```swift
  extension WorkoutListDataSource: UISearchBarDelegate {
    func searchBarCancelButtonClicked(_ searchBar: UISearchBar) {
      self.isSearching = false
    }
  
    func searchBarTextDidBeginEditing(_ searchBar: UISearchBar) {
      self.isSearching = true
    }
  }
  ```

그리고 WorkoutListDataSource에 isSearching(Bool)와 searchingList([운동종목])을 프로퍼티로 넣고<br>

WorkoutListVC에선
```swift
extension WorkoutListViewController: UISearchResultsUpdating {
  func updateSearchResults(for searchController: UISearchController) {
    guard let searchingText = searchController.searchBar.searchTextField.text else {
      return
    }

    // 이 부분에서 업데이트되는 searchingList를 전달한다.
    self.dataSource.showSearchData(searchingList:  검색데이터가 담긴 배열) 
    self.tableView.reloadData()
  }
}
```

이제 WorkoutListTableView는 isSearching 여부도 판단하게 된다.
```swift
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return  isSearching ? 검색결과갯수 : 해당 section에 맞는 운동종목갯수
  }
  
  func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    
    //...cell 선언 과정
    
    let workout = isSearching ? self.searchingList[indexPath.row] : 해당 row에 맞는 운동종목 표기

    cell.setUp(with: workout)
    
    return cell
  }
```

---
# Protocol과 Extension 활용해서 공통데이터 한 번 더 묶어보기
여기까지 하다보니, 두 뷰컨에서 모두 WorkoutListTableViewDataSource를 생성하고 있다.<br> 근데 어차피 같은 데이터를 보여주고, 변경해도 똑같이 변경되는데, 그리고 UITableViewDataSource의 프로토콜 채택해서 관련 메서드들도 다 오픈되어야하는데 하나의 datasource를 두고 이걸 가져다 쓰는게 더 나을 거 같다는 생각이 들었다 <br>

그래서 WorkoutListTableViewDataSource를 싱글톤 패턴에 맞춰 변경한 뒤, 프로토콜을 만들어 그걸로 데이터에 접근할 수 있도록 변경했다.

```swift
protocol ContainWorkoutList {
  var workoutListDataSource: WorkoutListDataSource { get }
}
extension ContainWorkoutList {
    var workoutListDataSource: WorkoutListDataSource {
      return WorkoutListDataSource.shared
    }
}
```

이렇게 protocol + extension을 활용해서 만든 후 두 VC는 모두 해당 프로토콜을 채택한다. 그러면 따로 생성안해도 아래와 같이 설정할 수 있다.
```swift
self.tableView.dataSource = workoutListDataSource
```

---
# Summary
일단 뷰컨트롤러의 코드길이가 훅 줄어서 너무 좋다.<br>
코드를 재활용할 수 있는 방법을 찾아서 이번 리팩토링이 의미가 있었다.<br>
이 부분 말고도 다른 부분도 캐면 여럿 나올 것 같다<br>


그리고 더 고민해보고싶은 부분은, 지금 UITableViewDataSource가 하나의 고정된 데이터만을 다루고 있다(운동종목).<br> 여기서 생성하는 부분에 표현할 데이터나 필요한 소스들을 던지고 / Generic을 좀 더 활용해보면 더 추상화된 코드가 나올 수 있다고 생각들었다.<br> 어떠한 데이터를 던져도 표현가능한 데이터소스를 만드는 것을 더 초점에 두고 고민해봐야할 것 같다.<br>

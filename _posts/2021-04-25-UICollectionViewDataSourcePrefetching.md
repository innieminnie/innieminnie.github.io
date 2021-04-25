---
layout: post
title: "UICollectionViewDataSourcePrefetching"
date: 2021-04-12 01:37:00
categories: Prefetching UITableViewDataSourcePrefetching UICollectionViewDataSourcePrefetching 
---
---

## DataPrefetching

- prefetch 기술 사용 이전의 문제점

​	iOS에서 UITableView나 UICollectionView를 활용해서 정보를 표현하는 경우가 많습니다. 표현하고자 하는 정보를 가져오는 작업이 많은 비용을 필요로 할 때( 데이터 크기가 큰 이미지를 다운로드 하거나, 비교적 시간이  오래 걸리는 네트워크 통신 작업(async)을 통해 정보를 호출하는 경우), TableView나 CollectionView의 cell 구성 작업은 지연되고, 사용자에겐  <b>' 앱이 느리다 :( , 스크롤이 버벅거린다 :( '</b> 는 경험을 제공할 수 있습니다. 

[Prefetching Collection View Data](https://developer.apple.com/documentation/uikit/uicollectionviewdatasourceprefetching/prefetching_collection_view_data) 의 샘플코드가 기본적인 동작방식을 잘 보여주고 있어 해당 코드를 기준으로 설명드리겠습니다. 추천추천~!

- Prefetching의 동작방식

​	Data Prefetching은 현재 보여지는 cell이 아닌, <b>곧 보여질 cell들</b>을 대상으로 미리 cell의 구성작업을 진행합니다. 밑의 이미지에서 "Prefetched collection view cells"가 prefetching의 대상입니다.

![2021-04-25-01.png](/assets/img/2021-04-25-01.png)

곧 디스플레이될 셀들의 구성작업이 끝난 후 실제로 디스플레이될 때, 이미 작업이 완료된 셀들이기에 셀이 원하는 정보를 표현하는 데까지 보여주는 시간이 단축되고 스크롤 시, 높은 만족감을 제공할 수 있습니다.


---
## UICollectionViewDataSourcePrefetching

​	prefetching 기술을 활용하기 위해선 정보를 표현하고자하는 view가 UITableView의 인스턴스냐, UICollectionView의 인스턴스냐에 따라 각각 UITableViewDataSourcePrefetching 프로토콜  또는UICollectionViewDataSourcePrefetching 프로토콜을 준수해야합니다. 두 프로토콜의 동작방식은 유사하기에 UICollectionView를 기반으로 서술해보겠습니다.



1. <b>UICollectionViewDataSourcePrefetching 프로토콜 준수하기( Prefetching 기본 설정하기 )</b>

    기본적으로 collectionView에 대해 UICollectionViewDatasource와 UICollectionViewDatasourcePrefetching 프로토콜을 준수한 이후 collectionView의 dataSource와 prefetchDatasource를 설정합니다. dataSource에는 표현하고자 하는 정보 자체를 담는 'Model'이 들어있겠네요!

    ```swift
    // Set the collection view's data source.
    collectionView.dataSource = dataSource

    // Set the collection view's prefetching data source.
    collectionView.prefetchDataSource = dataSource
    ```
1. <b>무거운 작업을 할당하기 ( Load Data Asynchronously )</b>

    비동기적으로 데이터를 로딩하는 작업을 무거운 작업에 대한 예시로 보여드리겠습니다. 

    UICollectionViewDataSourcePrefetching프로토콜을 준수하게되면 밑에 나와있는 프로토콜의 메소드 를 필수적으로 구현해야합니다. 해당 메소드 내에 작업을 할당합니다. 메소드의 prefetchItemsAt indexPaths: [IndexPath] 파라미터가 '곧 디스플레이 될 셀의 indexPath' 로 시스템에서 할당한 갯수만큼 prefetching을 진행합니다.

    ```swift
    func collectionView(_ collectionView: UICollectionView, prefetchItemsAt indexPaths: [IndexPath]) {
        
        for indexPath in indexPaths {
            let model = models[indexPath.row]
            asyncFetcher.fetchAsync(model.identifier) // 무거운 비동기 작업!
        }
    }
    ```

---
## WWDC2016 What's New in UICollectionView in iOS 10

해당 프로토콜이 도입되던때 (iOS10부터) 의  WWDC 발표자료를 좀 더 살펴보겠습니다! 애플에서 prefetching관련프로토콜을 왜 도입하게 됐는지, 도입 후 갖게되는 성능적인 효과를 좀 더 자세하게 보여줍니다.



- Smooth Scrolling

![2021-04-25-02.png](/assets/img/2021-04-25-02.png)

  

  영상 3:30 초쯤부터 prefetch 도입이전과 도입이후의 비교를 직접 보시는 걸 추천드립니다!

  ​	prefetch 이전에는 cell의 row별로 displaying view에 진입시 cell이 구성됩니다. 스크롤을 진행한다고할 때 [35번~41번], [42번~48번], [49번~55번], [56번~62번]  단위로 cell이 구성된 후로 진입을 하게 됩니다. 영상을 보시면 약간 뚝뚝 끊어지는 느낌으로 cell이 진입하시는 걸 느낄 수 있습니다. 즉 cell의 row가 진짜 필요한 순간에 (display 시점 진입 직전) row에 해당하는 cell들을 구성해서 보여주는 작업이 스크롤 시 진행됩니다.  이러한 문제점을  <b>Dropping Frames</b> 라고 합니다.



- Dropping Frames

  사용자에게 스크롤이 부드럽다는 경험을 제공하기 위해선 app은 1초에 60프레임을 제공해야합니다. 즉 하나의 프레임이 16ms마다 window에 그려져야 합니다. 



![2021-04-25-03.png](/assets/img/2021-04-25-03.png)

​	밑의 세 개의 영역이 각 영역당 하나의 프레임을 나타낸다고 가정했을 때, 체크표시된 부분이 good frame으로 16ms내에 요구작업들을 모두 수행했다는 것을 나타냅니다. 반면 중간의 엑스표시된 부분이 16ms내에 요구작업들(파란색  막대기)을 완성하지 못한 채 다른 frame으로 넘어가는 상황입니다. 이와 같은 상황을 Dropping Frames라 합니다. 



![2021-04-25-04.png](/assets/img/2021-04-25-04.png)

![2021-04-25-05.png](/assets/img/2021-04-25-05.png)



16ms를 기준으로 윗부분에 해당하는 red zone에 도달하게 되는 경우를 밑의 수평적인 그래프와 같이 모두 safe하게 만들어주기 위해 prefetching 기술이 고안되었습니다. 



- Life Cycle of a Cell

cell life cycle을 보면 prepareForReuse -> cellForItemAtIndexPath -> willDisplayCell -> didEndDisplayingCell 순으로 진행됩니다. iOS9 이전의 life cycle에서 didEndDisplayingCell은 cell이 보여지는 뷰를 벗어나는 즉시 실행되고 사용된 cell은 바로 reuseQueue로 들어가 다음에 사용될 상황을 준비합니다. 

​	iOSX 부턴 " hesitant (지연) "을 적용하여 willDisplayCell의 시점을 조금 더 늦춰 cell이 보여지는 뷰 진입 직전에 넣고, didEndDisplayingCell 또한 마찬가지로 뷰를 벗어난 시점보다 조금 더 늦춰 reuseQueue에 넣는 작업을 미뤘습니다. 

​	" hesitant (지연) "를 통해 만약 이전에 보여준 cell을 뷰 밖으로 넘겼다가 다시 보고싶어 스크롤 방향을 전환하여 다시 보게된다면, cell을 처음의 과정부터 다시 구성하는 게 아니라 아직 reuseQueue에 들어가지 않은 cell을 (이전에 구성작업을 완성하고 유지하고 있는 cell)을 willDisplayCell부터 시행하여 볼 수 있습니다. cellForItemAtIndexPath에서 주로 작업의 크기가 무거운 작업을 하게 되는데 해당 과정을 다시 거치지 않아도 되는 효과를 가져옵니다.



​	이런 cell의 life cycle의 미세한 변화가 스크롤 성능변화를 이끌어 주었습니다. 영상의 11:25 부분에 스크롤의 개선을 보여주는데요, 

![2021-04-25-02.png](/assets/img/2021-04-25-02.png)

앞서 설명한 것과 달리 스크롤을 하면서 62번 이후의 cell이 아직 진입전이지만 63번 cell부터 순차적으로 주르륵 이미 셀이 구성되어서 진입 시 완성된 cell을 보여줍니다. !WoW!


---
## Prefetching API Tips

이제 prefetching 기술을 사용할 때 몇 가지 팁에 대해서 말씀드리겠습니다.

- GCD나 OperationQueue를 활용하라.
  - 비동기적으로 prefetching작업을 하도록 해서 main thread의 작업을 해치지 않도록 합니다.

- prefetching은 필수가 아니라 보조적인 기술이다.
  - 유저가 굉장히 빠른 속도로 스크롤을 해서 prefetching의 시간 또한 확보하지 못할 수 있습니다. 이러한 경우에는 prefetching보다 유저에게 보여지는 뷰를 구성하는 작업이 우선시 되어야합니다. 
- cancelPrefetching API를 활용하라.
  - 유저가 스크롤 하여 보는 화면이 어떤 것이냐에 따라(예를 들어, 스크롤 방향 전환) cancelPrefetching API 활용으로 불필요한 prefetching작업을 줄이는 것에 대해 고민해보면 좋을 것 같습니다.


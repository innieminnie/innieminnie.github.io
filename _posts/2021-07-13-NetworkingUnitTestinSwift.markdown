---
layout: post
title: "Networking Unit Test in Swift"
date: 2021-07-13 16:50:00
categories: XCTest TDD TestDouble DependencyInjection
---
---

## 목차
1. [네트워킹에서의 Unit Test](#네트워킹에서의-unit-test)
    - 일반적 네트워킹 상황에서 유닛테스트 작성 시 발생하는 문제
2. [Test Double와 Dependency Injection](#test-double-와-dependency-injection)
    - Test Double 이란?
    - Dependency Injection 이란?
3. [적용하기](#적용하기)
    - URLSessionProtocol 과  URLSessionDataTaskProtocol 만들기
4. [유닛테스트 작성하기](#유닛테스트-작성하기)
4. [마무리](#마무리)

---
### 네트워킹에서의 Unit Test
 - 일반적 네트워킹 상황에서 유닛테스트 작성 시 발생하는 문제

 TDD를 적용하기 위해 우리는 유닛테스트를 작성합니다. 해결하기 위한 상황을 가정하고, 해당 상황을 성공적으로 수행하기 위해 코드를 리팩토링을 합니다. 
 
 하지만 네트워크 호출이 필요한 상황이라면, 약간 복잡해집니다. 실제 네트워크 호출에 따른 결괏값을 예상해야하고, 네트워크 호출에 따라 결괏값이 네트워크 상황에 따라, 서버에 저장되어 있는 데이터의 변동에 따라 달라질 수 있습니다. 네트워크 상황에 "의존적"이게 됩니다. 즉, 일관적인 결괏값을 보장하지 못하기에 유닛테스트 작성 시 어려움에 처합니다.

 그러면, 실제 네트워크 호출을 하지 않더라도 성공/실패의 상황을 만들고 그에 따른 결괏값을 도출해낼 수 있도록 판을 짜야합니다.

---
### Test Double와 Dependency Injection
- <b>Test Double 이란?</b>

실제 네트워킹 상황과 유사한 환경을 조성하기 위해, Test Double에 대해 간단하게 짚고 넘어가보겠습니다. 현재 문제 상황과 동일하게 <b>테스트하려는 객체와 관련된 객체를 사용할 때, 해당 객체와 유사하게 만드는 것</b> 이 포인트입니다. 
    
대표적으로 이러한 가짜 객체의 종류가 5가지로 <b>Dummy, Stub, Spy, Mock, Fake</b>가 사용됩니다. 5가지의 구체적인 차이점에 대해선 <https://woowacourse.github.io/tecoble/post/2020-09-19-what-is-test-double/> 에 설명되어 있으니 읽어보시면 좋을 것 같아요! 

- <b>Dependency Injection 이란?</b>

현 상황에서 사용할 테스트더블의 종류는 Mock입니다. 가짜 네트워크 호출을 통해 예측되는 값을 설정해주어야 하고, 값에 따라 행위가 달라질 수 있기 때문입니다.

여기서 Dependency Injection (의존성 주입) 의 개념에 대해서 되짚어 볼 필요가 있습니다. 

실 네트워크 상황에선 URLSession(iOS에서 네트워킹시 기본적으로 사용되는 타입)에 따라 의존적입니다 (Product Code). 하지만 유닛테스트 시에는 실 네트워크 상황과 더불어 네트워크가 연결되지 않은 상황에서도 동작해야 합니다 (Test Code). 

Product Code와 Test Code의 네트워크 호출 관련 타입이 <b>공통 프로토콜 (URLSessionProtocol)</b>을 채택하고 구체적 구현부만 변화를 준다면 해당 프로토콜로 <b>의존성이 이관</b>됩니다. 

- URLSessionProtocol을 구현한다.
- MockURLSession을 구현한다.(URLSession의 동작을 모방한 가짜 객체)
- URLSession이 URLSessionProtocol을 채택한다. 
    > 네트워크 호출 시 URLSession.shared.dataTask ~를 수행한다.
- MockURLSession이 URLSessionProtocol을 채택한다. 
    > 네트워크 호출 시 MockURLSession.dataTask~ 를 수행한다.
    > dataTask ( return 값이 URLSessionDataTask ) 또한 MockURLSessionDataTask() 를 구현하여 활용한다.

네트워크가 연결된 상황이건, 네트워크를 가정한 상황이건 
<b> 네트워크 호출 관련 메소드를 수행한다 > 메소드 수행에 따른 데이터를 가져온다 </b> 흐름이 동작하도록 하기 위해서 네트워크 수행 관련 타입을 <b>프로토콜</b>을 활용하여 추상화해봅시다. 

---
### 적용하기
- URLSessionProtocol 과  URLSessionDataTaskProtocol 만들기

    - URLSessionProtocol 구현 이전, 실제 네트워크 호출 시 작성 방식
    ```swift
    class WebService {
        private let session: URLSession // 여기에 주목!
        
        func get(url: URL, completionHandler: @escaping ( _ data: Data?, _ error: Error?) -> Void) {
            var request = URLRequest(url: url)
            request.httpMethod = "GET"
            
            let task = session.dataTask(with: request) { (data, response, error) in
                completionHandler(data, error)
            }
            
            task.resume()
        }
    }
    ```
    ---
    - URLSessionProtocol와 URLSessionDataTaskProtocol 구현하기

        - dataTask 메소드는 URLSessionDataTask를 return 하기에, 프로토콜의 메소드는 URLSessionDataTaskProtocol을 return 합니다.
        - URLSessionDataTask는 resume()을 실행합니다. 이를 동일하게 구현하기 위해 URLSessionDataTaskProtocol 내에도 resume() 메소드를 선언해줍니다.

    ```swift
    protocol URLSessionProtocol { 
        func dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTaskProtocol
    }

    protocol URLSessionDataTaskProtocol { func resume() }
    ```

    ---
    -  URLSession 과 URLSessionDataTask의 Protocol 채택에 따른 준수사항 구현하기

    ```swift
    extension URLSession: URLSessionProtocol {
        func dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTaskProtocol {
            return dataTask(with: request, completionHandler: completionHandler) as URLSessionDataTask
        }
    }

    extension URLSessionDataTask: URLSessionDataTaskProtocol {} // 본래 URLSessionDataTask의 resume() 수행한다.
    ```

    ---
    지금까지 과정을 요약하자면, 실제 네트워크 호출 과정을 URLSessionProtocol와 URLSessionDataTaskProtocol 을 통해 접근해서 수행하는 것이지 구체적 수행 과정에서 변경된 부분은 없습니다. 이제 test를 위한 MockURLSession 과 MockURLSessionDataTask을 구현해보겠습니다.

    ```swift
    class MockURLSession: URLSessionProtocol { // acts like URLSession
        var mockURLSessionDataTask = MockURLSessionDataTask()
        var mockData: Data?
        var mockError: Error?
    
        private (set) var mockURL: URL?
    
        func successHttpURLResponse(request: URLRequest) -> URLResponse {
            return HTTPURLResponse(url: request.url!, statusCode: 200, httpVersion: "HTTP/1.1", headerFields: nil)!
        }

        // 프로토콜 채택에 따른 구현사항!!!
        func dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTaskProtocol {
            mockURL = request.url
            completionHandler(mockData, successHttpURLResponse(request: request), mockError)
            return mockURLSessionDataTask // MockURLSessionDataTask를 return 한다!!!
        }
    }

    class MockURLSessionDataTask: URLSessionDataTaskProtocol {
        private (set) var mockResumeCalled = false

        // 프로토콜 채택에 따른 구현사항!!!
        // 단순히 bool 값을 통해 resume()이 수행되는지에 대한 확인 작업만 합니다. 
        func resume() {
            mockResumeCalled = true
        }
    }
    ```

    ---
    - URLSessionProtocol 구현 이후, WebService 타입의 변화
    ```swift
    class WebService {
        private let session: URLSessionProtocol // 바뀐건 여기 뿐!
        
        init(session: URLSessionProtocol) { // 바뀐건 여기 뿐!
            self.session = session
        }
        func get(url: URL, completionHandler: @escaping ( _ data: Data?, _ error: Error?) -> Void) {
            var request = URLRequest(url: url)
            request.httpMethod = "GET"
            
            let task = session.dataTask(with: request) { (data, response, error) in
                completionHandler(data, error)
            }
            
            task.resume()
        }
    }
    ```

    이제 WebService는 네트워크 상태에 관계없이 어떤 URLSessionProtocol을 주입하느냐 에 따라 네트워크를 연결하거나, 네트워크 연결 없이 테스팅을 수행할 수 있습니다 :)

### 유닛테스트 작성하기

```swift
import XCTest
@testable import Demo
class DemoTests: XCTestCase {
    var WebService: WebService! 
    let session = MockURLSession()

    override func setUp() {
        super.setUp()
        WebService = WebService(session: session)
    }

    override func tearDown() { 
        super.tearDown()
    }

    func test_getRequestURL() throws {
        // Given
        guard let url = URL(string: "https://mockurl") else {
            fatalError("URL error")
        }
    
        // When
        webService.get(url: url) { (data, response) in
            //data 가공작업
        }
    
        // Then
        XCTAssert(session.mockURL == url)
    }   

    func test_checkResumeCall() {
        // Given
        let dataTask = MockURLSessionDataTask()
        session.mockDataTask = dataTask
        guard let url = URL(string: "https://mockurl") else {
            fatalError("URL error")
        }

        // When
        webService.get(url: url) { (success, response) in
            //data 가공작업
        }
        
        // Then
        XCTAssert(dataTask.mockResumeCalled)
    }
}
```

---
### 마무리
간단하게 URL값을 체킹하거나, resume()이 호출되는 경우에 대해 UnitTest를 작성해보았습니다. 

코드 참고: <https://masilotti.com/testing-nsurlsession-input/>

이에 더해서 JSON데이터를 파싱해서 가져오는 상황에 대해서도 추가적으로 가능합니다. 샘플 데이터를 Data 타입으로 선언해주고, webService.get(url: url) { ~ } 메소드 내에서 XCT메서드를 통해 확인할 수 있습니다. 이와 관련된 글은 
<https://techblog.woowahan.com/2704/> 에서 확인해보실 수 있습니다. 개인적으로 많은 도움이 됐구요! 

Test Double을 활용한 Unit Test에 대해 더 이해해 볼 수 있었고, 
어렵게 느껴졌던 Dependency Injection 개념을 Protocol을 통해 쉽게 접근할 수 있다는 것을 느꼈습니다. Protocol이 제공해주는 확장성을 실감할 수 있었어요! 이상입니다 :)
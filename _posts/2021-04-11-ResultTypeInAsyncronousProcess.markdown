---
layout: post
title: "비동기처리 에러핸들링에서의 Result Type 활용과 throw 방식에 대한 비교"
date: 2021-04-12 01:37:00
categories: AsyncronousErrorHandling Result 
---
---

## ResultType

​	Swift5에서부터 도입된 Result Type은 <b>Generic Enumeration</b>으로, success(성공) 또는 failure(실패)를 나타내는 <b>value</b>입니다. 또한 각 case마다 associated value(연관값)을 포함할 수 있습니다.

```swift
@frozen enum Result<Success, Failure> where Failure : Error
```

---
## Result의 동작방식

[Writing Failable Asynchronous APIs](https://developer.apple.com/documentation/swift/result/writing_failable_asynchronous_apis) 의 내용을 이해한 것을 바탕으로 작성했습니다.



​	throws 키워드를 활용하면, 작성한 내용이 실패가능성이 있음을 나타낼 수 있습니다. 하지만  <b>비동기적으로 return하는 작업(예를 들어, API모델링 하는 작업이 비동기적으로 리턴될 때)을 처리할 땐 throws 키워드를 사용할 수 없습니다.</b><br><br> Result Type이 나오기 전엔 Result와 같은 enum 타입을 직접 구현해주거나, throws 활용하여 비동기작업에 대한 처리를 진행했을텐데, 해당 문서에선 비동기적으로 return하는 작업을 처리시엔 throws키워드를 사용할 수 없다고만 나와있어 왜 안되지에 대한 고민이 있었습니다. 우선 Result Type을 비동기처리과정에서 사용하는 예시 코드를 함께 살펴보고, throws키워드를 사용할 수 없는 것에 대해선 Result Type과 비교하여 뒤에서 설명해보겠습니다.



[Writing Failable Asynchronous APIs](https://developer.apple.com/documentation/swift/result/writing_failable_asynchronous_apis) 의 예시에서 completionHandler를 비동기적으로 호출하는 과정을 보여줍니다 . EntropyError라는 Error타입을 만들고 AsyncRandomGenerator 의 fetchRemoteRandomNumber 메소드에서 EntropyError를 활용합니다.  

```swift
import Foundation

let queue = DispatchQueue(label: "com.example.queue")

enum EntropyError: Error {
    case entropyDepleted
}

struct AsyncRandomGenerator {
    static let entropyLimit = 5
    var count = 0
    
    mutating func fetchRemoteRandomNumber(completion: @escaping (Result<Int, EntropyError>) -> Void) {
        let result: Result<Int, EntropyError>
        
        if count < AsyncRandomGenerator.entropyLimit {
            result = .success(Int.random(in: 1...100))
        } else {
            result = .failure(.entropyDepleted)
        }
        
        count += 1
        
        queue.asyncAfter(deadline: .now() + 2) {
            completion(result)
        }
    }
}

var generator = AsyncRandomGenerator()

// Request one more number than the limit to trigger a failure.
(0..<AsyncRandomGenerator.entropyLimit + 1).forEach { _ in
    generator.fetchRemoteRandomNumber { result in // completion에 대한 구체적 구현부
        switch result {
        case .success(let number):
            print(number)
        case .failure(let error):
            print("Source of randomness failed: \(error)")
        }
    }
}


print("Waiting on some numbers.")
dispatchMain()
```



위의 예시에서 좀 더 나아가 설명하자면, 주로 <b>URLSession을 통한 비동기 네트워킹 과정</b>에서 네트워크 시 발생할 수 있는 에러상황들에 있어 <b>Result Type을 통해 에러핸들링</b>을 진행합니다.

```swift
class APIManager {
    let session: URLSession.shared
    
  // Something이라는 타입에 데이터를 매칭시키고 싶은 경우 (핸들링하고 싶은 구체적 내용에 따라 completion 이 바뀝니다.)
    func fetchData(completion: @escaping (Result<Something,Error>) -> Void) { 
        let request = URLRequest(url: "데이터를 받아오는 url")
        
        let task: URLSessionDataTask = session.dataTask(with: request) { (data, response, error) in
            guard let response = response as? HTTPURLResponse,
                  (200...399).contains(response.statusCode) else {
                    // Error 타입인 APIError의 에러명을 completion에 전달  
                completion(.failure(error ?? APIError.에러명)) 
                return
            }
            
            if let data = data,
               let successResponse = try? JSONDecoder().decode(Something.self, from: data) {
                 //successResponse을 completion에 전달
                completion(.success(successResponse))
                return
            }
            
            completion(.failure(APIError.에러명))
        }
        
        task.resume()
    }
}

```

---
## 비동기처리과정에서 throw 방식의 에러핸들링 :(

앞서 궁금증을 가졌던 부분인 throw를 통한 비동기처리과정 에러핸들링이 어려운 이유에 대해선,

1. 비동기처리의 내용이 포함된 메소드에 throws를 달아주면 throws가 메소드 내의 비동기처리과정의 결과(에러인지 아닌지)를 기다려주는게 아니라 메소드 호출 시점에 시행되기에 에러발생가능시점보다 먼저 throw가 되서 에러를 못잡게 됩니다. 

2. throws를 활용한 에러처리를 고수하여 1의 상황을 해결하기 위해선 <b>completion에 throws가 담긴 function 타입을</b> 담아야 합니다. 

   

코드로 위의 2번에 대해 더 구체적으로 표현하자면,https://alisoftware.github.io/swift/async/error/2016/02/06/async-errors/ 의 예시코드를 기반으로 각색하여 작성해보았습니다.)

```swift
enum APIError: Error { 
  case defaultError
}

struct Something {
  init(fromData: Data) throws { /* … */ }
  /* … */
}

typealias SomethingBuilder = Void throws -> Something // throws가 담긴 function 타입

func fetchData(completion: @escaping SomethingBuilder -> Void) { 
  let request = URLRequest(url: "데이터를 받아오는 url")
  let task: URLSessionDataTask = session.dataTask(with: request) { (data, response, error) in
    completion({ SomethingBuilder in
      if let error = error { throw APIError.defaultError }
      guard let data = data else { throw APIError.defaultError }
      return try Something(fromData: data)
    })
  }.resume()
}

fetchData { (somethingBuilder: SomethingBuilder) in
  do {
    let something = try somethingBuilder()
  } catch {
    print("\(error)")
  }
}
```



ResultType을 활용할 때와 가장 큰 차이점은 completion 부분에서 즉각적으로 성공시 반환하는 타입이냐,에러냐 가 아니라 <b>타입에 대한 try문을 반환하면서, 타입 내에서 throw를 또 진행하게 한다는 점</b>입니다. ErrorHandling을 한 번 더 해주는 상황으로 이어집니다. 복잡해지는군요 +_+..

---
## 느낀점

Result 와 throw 의 에러핸들링 방식을 비교해봤을때 <b>Result는 declarative(선언적)</b>,<b> throw방식은 imperative(명령적)</b> 하다고 생각이 들었습니다. 더불어 Result 을 활용하는것이 input에 대한 return값이 항상 동일하다, immutable하다, how(에러핸들링의 과정) 보단 what( ResultType의 연관값에 무엇을 담을건지) 에 집중하는 것 같아 함수형 프로그래밍이 지향하는 바에 가깝다고 생각하게 되었습니다. :)






---
layout: post
title: "Operation/OperationQueue의 비동기처리과정 Swift Concurrency TaskGroup 으로 리팩터링"
date: 2023-03-22
categories: Operation Concurrency Task TaskGroup
---
---

# Intro.
이전에 비동기프로그래밍을 학습하면서 진행했던 BankManager 프로젝트를 SwiftConcurrency를 적용한 방식으로 리팩토링해보았다.<br>

프로젝트는 콘솔에서 프로그램 진행을 시작하면, 랜덤 수의 은행고객들이 생성되어 은행에서 고객들의 업무를 비동기로 처리하는 것이다.<br>
업무는 크게 예금 / 대출 로 분류되며 각각 진행시간이 다르다.<br>

---
# 1. 기존 Operation / OperationQueue를 활용한 구현
ClientOperation (Operation 객체) / ClientOperationQueue(Bank가 queue 생성 및 설정하여 활용한다)

## 1-1. ClientOperation
ClientOperation은 생성 시, 아래의 properties를 갖는다.

  ```swift

  private(set) var waitingNumber: Int? // 대기번호
  private(set) var grade:  ClientGrade // 고객등급 (VVIP > VIP > normal 순으로 업무진행)
  private(set) var isLoadQualified: Bool? // 대출가능여부  (대출 심사 시 true여야만 업무 진행 가능)
  private(set) var business: BussinessType? // 고객의업무종류 (업무 종류에 따라 업무 소요시간이 상이함)
  private(set) var servicePriority: Operation.QueuePriority // 업무의 중요순위 (.high > .normal > .low 순으로 고객의 grade에 따라 정해진다)

  ```

OperationQueue에 들어간 Operation 객체가 수행할 작업은 override func main() 에 작성한다.

  ```swift
  //ClientOperation.swift

  override func main() {
      do {
        try operateBusiness()
      } catch {
        // 에러처리문 작성부분
      }
    }

    private func operateBusiness() throws {
      guard let clientBusiness = self.business else {
        return
      }
      
      print(ConsoleOutput.currentProcess(self, .start).message) // '업무시작' 콘솔 출력
      
      switch clientBusiness {
      case .deposit: // 예금업무
        handleDepositBusiness() // async
      case .loan: // 대출업무
        try handleLoanBusiness() // async
      }
      
      print(ConsoleOutput.currentProcess(self, .done).message) //'업무종료' 콘솔 출력
    }
  ```

ClientOperation의 clientBusiness에 따라 Thread.sleep의 수행시간을 다르게 한다.

  ```swift
    private func handleDepositBusiness() { // 예금업무 수행시간 0.7
      Thread.sleep(forTimeInterval: 0.7)
    }
    
    private func handleLoanBusiness() throws {  // 대출업무 수행시간 0.3
      Thread.sleep(forTimeInterval: 0.3)
      
      switch try headQuarter.handleLoanScreeningQueue(of: self) { //대출심사과정
      case true:
        Thread.sleep(forTimeInterval: 0.3) //통과 시 대출업무 수행시간 0.3
      case false:
        break
      }
    }
  ```

## 1-2. ClientOperationQueue
ClientOperation의 main()을 수행할 ClientOperationQueue 설정 부분을 작성한다.

  ```swift
  // Bank.swift

    func startWorking(from queue: [ClientOperation]) throws {
    // 타이머 시작 및 콘솔 '은행개점' 출력 과정 생략
      
      let clientOperationQueue = OperationQueue()
      clientOperationQueue.maxConcurrentOperationCount = clerkNumber
      clientOperationQueue.addOperations(queue, waitUntilFinished: true)
    }
  ```

<b>OperationQueue의 maxConcurrentOperationCount</b>
  - Operation의 작업내용을 전달하는 thread의 갯수를 제한한다. 여기서 clerkNumber는 가상의 은행원 수(3명)로 제한

<b>OperationQueue의 addOperations(_ ops: [Operation], waitUntilFinished wait: Bool)</b>
  - Operation의 배열을 OperationQueue에 추가한다.
  - waitUntilFinished: true 를 통해 OperationQueue에 추가된 모든 Operation들이 작업을 전부 수행할 때까지 thread를 block 시킨다. false로 설정하면 OperationQueue

<b> Operation의 QueuePriority</b>
  - QueuePriority가 높을수록 우선적으로 수행된다.



ClientOperation들은 동시수행이 가능한 3개의 thread에서 자리가 생기면 자신의 업무를 수행할 것이다.<br>이때 thread가 작업을 수행하는 순서는 servicePriority에 기반한다.

---
# 2. Swift Async/Await을 활용한 Refactoring
ClientOperation은 Operation의 객체가 아닌 Custom Type으로 변경되었다.<br>

변경됨에 따라 Operation의 속성 queuePriority는 TaskPriority타입으로 변경됨. 비동기 동작은 Task를 기준으로 이루어진다.<br>

실행 작업내용을 담는 main() 메소드는 execute() 라는 custom function으로 바뀐다. 각 ClientOperation 타입의 인스턴스들의 execute는 비동기 호출되어야한다. 업무처리과정은 순서에 맞춰 하나하나가 끝날 때까지 기다리면서 수행하는 것이 아니라, 수행가능한 여러 thread에서 동시다발적으로 수행되어야 하기 때문이다.<br>

## 2-1. Async Await 기본 사용법
메서드 선언부 마지막에 <b>async</b> 를 기입하고,<br>

함수 내용부에선 작업이 끝나기를 기다리는 부분 앞에 <b>await</b> 를 기입한다.

  ```swift
  //ClientOperation.swift
  func execute() async { //Operation 방식 구현 당시의 override func main() 부분이 대체되는 곳
      do {
        print(ConsoleOutput.currentProcess(self, .start).message)
        try await operateBusiness()
        print(ConsoleOutput.currentProcess(self, .done).message)
      } catch {
        switch error {
        case BankOperationError.unknownError:
          print(BankOperationError.unknownError.rawValue)
          break
        default:
          print(BankOperationError.unknownError.rawValue)
          break
        }
      }
    }
  ```

  - 위 코드에서 <b>async</b>는 execute() 메서드가 호출되는 곳에서 비동기처리를 할 것이니, 호출부에서 execute() 하단에 있는 코드가 해당 함수가 끝나기까지 기다리지 않고 코드를 수행할 수 있음을 의미한다. 
  - <b>try await operateBusiness()</b>에서 <b>try</b>는 do-catch문에서 에러발생가능성이 있는 곳을 표기하는 것으로 operateBusiness 실행 시 오류가 발생하면 catch문을 따라가 에러처리를 한다. 
  - <b>try await operateBusiness()</b>에서 <b>await</b>는 메소드의 async와 상응하는 부분이다. 비동기코드가 실행되는 곳으로 operateBusiness라는 비동기 메서드가 수행완료될 때까지 해당위치에서 기다렸다가 하단의 print문을 출력한다. 

  ```swift
  private func operateBusiness() async throws {
      // clientBusiness(ClientOperation의 업무종류) 확인 코드 생략
      switch clientBusiness {
      case .deposit:
        try await handleDepositBusiness() //예금업무 수행
      case .loan:
        try await handleLoanBusiness() // 대출업무 수행
      }
    }
    
    private func handleDepositBusiness() async throws {
      try await Task.sleep(nanoseconds: 7 * 100_000_000) // task 일시정지하는 async 메서드, thread은 non-block상태로 다른 task를 수행할 수 있도록 되어있다.
    }
    
    private func handleLoanBusiness() async throws {
      try await Task.sleep(nanoseconds: 3 * 100_000_000) // task 일시정지하는 async 메서드
      
      switch try headQuarter.handleLoanScreeningQueue(of: self) {
      case true:
        try await Task.sleep(nanoseconds: 3 * 100_000_000) // task 일시정지하는 async 메서드
      case false:
        break
      }
    }
  ```

  - operateBusiness와 handleDeposit/LoanBusiness 또한 비동기처리를 담고 있기에 async를 메소드이름 뒤에 붙이고, 에러처리 또한 execute에서 하고 있기에 에러를 throw할 수 있다. 따라서 <b>async throws</b> 라고 메소드 선언 시 작성하고 <b>try await</b>를 비동기가 발생하는 곳 앞에 작성한다. async가 await와 상응하듯, throws는 try와 상응한다.
  - <b>1. execute > 2. operateBusiness > 3. handleDepositBusiness / handleLoadBusiness</b> 순으로 호출되는데, handleDepositBusiness / handleLoadBusiness 의 Task.sleep이 async 타입메서드이므로 execute와 operateBusiness 또한 async throws를 선언하여 비동기코드가 발생하는 영역이라는 표시를 해줘야한다.
  - 현재는 ClientOperation 내부에서 async throws를 연쇄적으로 작성해줘야하는 부분에 대해 이야기했지만, ClientOperation 외부에서 execute를 실행하는 곳에서도 async throws를 작성해줘야한다.

## 2-2. TaskGroup 활용하기
ClientOperation의 비동기 작업 내용을  async / await를 활용하여 refactor했다면,<br> 다음은 ClientOperation이 동작하는 영역을 설정하는 곳을 refactor 해야한다.<br> 기존에는 Bank에서 OperationQueue를 생성 및 설정하여 queue에 Operation을 추가하면 되었다.<br>

우선 Task와 Task Group은,<br>
<b> Task </b>: 비동기적으로 실행하는 작업의 기본 단위.<br>
<b> Task Group </b>: Task는 부모-자식 관계를 갖는 위계적 구조를 갖는다. Task Group에서 child tasks를 생성할 수 있다. Task Group에서 생성되는 child tasks들은 같은 부모 task를 갖는다. 이러한 task 와 task group의 뚜렷한 관계에 대해 구조적이라 볼 수 있어 <b> structured concurrency </b> 접근 방식으로 불린다.<br>

TaskGroup을 활용하여 refactor한 부분을 살펴보면

  ```swift
  // Bank.swift
  func startWorking(from queue: [ClientOperation]) async throws {
    // 타이머 시작 부분 생략
      await withTaskGroup(of: Void.self) { group in
        for client in queue {
          group.addTask(priority: client.priority) {
            await client.execute()
          }
        }
      }
    //이하 생략
    }
  ```

  - withTaskGroup은 TaskGroup을 생성한다. of: Void.self는 taskgroup이 생성하는 childtasks의 return type.
  - trailing closure부분을 보면 queue의 client를 돌면서 task group에 child task를 추가하고 있다 (addTask). 이 때 추가되는 각 task는 TaskPriority를 설정하여 수행우선순위를 조절할 수 있는데, 이는 clien의 priority (TaskPriority)이다.
  - ChildTask는 바로 client가 업무를 수행하는 비동기코드이다. 즉, client의 priority에 따라 업무수행을 비동기적으로 수행하고 있으며 await withTaskGroup이므로 taskgroup의 동작이 전부 끝날 때까지 기다렸다가 모든 client의 업무가 처리 완료되면 아래의 코드를 수행한다.

TaskGroup에서 생성되는 Child task는 동시적으로 동작하고, TaskGroup은 생성된 모든 child task들이 동작을 끝내면 return 한다.<br> 

따라서 기존 OperationQueue에서 Operation들이 각자 자신의 업무를 수행하고 끝내는 방식을 대체할 수 있다. 간단화하면 operation는 task로, operationqueue는 taskgroup 으로 이동한 것이다.<br>

TaskGroup을 사용할 때, child tasks의 return type이 전부 동일해야한다. 위 코드에서 withTaskGroup에서 return type은 Void로 해당 task group에 추가되는 모든 child tasks는 Void return형을 갖는다.<br>
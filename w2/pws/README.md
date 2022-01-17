# **항목 12: 객체의 모든 부분을 빠짐없이 복사하자**

```cpp
void logCall(const std::string& funcName);
class Customer {
public:
    Customer(const Customer& rhs);
    Customer& operator=(const Customer& rhs);
private:
    std::string name;
}

Customer::Customer(const Customer& rhs)
: name(rhs.name) {
    logCall("Customer copy constructor");
}

Customer& Customer::operator=(const Customer& rhs) {
    logCall("Customer copy assignment operator");
    name = rhs.name;
    return *this;
}
```
위의 코드는 문제가 없으나

```cpp
class Date{...};
class Customer {
public:
    ...
private:
    std::string name;
    Date lastTransaction;
}
```
`Date` 클래스를 추가하면서 `부분 복사(partial copy)`가 된다.

`name` : 복사, `lastTransaction` : 복사안함

**클래스에 데이터 멤버를 추가했으면, 추가한 데이터 멤버를 처리하도록 복사 함수를 다시 작성할 수밖에 없다, (생성자, 비표준형 `operator=` 함수도 모두 바꿔야 한다.)**

```cpp
class PriorityCustomer : public Customer {
public:
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
private:
    int priority;
};
 
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs) 
: priority(rhs.priority) {
    logCall("PriorityCustomer copy constructor");
}
 
PriorityCustomer& PriorityCustomer :: operator=(const PriorityCustomer& rhs) {
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
```
`PriorityCustomer` 클래스에 선언된 데이터 멤버를 모두 복사하지만 `Customer`로부터 상속된 데이터 멤버들은 복사가 안 되고 있다.

`PriorityCustomer`의 복사 생성자에는 기본 클래스 생성자에 넘길 인자들도 명시되어 있지 않아서 `PriorityCustomer` 의 `Customer` 부분은 인자 없이 실행되는 `Customer` 기본 생성자에 의해 초기화된다.

파생클래스에 대한 복사 함수를 만들 경우 기본 클래스 부분을 빠뜨리지 않도록 한다, 기본 클래스 부분은 `private`멤버일 가능성이 높기 때문에 파생 클래스의 복사 함수 안에서 기본 클래스의 (대응되는) 복사 함수를 호출하도록 만든다

```cpp
class PriorityCustomer : public Customer {
public:
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
private:
    int priority;
};
 
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs) 
: Customer(rhs), priority(rhs.priority) {
    logCall("PriorityCustomer copy constructor");
}
 
PriorityCustomer& PriorityCustomer :: operator=(const PriorityCustomer& rhs) {
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs);
    priority = rhs.priority;
    return *this;
}
```

코드양을 줄이기 위해 **복사 대입 연산자에서 복사 생성자를 호출**하거나 **복사 생성자에서 복사 대입 연산자를 호출**하는 것은 불가하니 하지말자

양쪽에서 겹치는 부분을 별도의 멤버 함수에 분리해 놓은 후에 이 함수를 호출하게 만들 수 있다. 보통 `private` 멤버로 두고 `init` 이름을 가지는 경우가 많다.
> 객체 복사 함수는 주어진 객체의 모든 데이터 멤버 및 모든 기본 클래스 부분을 빠뜨리지 않고 복사해야 한다.

> 클래스 복사 함수 두 개를 구현할 때, 한 쪽을 이용해서 다른 쪽을 구현하려는 시도는 절대로 하지 마세요. 그 대신, 공통된 동작을 제 3의 함수에다 분리해 놓고 양쪽에서 이것을 호출하게 만들어서 해결합시다.
# **항목 13: 자원 관리에는 객체가 그만!**
## **문제**
```cpp
class Investment {...};
Investment* createInvestment();

void f() {
    Investment *pInv = createInvestment();
    ...
    delete pInv
}
```
위의 경우 여러 변수가 있다.

1. `...` 부분 도중 `return`
2. 하나의 루프에 같이 있을 때 `continue` 등에 의해 루프문 탈출
3. `...` 안에서 `throw exception`

결과 : 메모리 누출, 객체의 자원 누출

## **해결방안**
자원을 객체에 넣고 그 자원 해제를 **소멸자**가 맡도록 하며, 그 소멸자는 실행 제어가 `f`를 떠날 때 호출되도록 만든다.

`auto_ptr` : 포인터와 비슷하게 동작하는 객체(스마트 포인터)로서, 가리키고 있는 대상에 대해 소멸자가 자동으로 `delete`를 불러주도록 설계되어 있다.

### **`auto_ptr` 사용**
```cpp
void f() {
    // 팩토리 함수 호출, 예전처럼 pInv 사용
    std::auto_ptr<Investment> pInv(createInvestment());
}
// auto_ptr의 소멸자를 통해 pInv를 삭제한다.
```
#### **자원 관리에 객체를 사용하는 방법의 특징**

**1. 자원 획득 후 자원관리 객체에 넘긴다**

`create-Investment`함수가 만들어 준 자원은 그 자원을 관리할 `auto_ptr` 객체를 초기화하는 데 쓰이고 있다.

**2. 자원 관리 객체는 자신의 소멸자를 사용해서 자원이 확실히 해제되도록 한다.**
소멸자는 어떤 객체가 소멸될 때 자동적으로 호출되기 때문에, 실행 제어가 어떤 경위로 블록을 떠나는가에 상관없이 자원 해제가 제대로 이루어지게 된다.

`auto_ptr`은 자신이 소멸될 때 자신이 가리키고 있는 대상에 대해 자동으로 `delete`를 하기 때문에 어떤 객체를 가리키는 `auto_ptr`의 개수가 둘 이상이면 절대로 안된다.

이를 해결하기 위해 `auto_ptr` 객체를 복사하면 원본 객체는 `null`로 만든다. 복사하는 객체만이 그 자원의 유일한 소유권을 갖는다고 가정하기 때문

```cpp
// pInv1이 createInvestment 함수에서 반환된 객체를 가리킴(=OBJ)
std::auto_ptr<Investment> pInv1(createInvestment());
// pInv2는 OBJ를 가리키고, pInv1은 null
std::auto_ptr<Investment> pInv2(pInv1); 
// pInv1은 OBJ를 가리키고 pInv2는 null
pInv1 = pInv2;
```


#### **`auto_ptr`을 사용할 수 없는 상황**
STL 컨테이너의 경우에 원소들이 '정상적인' 복사 동작을 가져야 하기 때문에 `auto_ptr`은 이들의 원소로 허용되지 않는다.

### **참조 카운팅 방식 스마트 포인터 RCSP**

`auto_ptr`을 쓸 수 없는 상황에 대안으로 쓰기 좋다

특정한 어떤 자원을 가리키는(참조하는) 외부 객체의 개수를 유지하고 있다가 그 개수가 0이 되면 해당 자원을 자동으로 삭제하는 스마트 포인터

#### **RCSP 사용**
```cpp
void f() {
    ...
    std::tr1::shared_ptr<Investment> pInv(createInvestment());
    // 팩토리 함수 호출, auto_ptr 처럼 pInv 사용
    ...
}
// shared_ptr의 소멸자를 통해 pInv를 자동으로 삭제
```
#### **`shared_ptr`을 사용한 복사**
```cpp
void f() {
    ...
    // pInv1이 createInvestment 함수에서 반환된 객체를 가리킴(=OBJ)
    std::tr1::shared_ptr<Investment> pInv1(createInvestment());
    // pInv1 및 pInv2가 동시에 OBJ를 가리킴
    std::tr1::shared_ptr<Investment> pInv2(pInv1);
    pInv1 = pInv2;
}
// pInv1, pInv2 가 소멸되며, 이들이 가리키고 있는 객체도 자동으로 삭제
```

#### **주의**
`auto_ptr` 및 `tr1::shared_ptr`은 소멸자 내부에서 `delete` 연산자를 사용한다 (`delete []` 연산자가 아니다)

동적으로 할당한 배열에 대해 `auto_ptr`이나 `tr1::shared_ptr`을 사용하면 안된다.

컴파일 에러가 나지 않으니 주의가 필요
```cpp
std::auto_ptr<std::string> aps(new std::string[10]);
std::tr1::shared_ptr<int> spi(new int[1024]);
// 둘다 잘못된 delete가 사용되면서 문제 발생
```
동적 할당 배열은 `vector`, `string`으로 거의 대체 가능하므로 표준 라이브러리에는 동적 할당 배열을 위한 `auto_ptr`, `tr1::shared_ptr`같은 클래스가 제공되지 않는다.

**자원관리 한다고 깝치지 말고 자원관리 클래스(`tr1::shared_ptr`, `auto_ptr`을 사용하자**
> 자원 누출을 막기 위해, 생성자 안에서 자원을 획득하고 소멸자에서 그것을 해제하는 RAII 객체를 사용합시다

> 일반적으로 널리 쓰이는 RAII 클래스는 `tr1::shared_ptr` 그리고 `auto_ptr`입니다. 이 둘 가운데 `tr1::shared_ptr`이 복사 시의 동작이 직관적이기 때문에 대개 더 좋습니다. 반면, `auto_ptr`은 복사되는 객체(원본 객체)를 `null`로 만들어 버립니다.
# **항목 14: 자원 관리 클래스의 복사 동작에 대해 진지하게 고찰하자**

힙에 생기지 않는 자원은 `auto_ptr` 혹은 `tr1::shared_ptr` 등의 스마트 포인터로 처리해 주기엔 맞지 않다는 것이 일반적인 견해, 이런 경우 자원 관리 클래스를 스스로 만들어야 할 필요를 느낀다

**예)**
```c
// Mutex 타입의 Mutex 객체를 조작하는 C API를 사용한다고 가정
void lock(Mutex *pm);
void unlock(Mutex *pm);
```
```cpp
// 뮤텍스 잠금을 관리하는 클래스를 만든다, RAII 법칙을 따른다
class Lock {
public:
    explicit Lock(Mutex *pm)
    : mutexPtr(pm) {
        lock(mutexPtr);
    }
    ~Lock() {
        unlock(mutexPtr);
    }
private:
    Mutex *mutexPtr;
}
```

```cpp
Mutex m; // 뮤텍스 정의
...
{
    Lock m1(&m); // 뮤텍스 잠금
    ... // 연산 수행
}
// 블록의 끝에서 뮤텍스에 걸린 잠금이 자동으로 풀린다.
```
```cpp
Lock m11(&m);
Lock m12(m11); // Lock 객체가 복사된다면 어떻게 해야 할까?
```

## **선택지**
### **1. 복사를 금지한다.**
스레드 동기화 객체 대한 '사본'이라는 게 거의 의미가 없다. 복사하면 안되는 RAII 클래스에 대해서는 반드시 복사가 되지 않도록 막아야 한다.

`delete` 활용
```cpp
class Lock {
public:
    ...
    Lock(const Lock& m) = delete;
    Lock& operator = (const Lock& m) = delete;
}
```

### **2. 관리하고 있는 자원에 대해 참조 카운팅을 수행한다.**
객체가 소멸될 때까지 자원을 해제하지 않는 게 바람직할 경우, 해당 자원을 참조하는 객체의 개수에 대한 카운트를 증가시키는 식으로 RAII 객체의 복사 동작을 만든다.

`tr1::shared_ptr` 이 사용하고 있다. 

`tr1::shared_ptr`은 참조 카운트가 0이 될 떄 자신이 가리키고 있던 대상을 삭제해 버리도록 기본 동작이 만들어져 있다. 

하지만 `tr1::shared_ptr`이 `삭제자(delete)` 지정을 허용한다. 

**삭제자** : `tr1::shared_ptr`이 유지하는 참조 카운트가 0이 되었을 때 호출되는 함수 혹은 함수객체 (`auto_ptr`의 경우 이 기능이 없어서, `auto_ptr`은 포인터를 바로 삭제한다)

```cpp
class Lock {
public:
    // shared_ptr을 초기화하는데, 가리킬 포인터로 Mutex객체의 
    // 포인터를 사용하고 삭제자로 unlock 함수를 사용한다.
    explicit Lock(Mutex *pm)
    : mutexPtr(pm, unlock) { 
        lock(mutexPtr.get());
    }
private:
    // 원시 포인터 대신에 shared_ptr을 사용한다
    std::tr1::shared_ptr<Mutex> mutexPtr;
}
```
### **3. 관리하고 있는 자원을 진짜로 복사한다.**
자원을 다 썼을 때 각각의 사본을 확실히 해제하는 것이 자원 관리 클래스가 필요한 유일한 명분이 된다.

깊은 복사를 수행해야 한다.
### **4. 관리하고 있는 자원의 소유권을 옮긴다.**
어떤 특정한 자원에 대해 그 자원을 실제로 참조하는 RAII 객체는 딱 하나만 존재하도록 만들고 싶어서, 그 RAII 객체가 복사될 때 그 자원의 소유권을 사본쪽으로 아예 옮겨야 할 경우도 생긴다.

`auto_ptr`의 복사 동작이 그런 경우이다.

---
객체 복사 함수는 컴파일러에 의해 생성될 여지가 있기 떄문에, 컴파일러가 생성한 버전이 원한 바와 맞지 않으면 객체 복사 함수를 직접 만들 수밖에 없다.
> RAII 객체의 복사는 그 객체가 관리하는 자원의 복사 문제를 안고 가기 때문에, 그 자원을 어떻게 복사하느냐에 따라 RAII 객체의 복사 동작이 결정됩니다.

> RAII 클래스에 구현하는 일반적인 복사 동작은 복사를 금지하거나 참조 카운팅을 해 주는 선으로 마무리하는 것입니다. 하지만 이 외의 방법들도 가능하니 참고해 둡시다.

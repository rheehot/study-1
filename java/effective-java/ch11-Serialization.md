# Ch11. Serialization
> Effective Java를 읽으며 공부했던 내용을 정리한다  
> 직렬화 API는 직렬화(객체를 바이트 스트림으로 encoding)하고, 역직렬화(encoding된 바이트 스트림으로부터 객체를 복원)하는 프레임워크를 제공한다  
> 직렬화는 원격 통신을 위한 표준 영속 데이터 포맷 제공하며, `serialization proxy pattern`은 객체 직렬화의 많은 함정을 피하게 해준다  


* [74. Implement Serializable judiciously](#규칙-74-implement-serializable-judiciously)
* [75. Consider using a custom serialized form](#규칙-75-consider-using-a-custom-serialized-form)
* [76. Write readObject methods defensively](#규칙-76-write-readobject-methods-defensively)
* [77. For instance control, perfer enum types to readResolve](#규칙-77-for-instance-control-perfer-enum-types-to-readresolve)
* [78. Consider serialization proxies instead of serialized instances](#규칙-78-consider-serialization-proxies-instead-of-serialized-instances)


## 규칙 74. Implement Serializable judiciously
> Serializable을 분별력 있게 구현하자

* 클래스의 인스턴스가 직렬화 가능하려면 `implements Serializable`을 추가
* 너무 간단해 별로 `할게 없는 것처럼 오해`를 한다
* 직렬화 가능하도록 만드는 비용은 무시 가능하지만, `유지보수 비용은 무시할 수 없다`
* 필드가 외부 API의 일부가 되므로, 정보 은닉의 효력을 상실
* 기본 직렬화 형태를 사용하고 나중에 클래스의 내부 구현을 변경한다면, 변경으로 인해 직렬화 형태가 호환되지 않을 수 있다
* `ObjectOutputStream.putFields()`, `ObjectInputStream.readFields()`를 사용하여 원래 직렬화 형태를 유지하면서 `내부 구현을 변경`할 수 있다
   * 사용 방법이 어렵고, 코드가 지저분헤진다
   * 그러므로 `직렬화 형태를 신중하게 설계`


### Serializable을 구현하는 비용

#### 1. 직렬화를 수반하면서 클래스 진화를 제약하는 예
* serial version UID -> stream unique identifier
* 직렬화 가능한 모든 클래스는 `고유 식별 번호`를 갖는다
* serial version UID를 명시적으로 선언하지 않으면, 시스템에서 `런타임시 자동으로 생성`
   * 클래스의 모든 코드가 영향을 준다
   * UID를 명시적으로 선언하지 않으면 내부 구현 변경시 호환성이 깨지게 된다

#### 2. 결함과 보안상의 허점을 증대시킨다
* 객체는 생성자를 사용해 생성
* serialization은 `언어 영역을 벗어나는 방식`으로 객체를 생성
* `JVM이 자동으로 해주는 기본적인 역직렬화` 또는 `메소드 오버라이딩으로 독자적인 역직렬화`시 감춰진 생성자 사용
* 감춰진 생성자에서 해야할 것
   * 모든 불변 규칙이 지켜지는지 확인
   * 외부 공격자가 접근하지 못하도록 방지
   * 기본 메커니즘에 의존하면 불변성 및 불법적인 접근에 취약

#### 3. 새 버전의 클래스 배포 부담 증가
* 구버전과의 `호환성이 유지`되는지 test
* test 분량 - `직렬화 가능 클래스 수 x 배포판 수`
* 이진 호환성 + 의미적 호환성도 test해야해서 자동화할 수 없다



### Serializable의 구현은 쉽게 결정할 일이 아니다
* 직렬화는 실질적인 이점을 제공
* `객체 전송, 영속성 직렬화`에 의존하는 프레임워크와 관계되는 클래스 -> 필수
* `Value Class(Date, BigInteger)` -> 필수
* thread pool과 같은 `활동적인 개체를 나타내는 클래스` -> 구현 필요 X
* 상속을 위해 설계된 class, interface -> 구현(implements, extends) 필요 X
   * class를 상속받거나 interface를 구현하려는 누군가에게 부담이 된다
   * 모든 class, interface가 Serialiable를 구현할 경우에는 괜찮다



### Serializable를 구현하면서 상속을 위해 설계된 클래스
* `Throwable`
   * RMI(remote method invocation)시 발생된 exception을 서버로부터 클라이언트에게 전달해야 하기 때문
* `Component`
   * GUI compoment들이 전송, 저장, 복구될 수 있어야 하기 때문
* `HttpServlet`
   * session status를 caching할 수 있어야 하기 때문


### 직렬화 가능하고 확장 가능하면서 인스턴스 필드를 갖는 Class를 만들 때 주의할 점
* 인스턴스 필드들이 `default 값으로 초기화`되는 경우(정수 0, boolean false, 객체 참조 null) 불변 규칙이 깨진다면, `readObjectNoData()`를 반드시 추가
```java
private void readObjectNoData() throws InvalidObjectException {
    throw new InvalidObjectException("Stream data required");
}
```


### Class에서 Serialiable을 구현하지 않는다는 결정을 할 경우 주의사항
* 상속을 위해 설계된 클래스 `직렬화 불가능` -> 서브 클래스 `직렬화 불가능`
* 특히, 접근 가능한 default 생성자를 수퍼 클래스가 제공하지 않는다면 불가능
   * 상속을 위해 설계된 대부분의 클래스들은 상태를 갖지 않으므로, 생성자 추가는 어렵지 않다
* 불변 규칙이 이미 확립된 객체를 생성하는 것이 가장 좋은 방법
* 클라이언트가 제공하는 데이터가 불변 규칙을 확립하는데 필요하다면 default 생성자의 사용은 배제된다
* 다른 생성자가 default 생성자와 초기화 메소드를 추가하면, 클래스의 상태만 복잡하게 만들게 된다


#### 자신은 직렬화 불가능하지만, 직렬화 가능한 서브 클래스를 허용하는 클래스에 매개변수 없는 생성자를 추가하는 방법
```java
// 생성자가 1개 존재할 경우
public AbstractFoo(int x, int y) {
    ...
}

// 자신은 직렬화 불가능하지만, 직렬화 가능한 서브 클래스를 허용하는 클래스
public abstract class AbstractFoo {
    private int x, y;  // 인스턴스 상태

    // 초기화 추적에 사용
    private enum State { NEW, INITIALIZING, INITIALIZED };
    private final AtomicReference<State> init = new AtomicReference<>(State.NEW);

    public AbstractFoo(int x, int y) {
        initialize(x, y);
    }

    // readObject()에서 인스턴스의 상태를 초기화
    protected AbstractFoo(){}
    protected final void initialize(int x, int y) {
        if(!init.compareAndSet(State.NEW, State.INITIALIZING)) {
            throw new IllegalStateException("Already initialized");
        }
        this.x = x;
        this.y = y;
        ... // 원래 생성자의 일 수행
        init.set(State.INITIALIZED);
    }

    // 내부 상태로의 접근 제공 -> 서브 클래스의 writeObject()로 직렬화 가능
    protected final int getX() {
        checkInit();
        return x;
    }

    protected final int getY() {
        checkInit();
        return y;
    }

    // 모든 public, protected 인스턴스 메소드에서 호출해야 한다
    private void checkInit() {
        if(init.get() != State.INITIALIZED) {
            throw new IllegalStateException("Uninitialized");
        }
    }
    ...
}

// 상태가 있고 직렬 불가능한 클래스의 직렬화 가능한 서브 클래스
public class Foo extends AbstractFoo implements Serializable {
    private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject();

        // 수퍼 클래스의 상태를 직접 역직렬화하고 초기화
        int x = s.readInt();
        int y = s.readInt();
        initialize(x, y);
    }

    private void writeObject(ObjectOutputStream s) throws IOException {
        s.defaultWriteObject();

        // 수퍼 클래스의 상태를 직접 직렬화
        s.writeInt(getX());
        s.writeInt(getY());
    }

    // 직렬화 메커니즘에서 사용 X
    public Foo(int x, int y) {
        super(x, y);
    }

    private static final long serialVersionUID = 129329324343L;
}
```

### inner class에서는 Serialiable을 구현 X
* inner class는 컴파일러가 생성한 `syntheic field`를 사용
   * enclosing instance에 대한 `참조`와 유효범위에 있는 `지역 변수들의 값` 저장
* 익명의 내부 클래스나 지역 내부 클래스처럼 필드들이 클래스와 어떻게 대응되는지 명시 X
   * 내부 클래스의 기본 직렬화 형태는 정의가 불분명
* static 멤버 클래스는 Serialiable을 구현 가능


### 정리
* Serialiable interface의 구현은 쉽지 않다
* 상속을 위한 클래스는 특별히 주의 필요
   * 서브 클래스에서 직렬화 가능하게, 불가능하게 하는 것을 절충한 설계 관점 필요
   * 접근 가능한 default 생성자를 제공 -> 서브 클래스에서 Serialiable 구현 가능
   * 서브 클래스를 직렬화-역직렬화시 상위의 수퍼 클래스부터 차례로 default 생성자 호출(서브 클래스를 완벽하게 초기화하기 위함)



## 규칙 75. Consider using a custom serialized form
> 독자적인 직렬화 형태의 사용을 고려하자

* 시간의 압박을 받으면서 클래스를 만들 경우 `API를 가장 좋게 설계`하는데 집중
* `일정 기간 쓰다 버릴` 구현체가 Serializable를 구현하고 `기본 직렬화 형태`를 사용한다면 버릴 수 없고 형태를 유지해야 한다
* `유연성`, `성능`, `정확성` 관점에 타당할 경우 기본 직렬화 형태 수용
* 객체의 기본 직렬화 형태
   * 그 객체를 뿌리로 하는 객체 그래프를 물리적으로 표현한 효율적인 인코딩
   * 객체가 닿을 수 있는 모든 객체에 포함된 데이터
* 이상적인 객체 직렬화
   * 객체가 표현하는 `논리적 데이터만` 포함(물리적 표현과는 독립)


### 객체의 물리적 표현이 논리적 표현과 동일한 경우
* 기본 직렬화 형태에 적합
* 기본 직렬화 형태가 적합하더라도 `불변 규칙의 준수와 보안을 위해 readObject()` 제공
* private 필드가 직렬화로 인해 `public API에 속하므로 문서화` 필수
```java
// 기본 직렬화에 적합한 클래스 - 사람의 이름을 표현
public class Name implements Serialiable {

    /**
     * 성, non null
     * @serial 
     */
    private final String lastName;

    /**
     * 이름, non null
     * @serial 
     */
    private final String firstName;

    /**
     * 중간 이름 nullable
     * @serial 
     */
    private final String middleName;
}
```


### 물리적 표현이 논리적 데이터와 다른 경우
```java
// 엄청난 정보를 직렬화
// 문자열 순차, 물리적으로는 이중 링크 리스트로 순차를 나타낸다
public final class StringList implements Serialiable {
    private int size = 0;
    private Entry head = null;

    private static class Entry implements Serialiable {
        String data;
        Entry next;
        Entry previous;
    }
}
```

#### 단점
* 외부 API가 현재의 내부 구현에 영원히 얽매이게 된다
   * StringList class는 영훤히 링크 리스트 형태로 처리해야 한다
* 과도한 저장 공간을 차지할 수 있다
   * 불필요한 링크 리스트 구조를 직렬화에 포함
* 시간이 오래 걸린다
   * 모든 객체 그래프를 훓어야 하므로
* stack overflow 발생
   * 모든 객체 그래프를 순환하므로

```java
// 독자적인 직렬화 사용 -> writeObject(), readObject()로 논리적인 직렬화 구현
public final class StringList implements Serialiable {
    // transient -> 기본 직렬화에서 제외
    private transient int size = 0;
    private transient Entry head = null;

    // 물리적인 표현은 직렬화 X
    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }

    public final void add(String s) {}

    /**
     * 인스턴스를 직렬화
     * @serialData 리스트의 크기는 ({@code int})로 나오며,
     * 그 다음에 리스트의 모든 요소들이 (각각 {@code String} 타입) 순서대로 나온다 
     */
    private void writeObject(ObjectOutputStream s) throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);

        // 모든 요소를 byte stream으로 출력하여 직렬화
        for(Entry e = head; e != null; e = e.next) {
            s.writeObject(e.data);
        }
    }

    private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        int numElements = s.readInt();

        // 직렬화된 모든 요소를 읽어 리스트에 추가
        for(int i=0; i<numElements; i++) {
            add((String) s.readObject());
        }
    }
    ...
}
```
* 모든 인스턴스 필드가 transient일 경우, defaultReadObject(), defaultWriteObject() 사용 
   * 직렬화 형태에 영향을 주어 유연성이 좋아진다
   * 상위 버전에서 직렬화 -> 하위 버전에서 역직렬화시 
      * 상위 버전에서 추가된 필드 무시
* `writeObject()`는 직렬화된 public API를 정의하므로 문서화
* 직렬화, 역직렬화 과정에서 불변 규칙이 완전한 복사본이 만들어지므로 정확


### 객체의 불변 규칙이 내부 구현에 얽매이는 경우
* hash table
   * 물리적인 표현 -> key, value를 포함하는 hash bucket의 sequence
   * 각 항목이 들어있는 bucket은 key의 hashCode로 결정, JVM에 따라 달라진다
   * 직렬화, 역직렬화시 불변 규칙 손상


### transient   
* transient가 지정되지 않은 필드는 `defaultWriteObject()` 호출시 직렬화
   * 역직렬화시 default value로 초기화
   * `readObject()`에서 `defaultReadObject()를 호출하고, 초기화`하거나 `lazy initialization`
* transient로 해야할 필드는 transient로 반드시 지정
   * caching된 hash 값처럼 주 데이터 필드들로부터 계산될 수 있는 필드 등
* 필드의 값이 논리적인 상태인지 확인


### 객체 전체를 읽는 메소드는 직렬화에 따른 동기화 필요
```java
// 모든 메소드를 동기화하는 thread safe 객체가 있고, 기본 직렬화를 사용한다면 사용
private synchronized void writeObject(ObjectOutputStream s) throws IOException {
    s.defaultWriteObject();
}
```
* 동기화 가능한 class에는 명시적 serialVersionUID 선언
   * 버전간의 `비호환성 제거`
   * 런타임시 생성하지 않아 `성능상 이점`
   * 구버전과 호환되지 않는 버전을 만들 경우 UID를 수정
      * 직렬화시 InvalidClassException 발생


### 정리
* 어떤 직렬화 형태를 사용해야 하는지 신중하게 결정
* 기본 직렬화 사용
   * 물리적 표현이 논리적 데이터와 같을 경우
* 독자적 직렬화 사용
   * 물리적 표현이 논리적 데이터와 다른 경우
* 직렬화 형태는 시간을 들여 설계
   * 호환성 유지를 위해 쉽게 제거할 수 없다
   * 잘못 선택하면, class 복잡도와 성능에 영구적으로 영향을 끼친다



## 규칙 76. Write readObject methods defensively
> 방어 가능한 readObject()를 작성하자

```java
// 불변성을 위해 생성자와 getter에서 Date를 defensive copy
public class Period {

    private final Date start;
    private final Date end;

    /**
     * @param start 시작일
     * @param end 종료일(시작일보다 작다)
     * @throws IllegalArgumentException 시작일이 종료일보다 늦으면 발생
     * @throws NullPointerException 시작일이나 종료일이 null이면 발생
     */
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if(start.compareTo(end) > 0){
            throw new IllegalArgumentException(start + " after " + end);
        }
    }

    public Date getStart() {
        return new Date(start.getTime());
    }

    public Date getEnd() {
        return new Date(end.getTime());
    }

    @Override
    public String toString() {
        return start + " - " + end;
    }
}
```
* 물리적인 표현이 논리적인 데이터 내용과 일치 -> 기본 직렬화 형태 사용
* implements Serialiable을 추가하는 순간 불변성 보장 X
* `readObject()`는 또 다른 public 생성자
   * 유효성 검사 필요
   * defensive copy 필요
   * 정상적으로 만들어진 인스턴스를 직렬화하여 생성된 `Byte Stream`을 인자로 받는 생성자


### 잘못된 byte stream을 역직렬화 방지
```java
public class BogusPeriod {
    // 잘못된 byte stream
    private static final byte[] serializedForm = new byte[] {
        ...
    };

    public static void main(String[] args) {
        Period p = (Period) deserialize(serializedForm);
        System.out.println(p);
    }

    // 지정된 직렬화 객체 반환 - 유효성 검사가 없어 잘못된 객체 역직렬화
    private static Object deserialize(byte[] sf) {
        try {
            InputStream is = new ByteArrayInputStream(sf);
            ObjectInputStream ois = new ObjectInputStream(is);
            return ois.readObject();
        } catch(Exception e) {
            throw new IllegalArgumentException(e);
        }
    }
}

// after - 유효성 검사 추가
private void readObject(ObjectInputStream s) throw IOException, ClassNotFoundException {
    s.defaultReadObject();

    // 불변 규칙 검사
    if(start.compareTo(end) > 0)
        throw new InvalidObjectException(start + " after " + end);
}
```

### byte stream을 날조한 객체 생성 방지
```java
public class MutablePeriod {

    public final Period period;

    public final Date start;

    public final Date end;

    public MutablePeriod() {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);

            // 적법한 Period 인스턴스 직렬화
            oos.writeObject(new Period(new Date(), new Date()));

            byte[] ref = {0x71, 0, 0x7e, 0, 5};
            bos.write(ref);  // 시작일자 필드
            ref[4] = 4;
            bos.write(ref);  //  종료일자 필드

            // Period 인스턴스 역직렬화 후 Date 참조를 가져온다
            ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
            period = (Period) ois.readObject();
            start = (Date) ois.readObject();
            end = (Date) ois.readObject();
        } catch (Exception e) {
            throw new AssertionError(e);
        }
    }

    // 공격 코드
    public static void main(String[] args) {
        MutablePeriod mp = new MutablePeriod();
        Period p = mp.period;
        Date pEnd = mp.end;

        // 임의의 년도 조절 가능
        pEnd.setYear(78);
        System.out.println(p);
    }
}
```
* 불변 규칙을 지키면서 생성되었지만, 내부 컴포넌트를 마음대로 변경 가능
   * readObject()가 defensive copy를 하지 않아서


#### 개선
* 객체 역직렬화 시, 클라이언트가 소유하면 안되는 객체 참조를 포함하는 `모든 필드를 defensive copy`하자
```java
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();

    // defensive copy -> final은 불가능
    start = new Date(start.getTime());
    end = new Date(start.getTime());

    if(start.compareTo(end) > 0)
        throw new InvalidObjectException(start + " after " + end);
}
```


### readObject()가 적합한지 검증하는 리트머스 테스트
* 인자를 받는 public 생성자 존재, 인자에 대한 검증을 하지 않고, transient가 아닌 필드에 그대로 저장해도 아무런 문제가 없는가?
* 있다면?
   * `readObject()`를 제공해 모든 유효성 검사 수행 및 defensive copy
   * `serialization proxy pattern` 사용


### 정리
* `readObject()` 구현시 public 생성자(byte stream으로 적법한 인스턴스를 생성하는)를 만든다고 생각
* 역직렬화시 주어지는 byte stream이 올바르다고 생각하지 말자
* `readObject()` 작성 지침
   * 외부에 공개되지 않아야 하는 객체 참조 필드를 갖는 클래스의 경우
      * 객체 참조 필드를 `defensive copy`
      * 불변 클래스에 포함된 가변 컴포넌트가 해당
   * defensive copy 전에 불변 규칙을 모두 검사
      * 검사 실패시 InvalidObjectException 던지자
   * 역직렬화 후 객체의 전체 객체 그래프를 검사해야 한다면 `ObjectInputValidation` 사용
   * 직,간접적으로 오버라이딩 가능한 메소드 호출 X
      * 역직렬화 전에 오버라이딩한 메소드가 호출되어 프로그램 실행이 실패된다



## 규칙 77. For instance control, perfer enum types to readResolve
> 인스턴스 제어에는 readResolve()보다 enum을 사용하자

### Singleton 직렬화
```java
// Singleton
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {}

    public void leaveTheBuilding() {}

    // 인스턴스 제어
    private Object readResolve() {
        // 진짜 객체 반환, 가짜는 GC에게..
        return INSTANCE;
    }
}
```
* `implements Serializable` 시 더이상 Singleton이 아니다
* `readObject()`는 항상 새로운 인스턴스를 반환
   * 클래스 초기화시 생성된 인스턴스와는 다르다
* `readResolve()`
   * readObject()에서 생성한 인스턴스를 다른 인스턴스로 바꾼다
   * 역직렬화된 후 새롭게 생성된 객체에 대해 자동 호출되어 메소드가 반환하는 객체가 새로운 객체 대신 반환된다
   * 역직렬화된 객체의 참조는 유지되지 않으므로 GC 대상이 된다 
   

### 모든 인스턴스 필드는 transient로 선언
* `readResolve()`에서 `역직렬화된 객체를 무시`하므로 직렬화된 형태는 실제 데이터가 포함될 필요가 없다
* `readResolve()` 호출 전 객체 참조를 훔칠 수 있다
   * `transient`가 아닌 객체 참조 필드를 포함 -> readResolve()가 호출되기 전 필드 역직렬화
   * 역질력화되는 시점에 역직렬화된 원래의 싱글톤 참조를 정교하게 만든 스트림이 `훔칠` 수 있다

```java
// 문제있는 Singleton - transient가 아닌 객체 참조 필드 존재
public class Elvis implements Serializable {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {}

    private String[] favoriteSonds = {"Hound Dog", "Heartbreak Hotel"};
    
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }

    private Object readResolve() throws ObjectStreamException {
        return INSTANCE;
    }
}

// stealer class
public class ElvisStealer implements Serializable {
    static Elvis impersonator;
    private Elvis payload;

    private Object readResolve() {
        // unresolved Elvis 인스턴스에 참조를 저장
        impersonator = payload;

        // 해당 필드에 적합한 타입의 객체 반환
        return new String[] {"A Fool Such as I"};
    }
    private static final long serialVersionUID = 0;
}

// Singleton에서 다른 인스턴스가 생성됨을 증명하는 클래스
public class ElvisImpersonator {
    // 조작된 byte stream
    private static final byte[] serializedForm = new byte[] {
        ...
    };

    public static void main(String[] args) {
        // ElvisStealer.impersonator를 초기화, 진짜 Elvis 반환
        Elvis elvis = (Elvis)deserialize(serializedForm);
        Elvis impersonator = ElvisStealer.impersonator;

        elvis.printFavorites();  // [Hound Dog, Heartbreak Hotel]
        impersonator.printFavorites();  // [A Fool Such as I]
    }
}
```
* 필드를 `transient`로 선언하여 해결할 수 있다
   * `Enum Singleton`이 더 좋은 방법


### Enum Singleton
* Java 1.5 기준으로 직렬화 가능한 모든 인스턴스 제어 클래스에 `readResolve()` 사용
   * 허술하고 많은 주의 필요 
* enum으로 선언하면, `enum 상수 외에 어떤 인스턴스도 생성될 수 없음`이 확실히 보장
   * JVM이 보장하므로 의존할 수 있다
   * 사용자 입장에서도 특별히 주의할 것이 없다

```java
// Enum Singleton
public enum Elvis {
    INSTANCE;

    private String[] favoriteSonds = {"Hound Dog", "Heartbreak Hotel"};
    
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
}
```
* 컴파일 시점에 알려지지 않은 클래스는 `enum`을 사용할 수 없으므로 `readResolve()` 제공

### readResolve()의 접근성 중요
* final class에 둘 때는 반드시 private
* 아니라면 신중하게 고려하여 선택
* protected 또는 public 이면서 서브 클래스가 오버라이딩 하지 않는다면 역직렬화시 `ClassCastException` 발생시킬 수 있는 수퍼 클래스 인스턴스를 생성한다


### 정리
* 인스턴스 제어에 관련된 불변 규칙이 있으면 `enum` 사용
* enum을 사용할 수 없고, 직렬화 가능하면서 인스턴스 제어도 필요하다면 `readResolve()` 제공
   * 객체 참조 필드는 `transient`로 선언



## 규칙 78. Consider serialization proxies instead of serialized instances
> 직렬화된 인스턴스 대신 직렬화 프록시의 사용을 고려하자

* Serialiable interface 구현시 결함과 보안 문제 발생 가능성이 크다
* 언어 영역 밖의 메커니즘을 사용해서 인스턴스가 생성되기 때문
* `serialization proxy pattern`을 사용하여 위험을 현저하게 줄일 수 있다


### serialization proxy pattern
* 직렬화 가능한 클래스의 private static nested class(enclosing class의 내부 상태를 나타내는) 설계 -> `serialization proxy`
* enclosing class를 매개변수로 하는 `단일 생성자` 존재
   * 인자로부터 데이터만 복사
   * 일관성 검사, defensive copy X
* 기본 직렬화 형태 -> enclosing class의 완벽한 직렬화 형태
* enclosing class, serialization proxy 모두 `implements Serialiable`을 선언

```java
public class Period {

    private final Date start;
    private final Date end;

    /**
     * @param start 시작일
     * @param end   종료일(시작일보다 작다)
     * @throws IllegalArgumentException 시작일이 종료일보다 늦으면 발생
     * @throws NullPointerException     시작일이나 종료일이 null이면 발생
     */
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (start.compareTo(end) > 0) {
            throw new IllegalArgumentException(start + " after " + end);
        }
    }

    public Date getStart() {
        return new Date(start.getTime());
    }

    public Date getEnd() {
        return new Date(end.getTime());
    }

    @Override
    public String toString() {
        return start + " - " + end;
    }

    // Period class의 serialization proxy
    private static class SerializationProxy implements Serializable {

        private static final long serialVersionUID = -7810344396436087362L;

        private final Date start;

        private final Date end;

        SerializationProxy(Period p) {
            this.start = p.start;
            this.end = p.end;
        }
    }

    private Object writeReplace() {
        return new SerializationProxy(this);
    }

    private Object readObject() throws InvalidObjectException {
//        throw new InvalidObjectException("Proxy required");
        return new Period(start, end);  // public 생성자 사용
    }
}
```
* `writeReplace()`
   * 직렬화에 앞서, `enclosing class -> serialization proxy` 변환
   * 직렬화 메커니즘에 의해 SerializationProxy를 직렬화
      * enclosing class를 생성하지 않는다
* `readObject()`
   * 역직렬화시, `serialization proxy -> enclosing class` 변환
   * `enclosing class의 생성자`를 사용해 인스턴스 생성
      * 직렬화시 언어 영역 밖의 특성 배제(장점)
      * 생성자가 불변 규칙을 확립하면 직렬화에도 유지
      * byte stream 위조(규칙 76의 BogusPeriod), 내부 필드 훔치기(규칙 76의 MutablePeriod)를 즉각 퇴치
      * enclosing class의 필드 final 가능


### EnumSet의 serialization proxy
* `serialization proxy pattern`이 defensive copy보다 더욱 강력하게 되는 방법
* 역직렬화된 인스턴스가 원래 직렬화된 인스턴스의 클래스와 다른 클래스를 가질 수 있다
* ex. EnumSet
   * public 생성자가 없고, static factory method만 존재
   * 기반이 되는 enum type의 크기에 따라 둘 중 하나의 서브 클래스 반환
      * 64개 이하면 RegularEnumSet, 아니면 JumboEnumSet 반환
   * 60개의 요소를 갖는 enum을 직렬화 후 5개를 추가하고 역직렬화 한다면?
      * RegularEnumSet(역직렬화 전) -> JumboEnumSet(역직렬화 후)

```java
// EnumSet의 serialization proxy
private static class SerializationProxy<E extends Enum<E>> implements Serializable {

    private static final long serialVersionUID = -9114127096552720951L;
    
    private final Class<E> elementType;

    private final Enum[] elements;

    SerializationProxy(EnumSet<E> set) {
        elementType = set.elementType;
        elements = set.toArray(EMPTY_ENUM_ARRAY);
    }

    private Object readResolve() {
        EnumSet<E> result = EnumSet.noneOf(elementType);
        for (Enum e : elements) {
            result.add((E) e);
        }
        return result;
    }
}
```

### serialization proxy pattern의 2가지 제약
* `client가 서브 클래스를 만들 수 있는 클래스`와는 호환 X
* 자신의 객체 그래프가 `순환 관계를 맺는 클래스` 호환 X
   * serialization proxy의 readResolve()에서 그 객체의 메소드 호출시 ClassCastException 발생
   * serialization proxy만 생겼을 뿐 객체는 생성되지 않아서


### 정리
* client가 서브 클래스를 만들 수 없는 클래스에 readObject(), writeObject()를 작성해야 할 경우 `serialization proxy pattern` 사용 고려
* 까다로운 불변 규칙을 갖는 객체를 `직렬화 하는 가장 쉬운 방법`이다


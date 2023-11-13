상황:
프로젝트 내에 ServiceExecutor를 수정해보기로 했다.

분석:
ServiceExecutor의 Outline은 다음과 같다
- 필드
	- 로그인 정보 테이블에 대한 칼럼명 맵핑
	- RowType 초기값
- Autowired
	- PasswordEncoder
	- EgovPropertyServiceImpl
	- COMMON_Mapper
- 메서드
	- getCommon()  -> HandlerMethodArgumentResolver
	- ExceptionLogging() -> HandlerExceptionResolver
	- getDsRowToMap() -> 별도의 Class


### getCommon()
해결책 1:
<u>HandlerMethodArgumentResolver</u>
의 경우 3.1.버전 이전에는 WebArgumentResolver로 불리던 확장 포인트 입니다.
내부에는 두개의 메서드인
- supportParameter
- resolveArgument
	- 반환할 MethodArgument를 생성하는 부분
	- webRequest.getNativeRequest/Response()를 통해 일반적으로 사용할 HttpServeletRequest/Respose를 불러올 수 있다.

+이 방식의 경우 Map은 타입으로 지정할 수 없어 CommonMap이라는 새로운 객체를 만들고 내부에 Map을 형성한 후 모든 메서드가 내부에 Map을 바로 향할 수 있도록 구현 했다. 기본 제공되는 객체와 똑같은 기능을 하는 객체이지만 별도의 타입이 필요할 때 사용할 수 있는 방법이다.

문제점 1:
getCommon을 HandlerMethodArgumentResolver를 Implements 받아서 CommonArgumentResolver를 구현해 보았다. 그러나 문제점이 있었는데 넥사크로에 이미 구현해 놓은 NexacroMethodArgumentResolver가 문제였다. ArgumentResolver 사이의 관계는 상하관계가 없는 형제(?) 관계이므로 값의 전달이 불가능 하다는 문제가 있었다. 그러다보니 getCommon 내부에서 필요한 파라미터 값을 NexacroMethodArgumentResolver에서 받아올 방법이 없었다.

해결책 2:
이를 해결하기 위해 CommonArgumentResolver에서 NexacroMethodArgumentResolver와는 별도로 프론트에서 받아온 XML 포맷의 Request의 Body를 받아서 Parsing 할 생각을 했다.

문제점 2: 
이러한 방법은 또 다른 문제점이 있었다. requeset.getInputStream()의 경우 한번 밖에 호출이 불가능다보니 CommonArgumentResolver에서 호출할 때는 Steam Closed라는 경고를 발생 시켰다.

해결책 3:
Stream을 어러번 할 수 있게 Filter 단계에서 별도로 request의 InputStream을 캐시에 저장해서 여러번 사용 할 수 있게 하는 방법을 시도해봤다.

문제점 3:
근본적으로 넥사크로에서 제공되는 코드와 어떤 충돌이 발생할지 몰라 조심스러운게 많아지는 부분이어서 시도하다가 중단하게 되었다.

<u>AOP</u>

해결책 4:
또 다른 방법은 Controller 단에서 CommonArgumentResolver를 포기하는 방법을 생각해 볼 수 있었다. 대신 기존의 Executor 방식을 큰 틀에서는 벗어나지 않지만 extends를 하지 않아 상호간의 종속성을 줄이면서도 중복코드를 줄일 수 있는 AOP를 사용해보려고 한다. ServiceImp 클래스의 메서드 Before 단계에서 getCommon() 메서드를 코드를 실행하여 Map 형식의 변수를 전달하는 방법을 사용하보려고 한다.

문제점 5:
AOP의 경우 ServiceImp 내부 메서드가 실행되기 이전에 getCommon과 같은 내용의 코드가 실행되도록 해주는 방식을 사용해보고자 했다. 그러나 이 방식은 단순히 중복된 코드를 줄여준다는 사실 외에는 구조적인 변화로 인한 이점은 없어 보였다. 
그러던 중 Executor 내부의 mCommon의로 관리되는 로그인 사용자 정보에 구조와 방식에 대해서 고민해보게 됬었다. 크게는 두가지 문제가 있었다. 
첫째는 스프링을 사용하지만 mCommon 변수는 스프링 빈에서 관리가 되지 못하고 있다는 사실이었다. 그렇다 그저 Java Static으로 관리 되어지고 있었다. 동시성에 최악의 방식인 순수 싱글톤 방식이었던 것이다. ServiceImp에 상속되어져서 @Service로 인해 빈에 등록되는 줄 알았지만 ServiceImp의 부모가 Executor가 되는 구조이기에 서로는 상관이 없을 수 밖에 없었다.
두번째는 고민이었는데 과연 로그인 정보를 모든 비즈니스 로직이 실행되기 전에 매번 DB에서 새롭게 불러와야되는가 였다. 결론은 그럴 필요가 없다는 것이었다. 항상 최신의 정보를 기준으로 트랜젝션이 이뤄져야 된다고 처음에 생각했지만 그보다는 사용자가 스스로 인지하고 있는 미리 받아온 정보로 작업이 이뤄져야 된다는 것이 더 직관적인 로직이라는 생각이 들었다.

해결책 5: 
이에 대한 해결책으로 사용할 수 있는 것은 *토비의 스프링 3.1* 에서 알게 된 SessionScope 방식의 빈이었다. 빈이기 때문에 스프링이 관리하게 해줄 수 있다는 부분과 처음에 로그인 할때 정보를 받아온 후 빈에 저장하는 방식이 될 것이라는 두가지 이유는 위의 두가지 문제점을 해결할 방법으로 더 좋은 방식이라고 판단 되었다. 처음에 로그인 할 때, 한번만 받아오기 때문에 InputStream으로 새롭게 정보를 받아올 필요도 없었다.
SessionScope를 사용하려고 할 때 걱정 된 부분이 있다면 mCommon이 Map타입이므로 새로운 객체를 구현하는 것이 어렵다고 느껴졌었다. 그러나 해결책 1에서 사용한 CommonMap을 구현하는 방식으로 해결할  수 있었다.

+++
- CommonMap 클라스를 만들고, @Component, @Scope("session") Annotation을 붙여서 세션 스코프 빈 객체를 생성하였다.
-  로그인 할때, 한번만 불러올 것을 목표로 하기 때문에 getCommon() 메서드를 initCommonMap()으로 변경하였다. 
- private 메서드가 하나도 없는 정말.... 스파게티 같은 코드였다... 우선 private 메서드로 코드를 일부 분리 하여 조금이나마 코드를 닦아(?) 보았다.
- CommonMap을 역정렬화 할 수 없다는 오류가 나왔다. 
   최신 스프링 버전에서 일부 코드가 수정되면서 나타나는 오류인 듯 한데 implements Serializable을 추가하여 해결 하였다
- Autowired 객체를 바로 생성자에서 사용하려고 불러드리니 NullPointExeception이 나왔다. 
  생성자가 Autowired 보다 먼저 일어나 Autowired 된 객체를 참조할 수 없어 나는 오류였다. Autowired 된 객체를 InitCommonMap() 메서드 하위에서 참조하도록 변경 하였다.
- 

### ExceptionLogging()
<u>HandlerExceptionResolver</u>
문제점 1:
Exception을 ServiceImpl 내부 비즈니스 메서드에서 try-catch 로 에러를 잡은 후, catch문 하위에서 ExecptionLogging() 
해결책 1:


### getDsRowToMap()

해결책 1: 
본래는 DataSet의 하위 메서드로 구현하여 DataSet을 자연스럽게 가공하는 형태의 메서드로 구현할 생각을 했습니다.

문제점 1:
DataSet의 경우 넥사크로에서 제공되는 Class 파일에 캡슐화 되어있어 상속받아서 구현할 방법을 찾을 수 었었습니다.

DataSet을 상속하여 구현할 수 있는지 확인!

해결책 2:
별도의 클래스를 생성하여 ConvertDataSetRowToMap(DataSet, Int)  메서를 구현합니다. 이떄,
Int 해당 Row의 값입니다.

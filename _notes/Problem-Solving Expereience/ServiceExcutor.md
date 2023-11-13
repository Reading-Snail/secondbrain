2023-10-22

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

### ExceptionLogging()
<u>HandlerExceptionResolver</u>
해결책1: 처음에는 간단해 보였다. Exception 관련 사항을 별도의 클라스로 분리하면 되는 문제 일줄 알았다. 그래서 해당 메서드에 집중하기 보다는. HandlerExceptionResolver와 같은 스프링 산하의 여러 예외처리 기능들에 집중해서 해결방법을 찾았었다.

문제점1: 그러나 문제는 다른 곳이 있었다.  ExceptionLogging() 메서드가 NexcroResult 객체에 종속적이라는 부분이 제일 큰 문제점이었다. 예외처리는 본디 예외를 만나는 순간 바로 ExceptionHandler를 거쳐 화면단에 UI 또는 log로 표시는 해줘야되기 마련인데 NexacroResult를 return으로 꼭 반환해야되다보니 해결할 방법이 없어보였다.

해결책2: 막막해보였으나 현재(2023-11-08) 드디어 대략적인 방향성을 발견했다. 처음에는 이 부분을 해결할 방법을 알지 못해 덮어두고 있었는데 설계자체의 근본적인 문제임을 발견하게 되었다. CleanCode를 보면 예외처리는 비즈니스코드와 분리되어야 한다고 강조한다. 하지만 이 코드들을 NexacroException을 통해 예외처리를 하는 것이 아닌 비즈니스코드에 사용되는 NexacroResult의 부분 메서드인 setErrMsg(), SetErrCode() 메서드를 사용하므로서 이를 우회적으로 실현하고 있었다. 그래서 예외처리 로직이 비즈니스 로직의 일부인 NexacroResult 객체에 종속적일 수 밖에 없었던 것이다.

해결책1: 예외처리 로직에서 NexacroResult를 제거하고  NexacroException 객체로 처리할 수 있게 변경한다. 또한  Exceptionlogging() 내부적으로 이루어지고 있던 예외종류 분류와 로그처리는 @ExceptionHandler 또는 @ControllerAdvice 처리할 수 있게 변경 할 수 있을 것 같다.

문제점1: @ExceptionHandler를 아무리 지지고 볶아보아도 잘 작동이 되지 않았다. 로그에는 자꾸 NexaceoExecptioinMappingReoslver 클라스와 관련된 에러가 계속 발생되었다. 다행이도 github에 해당 소스가 올라와 있어서 확인해보니 AbstractExctpionHandlerResolver를 상속하는 예외처리 클라스였다. 예외처리에 대해서 더 깊게 알아야 되겠다는 생각에 더 파보았다.
예외처리는 스프링에서
크게 세개의 Resolver로 관리된다.
1. ExceptionHandlerExceptionReolver
@ExcetionHandler를 구현해주는 클라스이다.
1. ResponseStatusExcetionResolver
@ResponseStatus를 사용하여 더 구체적인 예외 내용을 전달한다. 
1. DefaultHandlerExcetionResolver
위의 예외처리가 모두 실패 하였을 때 마지막으로 처리한다. 상황의 맞는 응답코드를 반환한다.

이제 이유가 보이기 시작한다. NexacroExceptionMappingRsolver도 결국 예외처리를 하기 위해 확보된 클라스였는데 내부 소스를 보니  modelandview로 넥사크로와 통신하고 있었다. 그런데 @ExceptionHandler에서 modelandview를 반환하지 않고, 다시 예외르 던져서 처리하려고 했는데 ExceptionResolver 간에는 수평 관계이기 때문에 서로를 몰라 예외처리가 해결이 안된 것이다.

해결책 1:
@HanlderException 에 NexacroMapingHandlerResolver의 소를 참고하여 별도의 예외처리르 하는 코드를 만들거나
NexacroMappingExceptionResolver와 소스가 동일한 CustomNeacroExceptionResolver 만들어  ServiceExecutor 내부의 Exceptionlogging() 메서드 로직을 가져와 넣어주면 되었다.
두번째 방식을 사용하는 것이 @HandlerException과 NexacroMappingHandlerResolver를 동시에 관리하는 것보다 더 간단해보인 관계로 두번째 방식을 시도해보기로 했다.



### getDsRowToMap()

해결책 1: 
본래는 DataSet의 하위 메서드로 구현하여 DataSet을 자연스럽게 가공하는 형태의 메서드로 구현할 생각을 했습니다.

문제점 1:
DataSet의 경우 넥사크로에서 제공되는 Class 파일에 캡슐화 되어있어 상속받아서 구현할 방법을 찾을 수 었었습니다.

DataSet을 상속하여 구현할 수 있는지 확인!

해결책 2:
별도의 클래스를 생성하여 ConvertDataSetRowToMap(DataSet, Int)  메서를 구현합니다. 이떄,
Int 해당 Row의 값입니다.

해결책 3:
해결책을 찾았다. 왜 인지는 모르겠으나 getRowToMap(int) 메서드가 DataSet 하위에 이미 구성되어있었다. 아무래도 당연히 없으니 ServiceExecutor에서 구현해 놨을 거라고 생각한 모양이다. 아무래도 구버전 nexacro에서는 없던 기능이 포함됐는지도 모르겠다. 간단 명료하다. CleanCode에서 말하는 파라미터가 적을 수록 좋으며 의미도 훨씬 명명백백하다. Dataset.getRowToMap(row) 이미 있는 메서드를 왜이리 돌고돌아 왔는지 모르겠다. 역시 꺼진불도 다시보자!


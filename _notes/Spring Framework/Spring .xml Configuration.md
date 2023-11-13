회사의 프로젝트의 xml config 파일들을 하나하나 분석하면서 이해해본다.
	- 스프링 프레임워크에 대한 이해도를 높인다.
	- 스프링 시큐리티의 구조와 작동 방식을 이해한다.
	- web.xml의 Filter와 dispacher-servlet의 Interceptor의 차이를 이해한다.
	- RootApplicationContext와 WebApplicationContext의 차이를 이해한다.
# web.xml
웹프로젝트의 작동하는 순서의 시작은 web.xml 부터이다.
### Filter - 들어오는 요청의 경로에 따라 원하는 처리작업을 실행한다.
- org.springframwork.web.filter.CharcterEncodingFilter
	특정 URL 패턴을 갖는 동작들에 대해서 인코딩을 강제한다.
	원래는 각각의 HTML 헤더에서 개별적으로 처리가 가능한 부분이다.
- org.egovframe.rte.ptl.mvc.filter.HTMLTagFilter
	HTML Tag의 경우 XSS에 취약하다. 하여 전자정부프레임워크에서는 HTMLTagFilter를 통해 각각의 HTML 태그를 약속된 특수문자와 문자의 조합으로 변형 하도록 권장하는데 이를 실행해주는 클래스이다.
- org.springframework.web.filter.DelegatingFilterProxy
	스프링 시큐리티의 처리과정을 거칠 URL을 설정하는 부분이다. /* 로 설정하여 모든 URL이 스피링 시큐리티를 거치도록 설정.
### RootApplicationContext 생성
- contextConifgLocation / org.springframework.web.context.COntextLoaderListener
	다른 파일로 분리해 놓은 xml config을 불러오는 부분이다.
	이 프로젝트의 경우 아래와 같은 형태로 분리 되어 있었다.
	- context-security : Spring Security Configuration 파일이다.
	- context-datasource : DB 설정
	- context-transaction : DB 트랜젝션 설정
	- context-mapper : 
	- context-common : 
	- context-properties : 
	- context-validation : DB 
	- context-nexacro
	- context-aspect
### WebApplicationContext 생성
- org.springframwork.web.servlet.DispatcherServlet
### 기타
- welcome-file-list
	기본 초기 화면
- error-page
	각각의 에러코드 또는 예외사항에 맞춰 에러 페이지를 설정
- nexacro xeni
	넥사크로 추가 기능을 위한 부분  (e.g 엑셀 업로드 다운로드 등..)
## context-security
- http
	- entry-point-ref
		- CustomerAuthenticationFilter
			- CustomerAuthenticationSucessHandler
			- CustomerAuthenticationFailureHandler
			- CompositeSessionAuthenticationStrategy
				- ConcurrentSessionControlAuthenitcationStrategy
				- 
- intercept-url
- logout
- csrf
- custom-filter
- session-management
- 


# Context-xxx.xml

# Dispatcher-servlet.xml



# Further Questions
- The Spring Boot developed project does not have 'web.xml' then how does the 'Filter' kind function works?
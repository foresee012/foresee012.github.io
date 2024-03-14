---
title: Spring Security 공식 문서를 기반으로 spring boot3.1.5 + spring security 6.1.5 + kotlin 환경에서 OAuth2 로그인 개발하기 - google (1)
description: spring security 최신 버전을 기반으로 oauth 로그인을 개발해보았다.
date: 2024-02-24
categories: SpringBoot
tags: [Spring Boot, Spring Security, Kotlin, Backend, 백엔드]
---

새로 프로젝트를 시작하면서 로그인 개발을 하게 되었다. 빠르게 MVP를 개발하면서 우선적으로 소셜 로그인을 지원하기로 했는데, 문제는 spring security가 6버전으로 넘어오면서 기존과는 달라진 점이 많아졌다는 거다. 그래서 이왕 개발을 하는 김에 제대로 공부하고, 블로그에 정리도 해놓으려고 한다.

필자가 개발하는 환경은 spring boot 3.1.5+spring security 6.1.5+kotlin 이다. (spring boot 3.2.2 + spring security 6.2.1으로 진행했었는데, spring security의 <span style="background-color: #F7DDBE">debug모드가 오류</span>가 난다. [공식 GitHub에 있는 이슈](https://github.com/spring-projects/spring-security/issues/14370)에서는 해결이 됐다고 돼있긴 한데, 해결이 제대로 되지 않는 것 같다. 그래서 결국 다운그레이드 해줬다. 아직 불안정한듯하니 아직까지는 최소 6.1.5를 사용하는 것을 권장함.) gradle 기준 implementation은 다음과 같이 해주면 된다.

~~~kotlin
implementation("org.springframework.boot:spring-boot-starter-security")
implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
~~~

일단 먼저 **google oauth 로그인** 개발을 할 것이다. oauth 로그인을 위해서는 google cloud platform(https://console.cloud.google.com/apis)에서 프로젝트를 등록하고, Oauth 클라이언트를 생성해야한다.

### OAuth란 무엇인가?
들어가기에 앞서서, OAuth란 무엇인지에 대해 알아보고자 한다. 보통 OAuth는 소셜 로그인을 하면서 접하게 된다. Open Authorization의 약자로 애플리케이션이 사용자 인증(Authentication)을 통해 사용자의 리소스 접근 권한(Authorization)을 위임받는 산업 표준 프로토콜이다.

OAuth는 Resource Owner(사용자)가 Client(사용하는 앱)에 서비스 이용 요청을 보내면, 해당 클라이언트가 Auth Server(인증 서버)에 엑세스 토큰(이때 OAuth로 받아오는 토큰과 우리 앱에서 사용하는 토큰은 별개이다.)을 요청하고, Auth Server가 Resource Owner에게 인가 동의 요청을 받으면 액세스 토큰을 Client에 전달해준다. 그 토큰을 통해서 Client는 Resource Server에 리소스를 요청할 수 있게 되는 것이다.

그래서 보통 소셜 로그인으로 회원가입을 하게 되면, OAuth를 통해 제 3의 서비스에 인증을 하고 권한을 받아와서, 거기서 받아온 정보값을 기반으로 우리 앱의 DB에 정보를 저장하고 관리하게 되는 것이다. 

### Spring Security 설정
Spring Security를 사용하기 위해서는 설정을 해줘야한다. SecurityConfig.kt 파일을 생성해준다. (경로는 알아서 지정하면 되지만, 나같은 경우에는 Config 패키지를 따로 두고 이곳에 config 파일을 모두 모아둔다.)

우선 공식문서를 기반으로 틀을 작성해보자. Servlet Applications>OAuth2>OAuth2 Login 항목을 기반으로 작성한다. (https://docs.spring.io/spring-security/reference/servlet/oauth2/login/index.html) spring boot 2.x 기반으로 작성되어있기는 한데 일단 이대로 해보고, 호환이 안 되는 부분이 있으면 고쳐보기로 했다.

~~~yml
spring:
  security:
    oauth2:
      client:
        registration:	
          google:	
            client-id: google-client-id
            client-secret: google-client-secret
~~~

yml 파일에 해당 내용을 추가해줘야 한다. 그러면 준비까지는 끝났다. 이제 본격적으로 config 파일을 생성해서 security 설정을 해줘야 한다.

내가 진행하고 있는 프로젝트의 로그인 프로세스는 다음과 같다.


(1) 사용자가 특정 소셜 서비스를 통해 로그인을 시도한다.<br/>
(2) Oauth 인증을 통해 사용자 정보를 받아온다.<br/>
(3-1) 만약 해당 인증 정보가 앱 DB에 없다면 회원가입을 진행한 후 로그인한다.<br/>
(3-2) 해당 인증 정보가 앱 DB에 있다면 로그인을 진행한다.<br/>
(4) 로그인 후 Access Token을 생성해서 반환해준다.

공식문서에서 보면 OAuth2LoginConfig를 생성하라고 되있는데, 일단 구글 로그인의 경우 CommonOAuth2Provider 등에서 자동 설정이 다 돼있기 때문에 SecurityConfig 파일만 생성해줘도 된다.

~~~kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig() {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http
                .authorizeHttpRequests { request ->
                    request
                      .requestMatchers(AntPathRequestMatcher("/auth/**")).permitAll()
                      .anyRequest().authenticated()
                }
                .oauth2Login {
                    it
                      .successHandler(authenticationSuccessHandler)
                      .failureHandler(authenticationFailureHandler)

                }

        return http.build()
    }

}
~~~

일단 딱 봐도 2.xx버전과는 뭔가 형식이 다르다. 여기서 먼저 언급해야할 중요한 점이 있는데, java의 경우 모르겠으나 kotlin의 경우 **공식 문서 예시 코드를 복붙해도 동작을 안한다.** 나도 kotlin을 이번 기회에 처음 접해서 하고 있는 지라 정확한 이유는 잘 모르겠지만, 공식 예제 코드는 문법 오류 나고 동작을 안 한다 (...) 위의 예제 코드 방식대로 작성해야한다.

spring security 3.xx버전에서 가장 큰 변화가 이 Config 파일 작성 방법인 것 같다. 이전의 builder 방식(계속 . . . 이 이어지는)이 아니고, ``` Lambda DSL ``` 방식으로 변화되었다. [공식 문서](https://docs.spring.io/spring-security/reference/migration-7/configuration.html)에 따르면 5.2 버전부터 존재했던 방식이고, 7버전부터는 예전 방식은 사용할 수 없다고 한다. ~~6.xx 버전인 지금도 예전 방식은 안 되는 것 같은데 이건 뭔지 모르겠다...~~ 


>The Lambda DSL is the preferred way to configure Spring Security, the prior configuration style will not be valid in Spring Security 7 where the usage of the Lambda DSL will be required. This has been done mainly for a couple of reasons: <br/><br/>The previous way it was not clear what object was getting configured without knowing what the return type was. The deeper the nesting the more confusing it became. Even experienced users would think that their configuration was doing one thing when in fact, it was doing something else. <br/><br/>
Consistency. Many code bases switched between the two styles which caused inconsistencies that made understanding the configuration difficult and often led to misconfigurations.

공식문서에 따르면 이러한 이유로 Lambda DSL의 변환을 결정했다고 한다. 대충 해석하자면, 1)이전 방식은 어떤 object가 configure되는지 모호하다. 2)많은 code가 두 방식 사이에서 이동하면서 혼란을 야기하여 이해가 더 어려워졌다. 1)은 Lambda DSL을 도입한 이유고, 2)는 Lambda DSL으로 완전히 넘어오는 이유 정도인 듯하다. 

이전에 security를 사용하며 config 파일을 작성했을 때 끝없는 온점의 향연(...)에 뭘 구성하고 있는지 잘 모르겠다는 생각을 했던 것 같기도 하다. 확실히 바뀐 버전이 좀 더 명확하기는 한 듯. ~~~하지만 deprecated 된 게 너무 많아서 작성하기가 힘들다...~~~

다시 코드로 돌아가자면, 먼저 and()가 deprecated 됐다. 그래서 and() 없이 계속 이어진다. 그리고 WebSecurityConfigurerAdapter도 deprecated 됐는데, 이 부분과 관련된 얘기는 차후에 이어나가겠다. 그리고, antMatcher도 deprecated됐다. 예전엔 ```authorizeRequests().antMatchers("/auth/**").permitAll()``` 방식으로 각 controller의 mapping값에 대한 permit를 설정해줬었다. 하지만 3.x버전에서는 ```.authorizeHttpRequests { request -> request.requestMatchers(AntPathRequestMatcher("/auth/**")).permitAll()}``` 이런 방식이다.

antMatchers 대신에 사용하는 것이 requestMatchers다. requestMatcher(*uri*) 형태로 작성해줘도 되고, requestMatcher(AntPathRequestMatcher(*uri*)) 형태로 작성해줘도도 된다. ```public open fun requestMatchers(vararg patterns: String!)``` 와 ```public open fun requestMatchers(vararg requestMatchers: RequestMatcher!)``` 의 두 시그니처가 모두 존재하기 때문에, AntPathRequestMatcher을 사용해야하는 경우(ex. httpMethod 지정 필요)가 아니면 그냥 string 값을 바로 넣어주면 될 것 같다.

그리고 예시 코드에서 authorizeHttpRequest는 request -> request. (...) 으로 작성했는데, OauthLogin의 경우에는 it을 사용해서 작성했다. 둘 중 어느 방법이어도 상관 없다... (하지만 Lambda DSL 방식을 도입한 이유를 생각해보건대 좀 더 명확한 방법은 전자일 것이다.)

이 코드대로 spring을 실행시키면 oauth2가 일단 기본적으로 동작은 할 것이다. (필자는 처음에 안 돼서 도대체 뭐가 문제인지 한참을 헤맸는데 @Bean을 빼먹어서 안 된 거였다... 이런 사소한 실수 때문에 오류가 발생하는 경우가 잦으니 꼭 놓치지 말기를 바란다.) 그러면 기본 설정은 끝났으니, 이 다음부터 커스텀을 시작해보겠다.
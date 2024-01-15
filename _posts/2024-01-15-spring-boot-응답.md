---
title: Spring Boot에서의 Request, Response 형식
description: Spring Boot에서 Request, Response 형식을 설정하는 방법에 대해 공부해보았다.
date: 2024-01-15
categories: SpringBoot
tags: [Spring Boot, Spring, JAVA, Backend, 백엔드]
---

API를 개발하다보면 Controller단에서 Request를 특정한 형식으로 받고 Response를 특정한 형식으로 보내주게 된다. 이때 Response는 정상(Status Code 200 등)과 오류(Status Code 404, 500 등)로 나눌 수 있는데, 어떤식으로 코드를 작성할 수 있는지에 대해 정리해보았다.

우선 이 방법의 정리에 앞서서 Spring은 어떻게 동작하는지부터 알아보자. 큰 구조를 보면 클라이언트에서 HTTP 요청이 올떄, 1)Dispatcher Servlet에서 HTTP 요청을 받아온 후에 이 요청을 2)Handler Mapping을 통해 적절한 Controller로 보내준다. 거기서부터 3)Controller-Service-Repository 등, 개발한 구조에 따라 적절하게 일을 하고 4)Controller가 결과를 반환하면 Dispatcher Servlet이 클라이언트에 HTTP 응답을 보내준다. (View단을 제외하고 개발하는, Rest Controller 기준이다.)

그렇다면 Request, Response 형식에 대해 고민할때 우리가 공부해보아야할 지점은 1)Dispatcher Servlet 2)Controller일것이다. (Handler Mapping은 단순히 컨트롤러를 찾아서 매핑해주는 역할이므로) 일단 기본적으로 Request, Response의 형식을 지정해주기 위해 사용하는 방식은 다음과 같이 주로 3가지가 있다.

#### DTO 활용
내가 주로 쓰는 방식이다. Request, Response에 대한 DTO를 별도로 생성한 후에 이를 이용해서 파라미터로 Request를 받아오고, Response를 반환한다. 

#### HashMap 등의 기존 자료구조 활용
Request를 받을 때, DTO를 생성하지 않고 HashMap 등의 자료구조를 활용해서 받을 수 있다. Body에 들어오는 값이 하나밖에 없다거나, 일회성으로 쓰이거나 하는 경우에는 이 방식을 사용하면 편했던 것 같다.

#### RequestEntity, ResponseEntity 활용
Spring 프레임워크에 있는 RequestEntity, ResponseEntity를 활용할 수 있다. 사실 이 방법은 기존에는 그닥 활용해본적이 없는데, 최근에 활용해보게 됐다.

그러면 이 방법들에 대해서 차근차근 알아보도록 하자.  
먼저 클라이언트에서 HTTP 요청(body가 있는 POST 요청이라고 가정하자.)이 온다. 그러면 이 요청을 Dispatcher Servlet에서 가장 먼저 가로챈다. Dispatcher Servlet은 HTTP 프로토콜로 들어오는 모든 요청을 가장 먼저 받는 **Front Controller**다. 이 과정에 대해서 좀 더 정리해서 공부해보자.

1. Spring이 서버에서 돌아가고 있다. 그리고 클라이언트가 HTTP 요청을 보낸다.
2. Spring Boot에는 Apache Tomcat 내장되어있다. 이 Apache Tomcat이 Servlet Container 개념으로, 클라이언트가 보낸 HTTP 요청을 받아온다. (Tomcat에 대해서는 추후 자세히 다루겠다.)
3. Servlet은 HTTP 메시지를 HttpServletRequest 객체에 담는다.
4. HttpServletRequest 객체의 Body값을 MessageConverter을 통해 Mapping해준다.

즉, HttpServletRequest 객체를 MessageConverter을 이용해 Object로 Mapping해서 내가 설정한 DTO의 형태가 되는 것이다. 응답을 할때도 마찬가지로 반환해준 DTO를 MessageConverter(Json 형식으로 변환할때는 MappingJackson2HttpMessageConverter에서 ObjectMapper을 활용해서 변환됨.)를 통해 body에 Mapping해준다. 사실 값을 받아오는 건 크게 문제가 없고, 내가 최근에 고민했던 문제는 **Response 형식**이다. Status 값을 설정해줘야했는데, 기존에는 Body에 status code값을 따로 같이 보내주는 방법을 사용했었기 때문에 Response 자체의 status를 지정해준적은 없었다. (오류는 Exception Handler 사용했었음.) 하지만 과제 요구사항으로 status 자체를 설정해줘야 할 일이 생겨서, 이걸 어떻게 해야하는지 한참 헤맸던 것 같다.

만약 DTO를 반환해준다고 하자. 그러면 이때, Request를 받아오는 것과 마찬가지로 MessageConverter를 통해 DTO를 body에 Mapping해주고 이를 반환해줄 것이다. 즉, HTTP의 status code를 설정해줄 수가 없다. HTTP header나 status code 등을 설정하기 위해서는 **ResponseEntity를 이용해서 (body에는 DTO를 넣어주면 된다.) 반환**해줘야한다. 단, Content-type의 경우에는 다른 설정을 안 해줘도 기본적으로 application/json이 설정되므로 다른 값을 설정해주려면 @GetMapping 등에서 설정해줘야 된다.

응답 형식 등은 서로 합의 하에 설정하면 되지만, 만약 이러한 요건이 있어서 header/status 등을 따로 처리해야한다면 ResponseEntity를 사용해야하고, status만 설정하면 된다면 @ResponseStatus와 Exception Handler을 활용하는 게 좀 더 깔끔할 수도 있다. (개인 취향 차이인듯하지만, 나는 개인적으로 남용할 필요는 굳이 없다고 생각은 한다.)
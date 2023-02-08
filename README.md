# Spring_Exception

# 8. 예외 처리와 오류 페이지

# 1. 프로젝트 생성 & 서블릿 예외 처리 - 시작

### 스프링 부트 스타터 사이트로 이동해서 스프링 프로젝트 생성

https://start.spring.io

**프로젝트 선택**

Project: Gradle Project

Language: Java

Spring Boot: 2.5.x

**Project Metadata** 

Group: hello

Artifact: exception

Name: exception

Package name: hello.exception

Packaging: **Jar**

Java: 11

**Dependencies**: Spring Web, Lombok , Thymeleaf, Validation

### 동작 확인

기본 메인 클래스 실행( ExceptionApplication.main() )

http://localhost:8080 호출해서 Whitelabel Error Page가 나오면 정상 동작하는 것입니다.

## **서블릿 예외 처리 - 시작**

스프링이 아닌 순수 서블릿 컨테이너는 예외를 어떻게 처리하는지 알아봅시다.

**서블릿은 다음 2가지 방식으로 예외 처리를 지원한다.**

`Exception` (예외)

`response.sendError(HTTP 상태 코드, 오류 메시지)`

### **Exception(예외)**

**자바 직접 실행**

자바의 메인 메서드를 직접 실행하는 경우 `main` 이라는 이름의 쓰레드가 실행된다.

실행 도중에 예외를 잡지 못하고 처음 실행한 `main()` 메서드를 넘어서 예외가 던져지면, 예외 정보를 남기고 해당 쓰레드는 종료된다.

**웹 애플리케이션**

웹 애플리케이션은 사용자 요청별로 별도의 쓰레드가 할당되고, 서블릿 컨테이너 안에서 실행된다.

애플리케이션에서 예외가 발생했는데, 어디선가 `try ~ catch`로 예외를 잡아서 처리하면 아무런 문제가 없다. 

그런데 만약에 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖으로 까지 예외가 전달되면 어떻게 동작할까?

```java
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
```

결국 톰캣 같은 WAS 까지 예외가 전달된다. 

WAS는 예외가 올라오면 어떻게 처리해야 할까요? 한번 테스트 해봅시다. 그 전에 먼저 스프링 부트가 제공하는 기본 예외 페이지가 있는데 아래와 같이  이 기능은 꺼둡시다. (나중에 설명하는 기능입니다.)

`application.properties`

```java
server.error.whitelabel.enabled=false
```

`ServletExController` - 서블릿 예외 컨트롤러

```java
package hello.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Slf4j
@Controller
public class ServletExController {

    @GetMapping("/error-ex")
    public void errorEx(){
        throw new RuntimeException("예외 발생!");
    }
}
```

**실행**

실행해보면 다음처럼 tomcat이 기본으로 제공하는 오류 화면을 볼 수 있다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/006aaf4c-7f73-45d1-8e80-7ee1006c7d29/Untitled.png)

로그는 아래처럼 생성됩니다.

```java
2023-01-30 02:33:14.882 ERROR 65008 --- [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    
	: Servlet.service() for servlet [dispatcherServlet] 
	in context with path [] threw exception [Request processing failed; 
		nested exception is java.lang.RuntimeException: 예외 발생!] with root cause

java.lang.RuntimeException: 예외 발생!
```

웹 브라우저에서 개발자 모드로 확인해보면 HTTP 상태 코드가 500으로 보입니다.

`Exception` 의 경우 **서버 내부에서 처리할 수 없는 오류가 발생한 것으로 생각**해서 HTTP 상태 코드 500을 반환한다.

이번에는 아무 URL 이나 호출해봅시다. 이렇게 말이죠. ( [localhost:8080/something](http://localhost:8080/something) )

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9370101b-fb6a-47be-8715-6df5a5923c55/Untitled.png)

이 경우에는 톰캣이 기본으로 제공하는 404 오류 화면을 볼 수 있습니다!

### **response.sendError(HTTP 상태 코드, 오류 메시지)**

오류가 발생했을 때 `HttpServletResponse` 가 제공하는 `sendError` 라는 메서드를 사용해도 됩니다.

이것을 호출한다고 당장 예외가 발생하는 게 아니라, 서블릿 컨테이너에게 오류가 발생했다는 점을 전달할 수 있는 것입니다.

이 메서드를 사용하면 HTTP 상태 코드와 오류 메시지도 추가할 수 있습니다.

`response.sendError(HTTP 상태 코드)`

`response.sendError(HTTP 상태 코드, 오류 메시지)`

`ServletExController` - 추가

```java
@GetMapping("/error-404")
public void error404(HttpServletResponse response) throws IOException {
    response.sendError(404, "404 오류!");
}

@GetMapping("/error-500")
public void error500(HttpServletResponse response) throws IOException {
    response.sendError(500);
}
```

**sendError 흐름**

```java
WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())
```

`response.sendError()` 를 호출하면 `response` 내부에는 오류가 발생했다는 상태를 저장해둔다.

그리고 서블릿 컨테이너는 고객에게 응답 전에 `response` 에 `sendError()` 가 호출되었는지 확인한다.

그리고 호출되었다면 설정한 오류 코드에 맞추어 기본 오류 페이지를 보여준다.

실행해보면 다음처럼 서블릿 컨테이너가 기본으로 제공하는 오류 화면을 볼 수 있다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8cddb372-f8da-414c-89b9-376e15a5f7a2/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/00c10dfd-af0b-4e74-908a-1771ab0be31a/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3dee596e-d874-423c-85fd-2e746467e4d8/Untitled.png)

### **정리**

이렇게 서블릿 컨테이너가 제공하는 기본 예외 처리 화면은 사용자가 보기에 불편합니다. 

그렇다면 의미 있는 오류 화면을 제공해봅시다.

# 2 서블릿 예외 처리 - 오류 화면 제공

위에서 보이듯 서블릿 컨테이너가 제공하는 기본 예외 처리 화면은 고객 친화적이지 않습니다. 서블릿이 제공하는 오류 화면 기능을 사용해봅시다.

서블릿은 `Exception` (예외)가 발생해서 서블릿 밖으로 전달되거나 또는 `response.sendError()` 가 호출되었을 때 각각의 상황에 맞춘 오류 처리 기능을 제공합니다.

이 기능을 사용하면 친절한 오류 처리 화면을 준비해서 고객에게 보여줄 수 있어요

과거에는 `web.xml` 이라는 파일에 다음과 같이 오류 화면을 등록했습니다.

```html
<web-app>
      <error-page>
        <error-code>404</error-code>
        <location>/error-page/404.html</location>
      </error-page>
      <error-page>
        <error-code>500</error-code>
        <location>/error-page/500.html</location>
      </error-page>
      <error-page>
        <exception-type>java.lang.RuntimeException</exception-type>
        <location>/error-page/500.html</location>
      </error-page>
</web-app>
```

지금은 스프링 부트를 통해서 서블릿 컨테이너를 실행하기 때문에, 스프링 부트가 제공하는 기능을 사용해서 서블릿 오류 페이지를 등록하면 됩니다.

서블릿 오류 페이지 등록 - `WebServerCustomizer`

```java
package hello.exception;

import org.springframework.boot.web.server.ConfigurableWebServerFactory;
import org.springframework.boot.web.server.ErrorPage;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

@Component
public class WebServerCustomizer implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {

    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/errorpage/404");
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
        ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "errorpage/500");
        factory.addErrorPages(errorPage404, errorPage500, errorPageEx);
        
    }
}
```

`response.sendError(404)` : `errorPage404` 호출

`response.sendError(500)` : `errorPage500` 호출

`RuntimeException` 또는 그 자식 타입의 예외: `errorPageEx` 호출

500 예외가 서버 내부에서 발생한 오류라는 뜻을 포함하고 있습니다. 그렇기 때문에 여기서는 예외가 발생한 경우도 500 오류 화면으로 처리합니다.

오류 페이지는 예외를 다룰 때 해당 예외와 그 자식 타입의 오류를 함께 처리합니다. 

위의 경우 `RuntimeException` 은 물론이고 `RuntimeException` 의 자식도 함께 처리합니다.

오류가 발생했을 때 처리할 수 있는 컨트롤러가 필요합니다. 

위의 경우 `RuntimeException` 예외가 발생하면 `errorPageEx` 에서 지정한 `/error-page/500` 이 호출됩니다.

위의 오류들을 처리할 컨트롤러가 필요합니다.

오류들을 처리할 컨트롤러 - `ErrorPageController`

```java
package hello.exception.servlet;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
@Controller
public class ErrorPageController {
    @RequestMapping("/error-page/404")
    public String errorPage404(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 404");
        return "error-page/404";
    }

    @RequestMapping("/error-page/500")
    public String errorPage500(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 500");
        return "error-page/500";
    }
}
```

처리할 컨트롤러는 만들었으니 이제 오류 처리 View 을 만들어봅시다.

오류 처리 View - `/templates/error-page/404.html`

```java
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>404 오류 화면</h2>
    </div>
    <div>
        <p>오류 화면 입니다.</p>
    </div>
    <hr class="my-4">
</div> <!-- /container -->
</body>
</html>
```

오류 처리 View - `/templates/error-page/500.html`

```java
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>500 오류 화면</h2>
    </div>
    <div>
        <p>오류 화면 입니다.</p>
    </div>
    <hr class="my-4">
</div> <!-- /container -->
</body>
</html>
```

서버를 실행시켜서 테스트해봅시다. 아래 링크로 테스트하면 됩니다.

`http://localhost:8080`

`http://localhost:8080/error-ex`

`http://localhost:8080/error-404`

`http://localhost:8080/error-500`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a665cce4-67c4-49ef-85a6-cb2ce4714b4c/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5a282861-374f-4c8e-b041-cb074ea3595c/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/067c9c46-0ddb-4ab5-b31f-59da4945c886/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b97659bb-64fe-440b-b9c5-74d577aa038c/Untitled.png)

페이지가 없는 [localhost:8080](http://localhost:8080) 도 404 에러로 연결되는 것을 볼 수 있으며 설정한 오류 페이지가 정상 노출되는 것을 확인할 수 있습니다.


# 3. 서블릿 예외 처리 - 오류 페이지 작동 원리

서블릿은 `Exception`(예외)가 발생해서 서블릿 밖으로 전달되거나 또는 `response.sendError()` 가 호출 되었을 때 설정된 오류 페이지를 찾습니다.

### 예외발생 흐름

```java
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
```

### sendError 흐름

```java
WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러 (response.sendError())
```

> 여기서 되짚어보면 인터셉터는 아래와 같다.
(`preHandle`, `postHandle` ,`afterCompletion` 은 인터셉터가 수행하는 것이다.)
> 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fd1a331f-bdae-4bd9-a17f-6a469005478e/Untitled.png)

WAS 는 해당 예외를 처리하는 오류 페이지 정보를 확인합니다.

`new ErrorPage(RuntimeException.class, “/error-page/500”)`

예를 들어서 `RuntimeException` 예외가 WAS 까지 전달되면, WAS 는 오류 페이지 정보를 확인한다. 확인해보면 `RuntimeException`의 오류 페이지로 `/error-page/500` 이 지정되어 있다. WAS 는 오류 페이지를 출력하기 위해서 `/error-page/500` 을 다시 요청한다.

### 오류 페이지 요청 흐름

```java
WAS '/error-page/500' 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

**즉,**

### 예외 발생과 오류 페이지 요청 흐름

```java
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS '/error-page/500' 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

**중요한 점은 웹 브라우저(클라이언트)는 서버 내부에서 이런 일이 일어나는지 전혀 모른다는 점입니다. 오직 서버 내부에서 오류 페이지를 찾기 위해 추가적인 호출을 합니다.**

**즉, 정리하면 아래와 같습니다.**

1. 예외가 발생해서 WAS 까지 전파된다.
2. WAS 는 오류 페이지 경로를 찾아서 내부에서 오류 페이지를 호출한다. 이 때 오류 페이지 경로로 필터, 서블릿, 인터셉터, 컨트롤러가 모두 다시 호출됩니다.

그렇다면 어떻게 필터와 인터셉터가 다시 호출될까요?  다시 호출되는 부분은 조금 뒤에 자세히 알아볼 것입니다.

### 오류 정보 추가

WAS 는 오류 페이지를 단순히 다시 요청만 하는 것이 아니라 오류 정보를 `request` 의 `attribute` 에 추가해서 넘겨줍니다. 만약 필요하다면 오류페이지에서 이렇게 전달된 오류 정보를 사용할 수 있습니다.

오류 출력 - `ErrorPageController`

```java
package hello.exception.servlet;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
@Controller
public class ErrorPageController {

    // RequestDispatcher 상수로 정의되어 있음
    public static final String ERROR_EXCEPTION = "javax.servlet.error.exception";
    public static final String ERROR_EXCEPTION_TYPE = "javax.servlet.error.exception_type";
    public static final String ERROR_MESSAGE = "javax.servlet.error.message";
    public static final String ERROR_REQUEST_URI = "javax.servlet.error.request_uri";
    public static final String ERROR_SERVLET_NAME = "javax.servlet.error.servlet_name";
    public static final String ERROR_STATUS_CODE = "javax.servlet.error.status_code";

    @RequestMapping("/error-page/404")
    public String errorPage404(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 404");
        printErrorInfo(request);
        return "error-page/404";
    }

    @RequestMapping("/error-page/500")
    public String errorPage500(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 500");
        printErrorInfo(request);
        return "error-page/500";
    }

    private void printErrorInfo(HttpServletRequest request) {
        log.info("ERROR_EXCEPTION: ex=", request.getAttribute(ERROR_EXCEPTION_TYPE));
        log.info("ERROR_EXCEPTION_TYPE: {}", request.getAttribute(ERROR_EXCEPTION_TYPE));

        // ex 의 경우에는 NestedServletException 스프링이 한 번 감싸서 반환함.
        log.info("ERROR_MESSAGE: {}", request.getAttribute(ERROR_MESSAGE));

        log.info("ERROR_REQUEST_URI: {}", request.getAttribute(ERROR_REQUEST_URI));
        log.info("ERROR_SERVLET_NAME: {}", request.getAttribute(ERROR_SERVLET_NAME));
        log.info("ERROR_STATUS_CODE: {}", request.getAttribute(ERROR_STATUS_CODE));
        log.info("dispatchType= {}", request.getDispatcherType());
    }

}
```

`request.attribute`에 서버가 담아준 정보

`javax.servlet.error.exception` : 예외
`javax.servlet.error.exception_type` : 예외 타입
`javax.servlet.error.message` : 오류 메시지
`javax.servlet.error.request_uri` : 클라이언트 요청 URI
`javax.servlet.error.servlet_name` : 오류가 발생한 서블릿 이름
`javax.servlet.error.status_code` : HTTP 상태 코드

실행해봅시다.

`http://localhost:8080`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9d00499b-fbee-4781-8fa1-0ac65c707409/Untitled.png)

`http://localhost:8080/error-404`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3ecc2840-1949-4bd5-b90a-0cd54e709db6/Untitled.png)

`http://localhost:8080/error-ex`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2857bd26-4068-4f77-9735-1ff692b86d83/Untitled.png)

`http://localhost:8080/error-500`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c4c973a2-a5eb-4893-9bcf-21b486fcc1aa/Untitled.png)

# 4. 서블릿 예외 처리 - 필터

여기서의 목표는 예외 처리에 따른 필터와 인터셉터 그리고 서블릿이 제공하는 `DispatchType` 을 이해하는 것입니다.

### 예외 발생과 오류 페이지 요청 흐름

```java
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS '/error-page/500' 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

오류가 발생하면 오류 페이지를 출력하기 위해 WAS 내부에서 다시 한번 호출이 발생합니다.

이때 필터, 서블릿, 인터셉터도 모두 다시 호출됩니다. 그런데 로그인 인증 체크 같은 경우를 생각해보면, 이미 한번 필터나, 인터셉터에서 로그인 체크를 완료했습니다.

따라서 서버 내부에서 오류 페이지를 호출한다고 해서 해당 필터나 인터셉트가 한번 더 호출되는 것은 매우 비효율적이죠….

결국 클라이언트로 부터 발생한 정상 요청인지, 아니면 오류 페이지를 출력하기 위한 내부 요청인지 구분할 수 있어야 합니다.

서블릿은 이런 문제를 해결하기 위해 `DispatcherType` 이라는 추가 정보를 제공합니다.

### DispatcherType

필터는 이런 경우를 위해서 `dispatcherTypes` 라는 옵션을 제공합니다.

이전 챕터인 ‘3. 서블릿 예외 처리 - 오류 페이지 작동 원리’ 에서 우리는 로그를 아래와 같이 추가했습니다.

```java
log.info("dispatchType={}", request.getDispatcherType())
```

그리고 출력했을 때 오류 페이지에서 `dispatchType=ERROR` 로 나오는 것을 확인했습니다.

고객이 처음 요청했을 때는 `dispatcherType=REQUEST` 입니다. 

이렇듯 서블릿 스펙은 실제 고객이 요청한 것인지, 서버가 내부에서 오류 페이지를 요청하는 것인지 `DispatcherType` 으로 구분할 수 있는 방법을 제공합니다.

`javax.servlet.DispatcherType`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3c52c9c4-35fa-4600-be4c-d42836b04d0d/Untitled.png)

`FORWARD`: MVC 에서 배웠던 서블릿에서 다른 서블릿이나 JSP 을 호출할 때 `RequestDispatcher.forward(request, response);`

`INCLUDE`: 서블릿에서 다른 서블릿이나 JSP 의 결과를 포함할 때 `RequestDispatcher.include(request, response);`

`REQUEST`: 클라이언트 요청 시

`ASYNC`: 서블릿 비동기 호출

`ERROR`: 오류 요청

### 필터와 DispatcherType

필터와 DispatcherType 이 어떻게 사용되는지 알아봅시다.

`LogFilter` - `DispatcherType` 로그 추가

```java
package hello.exception.filter;

import lombok.extern.slf4j.Slf4j;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.UUID;

@Slf4j
public class LogFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("log filter init");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String requestURI = httpRequest.getRequestURI();
        String uuid = UUID.randomUUID().toString();
        try {
            log.info("REQUEST [{}][{}][{}]", uuid, request.getDispatcherType(), requestURI);
            chain.doFilter(request, response);
        } catch (Exception e) {
            throw e;
        } finally {
            log.info("RESPONSE[{}][{}][{}]", uuid, request.getDispatcherType(), requestURI);
        }
    }

    @Override
    public void destroy() {
        log.info("log filter destroy");
    }
}
```

로그를 출력하는 부분에 `request.getDispatcherType()` 을 추가해줍니다.

`WebConfig`

```java
package hello.exception;

import hello.exception.filter.LogFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import javax.servlet.DispatcherType;
import javax.servlet.Filter;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new LogFilter());
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);

        return filterRegistrationBean;
    }

}
```

`filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);`

이렇게 `REQUSET`, `ERROR`두 가지를 모두 넣으면 클라이언트 요청은 물론, 오류 페이지 요청에서도 필터가 호출됩니다.

아무것도 넣지 않으면 기본 값이 `DispatcherType.REQUEST` 이겠지요? 즉, 클라이언트의 요청인 경우에만 필터가 적용됩니다. 특별히 오류 페이지 경로도 필터를 적용할 것이 아니라면 기본 값 `DispatcherType.REQUEST` 을 사용하면 됩니다.

물론 오류 페이지 요청 전용 필터를 적용하고 싶으면 `DispatcherType.ERROR` 만 지정하면 됩니다.

# 5. 서블릿 예외 처리 - 인터셉터

**이제 필터 중복 호출은 제거했으니 인터셉터 중복 호출을 제거해야 합니다.**

`LogInterceptor` - `DispatcherType` 로그 추가

```java
package hello.exception.interceptor;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.UUID;

@Slf4j
public class LogInterceptor implements HandlerInterceptor {

    public static final String LOG_ID = "logId";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestURI = request.getRequestURI();
        String uuid = UUID.randomUUID().toString();
        request.setAttribute(LOG_ID, uuid);
        log.info("REQUEST [{}][{}][{}][{}]", uuid, request.getDispatcherType(), handler);

        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("postHandle [{}]", modelAndView);

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        String requestURI = request.getRequestURI();
        String logId = (String) request.getAttribute(LOG_ID);
        log.info("RESPONSE [{}][{}][{}]", logId, request.getDispatcherType(), requestURI);

        if (ex != null) {
            log.error("afterCompletion error!!", ex);
        }
    }
}
```

앞서 필터의 경우에는 필터를 등록할 때 어떤 `DispatcherType` 인 경우에 필터를 적용할지를 선택할 수 있었습니다. 

그런데 인터셉터는 서블릿이 제공하는 기능이 아니라 스프링이 제공하는 기능입니다. 

즉, `DispatcherType` 과 무관하게 항상 호출되지요.

대신에 인터셉터는 다음과 같이 요청 경로에 따라서 추가하거나 제외하기 쉽게 되어 있기 때문에, 이러한 설정을 사용해서 오류 페이지 경로를 `excludePathPatterns` 를 사용해서 빼주면 됩니다. 아래와 같이 말이죠.

`WebConfig` - `addInterceptors()` 추가

```java
package hello.exception;

import hello.exception.filter.LogFilter;
import hello.exception.interceptor.LogInterceptor;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import javax.servlet.DispatcherType;
import javax.servlet.Filter;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor())
                .order(1)
                .addPathPatterns("/**")
                .excludePathPatterns("/css/**", "/*.ico", "/error", "/error-page/**");
    }

    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new LogFilter());
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);

        return filterRegistrationBean;
    }

}
```

인터셉터와 중복으로 처리되지 않기 위해 앞의 `logFilter()` 의 `@Bean` 에 주석을 달아두어야 합니다.

현재 에러 페이지 내부 호출에서는 인터셉터가 호출되지 않지만 만약 여기에서 `/error-page/**` 를 제거하면 `error-page/500` 같은 내부 호출의 경우에도 인터셉터가 호출됩니다.

### 전체 흐름 정리

여기까지 전체 흐름을 정리해봅시다.

`/hello` 정상 요청

```java
WAS(/hello, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러 -> View
```

`/error-ex` 오류 요청

필터는 `DispatchType` 으로 중복 호출 제거 ( `dispatchType=REQUEST` )

인터셉터는 경로 정보로 중복 호출 제거( `excludePathPatterns("/error-page/**")` )

```java
1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
3. WAS 오류 페이지 확인
4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) -> 컨트롤러(/error-page/500) -> View
```

# 6. 스프링 부트 - 오류 페이지 1

지금까지는 예외 처리 페이지를 만들기 위해서 다음과 같은 복잡한 과정을 거쳤습니다.

`WebServerCustomizer`  을 만들고

예외 종류에 따라서 `ErrorPage` 을 추가하고

예외 처리용 컨트롤러 `ErrorPageController` 을 만들었다.

그런데 스프링 부트는 이런 과정을 모두 기본으로 제공합니다.

`ErrorPage` 을 자동으로 등록한다. 이 때 `/error` 라는 경로로 기본 오류 페이지를 설정합니다.

`new ErrorPage(”/error”)`, 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용됩니다.

서블릿 밖으로 예외가 발생하거나, `response.sendError(…)` 가 호출되면 모든 오류는 `/error` 을 호출하게 됩니다.

`BasicErrorController` 라는 스프링 컨트롤러를 자동으로 등록합니다.

`ErrorPage` 에서 등록한 `/error` 을 매핑해서 처리하는 컨트롤러입니다.

참고

`ErrorMvcAutoConfiguration` 이라는 클래스가 오류 페이지를 자동으로 등록하는 역할을 합니다.

### **주의**

스프링 부트가 제공하는 기본 오류 매커니즘을 사용하도록 `WebServerCustomizer` 에 있는 `@Component` 을 주석처리 해야 합니다.

이제 오류가 발생했을 때 오류 페이지로 `/error` 을 기본 요청합니다. 스프링 부트가 자동 등록한 `BasicErrorController` 는 이 경로를 기본으로 받습니다.

### 개발자는 오류 페이지만 등록하면 됩니다.

`BasicErrorController` 는 기본적인 로직이 모두 개발되어 있습니다.

개발자는 오류 페이지 화면만 `BasicErrorController` 가 제공하는 룰과 우선순위에 따라서 등록하면 됩니다. 

정적 HTML 이면 정적 리소스, 
뷰 템플릿을 사용해서 동적으로 오류 화면을 만들고 싶으면 뷰 템플릿 경로에 오류 페이지 파일을 만들어서 넣어두기만 하면 됩니다.

### 뷰 선택 우선순위

`BasicErrorController` 의 처리 순서

뷰 템플릿

`resources/templates/error/500.html`

`resources/templates/error/5xx.html`

정적 리소스(`static`, `public`)

`resources/static/error/400.html`

`resources/static/error/404.html`

`resources/static/error/4xx.html`

적용 대상이 없을 때 뷰 이름(`error`)

`resources/templates/error.html`

해당 경로 위치에 HTTP 상태 코드 이름의 뷰 파일을 넣어두면 됩니다.

뷰 템플릿이 정적 리소스보다 우선순위가 높고 404, 500처럼 구체적인 것이 5xx 처럼 덜 구체적인 것보다 우선순위가 높습니다. 5xx, 4xx 라고 하면 500대 오류와 400대 오류를 처리해줍니다.

이제 오류 뷰 템플릿을 추가해봅시다.

`resources/templates/error/4xx.html`

```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>4xx 오류 화면 스프링 부트 제공</h2>
    </div>
    <div>
        <p>오류 화면 입니다.</p>
    </div>
    <hr class="my-4">
</div> <!-- /container -->
</body>
</html>
```

`resources/templates/error/404.html`

```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>404 오류 화면 스프링 부트 제공</h2>
    </div>
    <div>
        <p>오류 화면 입니다.</p>
    </div>
    <hr class="my-4">
</div> <!-- /container -->
</body>
</html>
```

`resources/templates/error/500.html`

```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>500 오류 화면 스프링 부트 제공</h2>
    </div>
    <div>
        <p>오류 화면 입니다.</p>
    </div>
    <hr class="my-4">
</div> <!-- /container -->
</body>
</html>
```

이렇게 세가지 오류 페이지를 등록했습니다. 

`ServletExController` - `error400` 추가

```java
package hello.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Slf4j
@Controller
public class ServletExController {

    @GetMapping("/error-ex")
    public void errorEx() {
        throw new RuntimeException("예외 발생!");
    }

    @GetMapping("/error-400")
    public void error400(HttpServletResponse response) throws IOException {
        response.sendError(400, "400 오류!");
    }

    @GetMapping("/error-404")
    public void error404(HttpServletResponse response) throws IOException {
        response.sendError(404, "404 오류!");
    }

    @GetMapping("/error-500")
    public void error500(HttpServletResponse response) throws IOException {
        response.sendError(500);
    }

}
```

서버를 실행시키고 나서 테스트를 해봅시다.

[http://localhost:8080/error-404](http://localhost:8080/error-404) → 404.html

[http://localhost:8080/error-400](http://localhost:8080/error-400) → 4xx.html (400 오류 페이지가 없지만 4xx 가 있다.)

[http://localhost:8080/error-500](http://localhost:8080/error-500) → 500.html

[http://localhost:8080/error-ex](http://localhost:8080/error-ex) → 500.html (예외는 500 으로 처리한다.)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e3450d12-04d9-4d17-a4b0-14bdd26210c0/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4c3cea9d-4a03-4243-8a10-be4de23c967a/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a8a648ea-a135-44f4-b7a8-ff47aa035c63/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/68290b61-211e-42db-9f4a-f1b79d5b5dbe/Untitled.png)

# 7. 스프링 부트 - 오류 페이지 2

### BasicErrorController 가 제공하는 기본 정보들

`BasicErrorController` 컨트롤러는 다음 정보를 model 에 담아서 뷰에 전달합니다. 뷰 템플릿은 이 값을 활용해서 출력할 수 있습니다.

```java
* timestamp: Fri Feb 05 00:00:00 KST 2021
* status: 400
* error: Bad Request
* exception: org.springframework.validation.BindException
* trace: 예외 trace
* message: Validation failed for object='data'. Error count: 1
* errors: Errors(BindingResult)
* path: 클라이언트 요청 경로 (`/hello`)
```

`resources/templates/error/500.html` - 오류 정보 추가

```html
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
</head>
<body>
<div class="container" style="max-width: 600px">
    <div class="py-5 text-center">
        <h2>500 오류 화면 스프링 부트 제공</h2>
    </div>
    <div>
        <p>오류 화면 입니다.</p>
    </div>
    <ul>
        <li>오류 정보</li>
        <ul>
            <li th:text="|timestamp: ${timestamp}|"></li>
            <li th:text="|path: ${path}|"></li>
            <li th:text="|status: ${status}|"></li>
            <li th:text="|message: ${message}|"></li>
            <li th:text="|error: ${error}|"></li>
            <li th:text="|exception: ${exception}|"></li>
            <li th:text="|errors: ${errors}|"></li>
            <li th:text="|trace: ${trace}|"></li>
        </ul>
        </li>
    </ul>
    <hr class="my-4">
</div> <!-- /container -->
</body>
</html>
```

오류 관련 내부 정보들을 고객에게 노출하는 것은 좋지 않습니다. 고객이 해당 정보를 읽어도 혼란만 더해지고, 보안상 문제가 될 수도 있습니다.다.

그래서 `BasicErrorController` 오류 컨트롤러에서 다음 오류 정보를 `model` 에 포함할지 여부를 선택할 수 있습니다.

`application.properties`

`server.error.include-exception=false:` `exception` 포함 여부 (true, false)

`server.error.include-message=never:` `message` 포함 여부

`server.error.include-stacktrace=never:` `trace` 포함 여부

`server.error.include-binding-errors=never:` `errors` 포함 여부

`application.properties`

```html
server.error.include-exception=true
server.error.include-message=on_param
server.error.include-stacktrace=on_param
server.error.include-binding-errors=on_param
```

기본 값이 `never` 인 부분은 다음 3 개의 옵션을 사용할 수 있습니다.

`never`, `always`, `on_param`

`never`: 사용하지 않음

`always`: 항상 사용

`on_param`: 파라미터가 있을 때 사용

`on_param` 은 파라미터가 있으면 해당 정보를 노출한다. 디버그 시 문제를 확인하기 위해 사용할 수 있다.

그런데 이 부분도 개발 서버에서 사용할 수 있지만, 운영 서버에서는 권장하지 않는다.

`on_param` 으로 설정하고 다음과 같이 HTTP 요청시 쿼리 파라미터를 전달하면 해당 정보들이 `model` 에 담겨서 뷰 템플릿에서 출력된다. `message=&errors=&trace=`

테스트

http://localhost:8080/error-ex

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/68d08d5c-dd64-4888-be4b-933c98c900cd/Untitled.png)

http://localhost:8080/error-ex?message=&errors=&trace=

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b8107f3d-5de2-4aea-9861-014df6aca61c/Untitled.png)

**그런데 당연히 실무에서는 이것들을 노출하면 안 됩니다.**

사용자에게는 이쁜 오류 화면과 고객이 이해할 수 있는 간단한 오류 메시지를 보여주고 오류는 서버에 로그로 남겨서 로그로 확인해야 합니다!

### 스프링 부트 오류 관련 옵션 (application.properties 에)

`server.error.whitelabel.enabled=true`: 오류 처리 화면을 못 찾을 시, 스프링 whitelabel 오류 페이지 적용

`server.error.path=/error`: 오류 페이지 경로, 스프링이 자동 등록하는 서블릿 글로벌 오류 페이지 경로와 `BasicErrorController` 오류 컨트롤러 경로에 함께 사용된다.

**확장 포인트**

에러 공통 처리 컨트롤러의 기능을 변경하고 싶으면 `ErrorController` 인터페이스를 상속 받아서 구현하거나 `BasicErrorController` 상속 받아서 기능을 추가하면 된다.

**정리**

스프링 부트가 기본으로 제공하는 오류 페이지를 활용하면 오류 페이지와 관련된 대부분의 문제는 손쉽게 해결할 수 있다.

# ==============================

# ========= 9. API 에외 처리 =========

# 1. API 예외 처리 개요

먼저 API 예외 처리는 어떻게 해야 할까요?

HTML 페이지의 경우 지금까지 설명했던 것 처럼 4xx, 5xx 와 같은 오류 페이지만 있으면 대부분의 문제를 해결할 수 있습니다.

**그런데 API의 경우에는 생각할 내용이 더 많습니다.**

 오류 페이지는 단순히 고객에게 오류 화면을 보여주고 끝이지만 API 는 각 오류 상황에 맞는 오류 응답 스펙을 정하고 JSON 으로 데이터를 내려주어야 합니다.

지금부터 API 의 경우 어떻게 예외 처리를 하면 좋은지 알아봅시다.

API 도 오류 페이지에서 설명했던 것처럼 처음으로 돌아가서 서블릿 오류 페이지 방식을 사용해봅시다.

`WebServerCustomizer` 다시 동작

```java
package hello.exception;

import org.springframework.boot.web.server.ConfigurableWebServerFactory;
import org.springframework.boot.web.server.ErrorPage;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

// 스프링 부트 기본 매커니즘을 사용할 때 @Component 을 주석 처리합니다.

@Component
public class WebServerCustomizer implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {

    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error-page/404");
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
        ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "/error-page/500");
        factory.addErrorPages(errorPage404, errorPage500, errorPageEx);

    }
}
```

`WebServerCustomizer` 가 다시 사용되도록 하기 위해 `@Component` 애노테이션에 있는 주석을 풉시다. 

이제 WAS 에 예외가 전달되거나, `response.sendError()` 가 호출되면 위에 등록한 예외 페이지 경로가 호출됩니다.

`ApiExceptionController` - API 예외 컨트롤러

```java
package hello.exception.api;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
public class ApiExceptionController {
    @GetMapping("/api/members/{id}")
    public MemberDto getMember(@PathVariable("id") String id) {

        if (id.equals("ex")) {
            throw new RuntimeException("잘못된 사용자");
        }
        return new MemberDto(id, "hello " + id);
    }

    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String memberId;
        private String name;
    }

}
```

단순히 회원을 조회하는 기능을 하나 만들었다. 예외 테스트를 위해 URL 에 전달된 `id` 의 값이 `ex` 이면 예외가 발생하도록 코드를 심어두었다.

### Postman 으로 테스트

HTTP Header 에 `Accept` 가 `application/json` 인 것을 꼭 확인합시다.

만약 정상 호출이라면 

http://localhost:8080/api/members/spring

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/907b1354-c5d8-4517-8b8d-d152278275b9/Untitled.png)

여기서는 Accept 의 VALUE 을 자동으로 설정해주는 것 같다.

만약 비정상적인 호출(예외 발생 호출)이라면

http://localhost:8080/api/member/ex

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/677116b8-b51b-44f6-a210-c6447e890008/Untitled.png)

API를 요청했는데, 정상의 경우 API로 JSON 형식으로 데이터가 정상 반환된다. 

그런데 오류가 발생하면 우리가 미리 만들어둔 오류 페이지 HTML이 반환된다. JSON 으로 데이터를 가져올 때 이런 식으로 HTML 을 가져오면 안된다. 

클라이언트는 정상 요청이든, 오류 요청이든 JSON이 반환되어야 합니다. 웹 브라우저가 아닌 이상 HTML을 직접 받아서 할 수 있는 것은 없습니다.

**문제를 해결하려면 오류 페이지 컨트롤러도 JSON 응답을 할 수 있도록 수정해야 합니다!!**

`ErrorPageController` - API 응답 추가

```java
@RequestMapping(value = "/error-page/500", produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<Map<String, Object>> errorPage500Api(
        HttpServletRequest request, HttpServletResponse response) {
    
    log.info("API errorPage 500");
    Map<String, Object> result = new HashMap<>();
    Exception ex = (Exception) request.getAttribute(ERROR_EXCEPTION);
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));
    result.put("message", ex.getMessage());

    Integer statusCode = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
    return new ResponseEntity(result, HttpStatus.valueOf(statusCode));
}
```

`produces = MediaType.APPLICATION_JSON_VALUE` 의 뜻은 클라이언트가 요청하는 HTTP Header 의 `Accept` 의 값이 `application/json` 일 때 해당 메서드가 호출된다는 것이다. 결국 클라이언트가 받고 싶은 미디어타입이 json이면 이 컨트롤러의 메서드가 호출된다.

응답 데이터를 위해서 `Map` 을 만들고 `status`, `message` 키에 값을 할당했습니다. Jackson 라이브러리는 `Map` 을 JSON 구조로 변환할 수 있습니다.

`ResponseEntity` 을 사용해서 응답하기 때문에 메시지 컨버터가 동작하면서 클라이언트에 JSON 이 반환됩니다.

포스트맨을 통해서 다시 테스트해봅시다.

http://localhost:8080/api/members/ex

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e424e3d3-3643-447e-8f25-183a83af966a/Untitled.png)

HTTP Header 에 `Accept` 가 `application/json` 이 아니면, 기존 오류 응답인 HTML 응답이 출력되는 것을 확인할 수 있습니다.

# 2. 스프링부트 기본 오류 처리

API 예외 처리도 스프링 부트가 제공하는 기본 오류 방식을 사용할 수 있습니다.

스프링 부트가 제공하는 `BasicErrorController` 코드를 봅시다.

`BasicErrorController` 코드

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dd431417-e988-49d6-b132-3395e3785ebb/Untitled.png)

`/error` 와 동일한 경로를 처리하는 `errorHtml(),` `error()` 두 메서드를 확인할 수 있다.

`errorHtml()`: `produces = MediaType.TEXT_HTML_VALUE` : 클라이언트 요청의 Accept 헤더 값이 `text/html` 인 경우에는 `errorHtml()` 을 호출해서 view 을 제공합니다.

`error()` : 그 외의 경우에 호출되고 `ResponseEntity` 로 HTTP Body 에 JSON 데이터를 반환한다.

### 스프링 부트의 예외 처리

앞서 학습했듯이 스프링 부트의 기본 설정은 오류 발생시 `/error` 을 오류 페이지로 요청합니다.

`BasicErrorController` 는 이 경로를 기본으로 받습니다. (`application.properties` 파일에서 `server.error.path` 로 수정 가능, 기본 경로는 `/error` )

### Postman 으로 실행해봅시다.

http://localhost:8080/api/members/ex

주의

`BasicErrorController` 을 사용하도록 `WebServerCustomizer` 의 `@Component` 을 주석처리합시다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/89c3e324-db1e-4ecd-b806-48e574643001/Untitled.png)

스프링 부트는 `BasicErrorController` 가 제공하는 기본 정보들을 활용해서 오류 API 을 생성해줍니다.

[`application.properties`](http://application.properties) 에 다음 옵션들을 설정하면 더 자세한 오류 정보를 추가할 수 있습니다.

`server.error.include-binding-errors=always`

`server.error.include-exception=true`

`server.error.include-message=always`

`server.error.include-stacktrace=always`

물론 오류 메시지는 이렇게 막 추가하면 보안상 위험할 수도 있습니다. 간결한 메시지만 노출하고 로그를 통해 확인하는 것이 바람직합니다.

### Html 페이지 vs API 오류

`BasicErrorController` 을 확장하면 JSON 메시지도 변경할 수 있습니다. 

그런데 API 오류는 조금 뒤에 설명할 `@ExceptionHandler`  가 제공하는 기능을 사용하는 것이 더 나은 방법이므로 지금은 `BasicErrorController` 을 확장해서 JSON 오류 메시지를 변경할 수 있다 정도로만 이해해 둡시다.

스프링 부트가 제공하는 `BasicErrorController` 는 HTML 페이지를 제공하는 경우에는 매우 편리합니다. 4xx, 5xx 등등 모두 잘 처리해줍니다. 그런데 API 오류 처리는 다른 차원의 이야기입니다. API 마다 각각의 컨트롤러나 예외마다 서로 다른 응답 결과를 출력해야 할 수도 있습니다. 예를 들어서 회원과 관련된 API 예외가 발생할 때 응답과 상품과 관련된 API 에서 발생하는 예외에 따라 그 결과가 달라질 수도 있습니다.

결과적으로 매우 세밀하고 복잡합니다. 따라서 이 방법은 HTML 화면을 처리할 때 사용하고 API 오류 처리는 뒤에서 섧명할 `@ExceptionHanlder` 을 이용하는 것이 좋습니다.

그렇다면 복잡한 API 오류는 어떻게 처리해야 하는지 지금부터 하나씩 알아봅시다.

# 3. HandlerExceptionResolver 시작

우리는 예외 처리를 아래와 같이 할 것입니다.

 예외가 발생해서 서블릿을 넘어 WAS 까지 예외가 전달되면 HTTP 상태코드가 500으로 처리됩니다. 발생하는 예외에 따라서 400, 404 등등 다른 상태코드로 처리하고 싶습니다.

그리고 오류 메시지, 형식 등을 API 마다 다르게 처리할 것입니다.

### 상태코드 변환

예를 들어서 `IllegalArgumentException`  을 처리하지 못해서 컨트롤러 밖으로 넘어가는 일이 발생하면 HTTP 상태 코드를 400으로 처리하고 싶습니다. 어떻게 해야 할까요?

`ApiExceptionController` - 수정

```java
@Slf4j
@RestController
public class ApiExceptionController {
    @GetMapping("/api/members/{id}")
    public MemberDto getMember(@PathVariable("id") String id) {

        if (id.equals("ex")) {
            throw new RuntimeException("잘못된 사용자");
        }
        if (id.equals("bad")) {
            throw new IllegalArgumentException("잘못된 입력 값");
        }

        return new MemberDto(id, "hello " + id);

    }
		...
}
```

[http://localhost:8080/api/members/bad](http://localhost:8080/api/members/bad) 라고 호출하면 `IllegalArgumentException` 이 발생하도록 만들었습니다.

실행해보면 상태 코드가 500 인 것을 확인할 수 있습니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0ceac9ab-860c-4174-9406-a62fcb186704/Untitled.png)

### HandlerExceptionResolver

스프링 MVC 는 컨트롤러(핸들러) 바깥으로 예외가 던져진 경우에 예외를 해결하고 동작을 새로 정의할 수 있는 방법을 제공합니다. 

컨트롤러 밖으로 던져진 예외를 해결하고 동작 방식을 변경하고 싶으면 `HanlderExceptionResolver` 을 사용하면 됩니다. 줄여서 `ExceptionResolver` 라고 합니다.

그림을 통해 확인해봅시다.

### ExceptionResolver 적용 전

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0ad45146-3671-4330-908e-2471accd5a47/Untitled.png)

### ExceptionResolver 적용 후

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9f87d438-c389-4cd4-b977-8d0bcc4e3bbf/Untitled.png)

참고: `ExceptionResolver` 로 예외를 해결해도 `postHandle()` 은 호출되지 않습니다. 

`HandlerExceptionResolver` - 인터페이스

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ff5c5429-4561-4ba9-bb0e-a6185d4cca3e/Untitled.png)

`handler`: 핸들러(컨트롤러) 정보

`Exception ex`: 핸들러(컨트롤러)에서 발생한 예외

그렇다면 HandlerExceptionResolver 인터페이스를 구현하는 클래스를 만들어봅시다.

`MyHandlerExceptionResolver`

```java
@Slf4j
public class MyHandlerExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof IllegalArgumentException) {
                log.info("IllegalArgumentException resolver to 400");
                response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());
                return new ModelAndView();
            }
        } catch (IOException e) {
            log.error("resolver ex", e);
        }
        return null;
    }
}
```

`ExceptionResolver` 가 `ModelAndView` 을 반환하는 이유는 마치 `try`, `catch` 을 하듯이 `Exception` 을 처리해서 정상 흐름처럼 변경하는 것이 목적입니다. 이름 그대로 `Exception` 을 `Resolver` 하는 것이지요!

여기서는 `IllegalArgumentException` 이 발생하면 `response.sendError(400)` 을 호출해서 HTTP 상태코드를 400 으로 지정하고 비어있는 `ModelAndView` 을 반환합니다.

### 반환 값에 따른 동작 방식

`HandlerExceptionResolver` 의 반환 값에 따른 `DispatcherServlet` 의 동작방식은 다음과 같습니다.

빈 `ModelAndView` → `new ModelAndView()` 처럼 빈 `ModelAndView` 을 반환하면 뷰를 렌더링하지 않고 정상 흐름으로 서블릿이 리턴됩니다.

`ModelAndView` 지정 → `ModelAndView` 에 `View`, `Model` 등의 정보를 지정해서 반환하면 뷰를 렌더링합니다.

`null` → `null` 을 반환하면 다음 `ExceptionResolver` 을 찾아서 실행합니다. 만약 처리할 수 있는 `ExceptionResolver` 가 없으면 예외 처리가 되지 않고, 기존에 발생한 예외를 서블릿 바깥으로 던집니다.

### ExceptionResolver 활용

예외 상태 코드 변환

예외를 `response.sendError(xxx)` 호출로 변경해서 서블릿에서 상태 코드에 따른 오류를 처리하도록 위임합니다. 이후에 WAS 는 서블릿 오류 페이지를 찾아서 내부 호출합니다. 예를 들어서 스프링 부트가 기본으로 설정한 `/error` 가 호출되는 것처럼 말이죠.

뷰 템플릿 처리

`ModelAndView` 에 값을 채워서 예외에 따른 새로운 오류 화면 뷰를 렌더링해서 고객에게 제공합니다.

API 응답 처리

`response.getWriter().println(”hello”);` 처럼 HTTP 응답 바디에 직접 데이터를 넣어주는 것도 가능합니다. 여기에 JSON 으로 응답하면 API 응답 처리를 할 수 있습니다.

`WebConfig` - 수정(`WebMvcConfigurer` 을 통해 등록함)

```java
package hello.exception;

import hello.exception.filter.LogFilter;
import hello.exception.interceptor.LogInterceptor;
import hello.exception.resolver.MyHandlerExceptionResolver;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import javax.servlet.DispatcherType;
import javax.servlet.Filter;
import java.util.List;

@Configuration
public class WebConfig implements WebMvcConfigurer {

		...

    @Override
    public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add(new MyHandlerExceptionResolver());
    }
}
```

`configureHandlerExceptionResolvers(…)` 을 사용하면 스프링이 기본으로 등록하는 `ExceptionResolver` 가 제거되므로 주의합시다. 우리는 `extendHandlerExceptionResolvers` 을 사용합니다.

### Postman 으로 실행

[http://localhost:8080/api/members/ex](http://localhost:8080/api/members/ex) → HTTP 상태 코드 500

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/91748689-5094-4a3b-b9f0-7b36a7f120bb/Untitled.png)

[http://localhost:8080/api/members/bad](http://localhost:8080/api/members/bad) → HTTP 상태 코드 400

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/962c7d7b-4428-4227-90bd-ae78d33e5c2e/Untitled.png)

이렇게 `HandlerExceptionResolver` 을 사용하여 API 예외처리를 수행했습니다.

# 4. HandlerExceptionResolver 활용

### 예외를 여기서 마무리하기

예외가 발생하면 WAS 까지 예외가 던져지고 WAS 에서 오류 페이지 정보를 찾아서 다시 `/error` 을 호출하는 과정은  너무 복잡합니다. `ExceptionResolver` 을 활용하면 예외가 발생했을 때 이런 복잡한 과정 없이 여기에서 문제를 깔끔하게 해결할 수 있습니다.

예제로 알아봅시다. 먼저 사용자 정의 예외를 하나 추가할 것입니다.

`UserException`

```java
package hello.exception.exception;

public class UserException extends RuntimeException {
    public UserException() {
        super();
    }

    public UserException(String message) {
        super(message);
    }

    public UserException(String message, Throwable cause) {
        super(message, cause);
    }

    public UserException(Throwable cause) {
        super(cause);
    }

    protected UserException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
}
```

`ApiExceptionController` - 예외 추가

```java
package hello.exception.api;

import hello.exception.exception.UserException;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
public class ApiExceptionController {
    @GetMapping("/api/members/{id}")
    public MemberDto getMember(@PathVariable("id") String id) {

        if (id.equals("ex")) {
            throw new RuntimeException("잘못된 사용자");
        }
        if (id.equals("bad")) {
            throw new IllegalArgumentException("잘못된 입력 값");
        }
        if (id.equals("user-ex")) {
            throw new UserException("사용자 오류");
        }

        return new MemberDto(id, "hello " + id);

    }

    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String memberId;
        private String name;
    }

}
```

[http://localhost:8080/api/members/user-ex](http://localhost:8080/api/members/user-ex) 호출 시 UserException 이 발생하도록 해두었습니다.

이제 이 예외를 처리하는 `UserHandleExceptionResolver` 을 만들어봅시다.

`UserHandlerExceptionResolver`

```java
package hello.exception.exception.resolver;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import hello.exception.exception.UserException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ui.Model;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@Slf4j
public class UserHandlerExceptionResolver implements HandlerExceptionResolver {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {

        try {
            if (ex instanceof UserException) {
                log.info("UserException resolver to 400");
                String acceptHeader = request.getHeader("accept");
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);

                if ("application/json".equals(acceptHeader)) {
                    Map<String, Object> errorResult = new HashMap<>();
                    errorResult.put("ex", ex.getClass());
                    errorResult.put("message", ex.getMessage());
                    String result = objectMapper.writeValueAsString(errorResult);
                    response.setContentType("application/json");
                    response.setCharacterEncoding("utf-8");
                    response.getWriter().write(result);
                    return new ModelAndView();
                } else {
                    // TEXT/HTML
                    return new ModelAndView("error/500");
                }
            }
        } catch (IOException e) {
            log.error("resolver ex", e);
        }
        return null;
    }

}
```

HTTP 요청 헤더의 `ACCEPT` 값이 `application/json` 이면 JSON 으로 오류를 내려주고 그 외 경우에는 `error/500` 에 있는 HTML 오류 페이지를 보여줍니다.

`WebConfig` 에 `UserHandlerExceptionResolver` 추가

```java
@Override
public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
    resolvers.add(new MyHandlerExceptionResolver());
    resolvers.add(new UserHandlerExceptionResolver());
}
```

### POSTMAN 실행.

http://localhost:8080/api/members/user-ex

`ACCEPT` : `application/json` 인 경우

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/686f0b83-21e9-429d-843c-2c6630b78561/Untitled.png)

`ACCEPT` : `text/html`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/32b6a516-f2b8-426e-8356-b51a9a3bd2d7/Untitled.png)

정리하자면 아래와 같습니다.

ExceptionResolver 을 사용하면 컨트롤러에서 예외가 발생해도 ExceptionResolver 에서 예외를 처리해버립니다.

따라서 예외가 발생해도 서블릿 컨테이너까지 예외가 전달되지 않고, 스프링 MVC 에서 예외 처리는 끝이 납니다. 

결과적으로 WAS 입장에서는 정상처리된 것으로 인식됩니다. 이렇게 예외를 이곳에서 모두 처리할 수 있다는 것이 핵심입니다!

서블릿 컨테이너까지 예외가 올라가면 복잡하고 지저분하게 추가 프로세스가 실행되지만 `ExceptionResolver` 을 사용하면 예외처리가 상당히 깔끔해집니다.

그런데 직접 `ExceptionResolver` 을 구현하려고 하니 상당히 복잡합니다.

다음 포스팅에서부터 스프링이 제공하는 `ExceptionResolver` 들을 알아봅시다.

# 5. 스프링이 제공하는 ExceptionResolver1

스프링 부트가 기본으로 제공하는 `ExceptionResolver` 는 다음과 같습니다.

`HandlerExceptionResolverComposite` 에 아래 순서로 등록함.

.1. `ExceptionHandlerExceptionResolver`

.2. `ResponseStatusExceptionResolver`

.3. `DefaultHandlerExceptionResolver` (우선순위가 가장 낮다.)

**ExceptionHandlerExceptionResolver**

`@ExceptionHandler` 을 처리합니다. API 예외 처리는 대부분 이 기능으로 해결합니다. 조금 뒤에 자세히 설명하겠습니다.

**ResponseStatusExceptionResolver**

HTTP 상태 코드를 지정해줍니다.

예) `@ResponseStatus(value = HttpStatus.NOT_FOUND)`

**DefaultHandlerExceptionResolver**

스프링 내부 기본 예외를 처리합니다.

먼저 가장 쉬운 `ResponseStatusExceptionResolver` 부터 알아봅시다.

### ResponseStatusExceptionResolver

`ResponseStatusExceptionResolver` 는 예외에 따라서 HTTP 상태 코드를 지정해주는 역할을 합니다. 

아래 두 가지 경우를 처리합니다.

`@ResponseStatus` 가 달려있는 예외

`@ResponseStatusException` 예외

하나씩 확인해봅시다.

예외에 아래와 같이 `@ResponseStatus` 애노테이션을 적용하면 HTTP 상태 코드를 변경해줍니다.

`BadRequestException`

```java
package hello.exception.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")
public class BadRequestException extends RuntimeException {
}
```

`BadRequestException` 예외가 컨트롤러 바깥으로 넘어간다면 `RequestStatusExceptResolver` 예외가 해당 애노테이션을 확인해서 오류 코드를 `HttpStatus.BAD_REQUEST` (400) 으로 변경하고 메시지도 담습니다.

`ResponseStatusExceptionResolver` 코드를 확인해보면 결국 `response.sendError(statusCode, resolvedReason)` 을 호출하는 것을 확인할 수 있습니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ded7908e-ad6a-405f-90a2-855487916dba/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/92d8b6e8-265e-4eb8-91ca-2d0896221642/Untitled.png)

`sendError(400)` 을 호출했기 때문에 WAS 에서 다시 오류 페이지`(/error)` 을 내부 요청합니다.

`ApiExceptionController` - 추가

```java
@Slf4j
@RestController
public class ApiExceptionController {

    @GetMapping("/api/response-status-ex1")
    public String responseStatusEx1() {
        throw new BadRequestException();
    }
		...
}
```

실행

http://localhost:8080/api/response-status-ex1?message=

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/426ed347-0a8d-4577-9e06-6e2e9444e80b/Untitled.png)

메시지 기능

`reason` 을 `messageSource` 에서 찾는 기능도 제공합니다. ( `reason = “error.bad”` )

`messages.properties`

```java
error.bad=Wrong Request Error. (Using Message)
```

`BadRequestException`

```java
package hello.exception.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

//@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "error.bad")
public class BadRequestException extends RuntimeException {
}
```

참고 - 편하게 message 까지 모기 위해서 [`application.properties`](http://application.properties)  에 아래 코드를 추가했습니다.

```java
server.error.include-message=always
```

메시지 사용 결과

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/775a4460-6c14-4449-bf5a-78df415c6bd1/Untitled.png)

### ResponseStatusException

`@ResponseStatus` 는 개발자가 직접 변경할 수 없는 예외에는 적용할 수 없습니다. (애노테이션을 직접 넣어야 하는데, 개발자가 코드를 수정할 수 없는 라이브러리의 예외 코드 같은 곳에는 적용할 수 없습니다.)

추가로 애노테이션을 사용하기 때문에 조건에 따라 동적으로 변경하는 것도 어렵습니다. 이 때는 `ResponseStatusException` 예외를 사용하면 됩니다.

`ApiExceptionController` - 추가

```java
@GetMapping("/api/response-status-ex2")
public String responseStatusEx2() {
    throw new ResponseStatusException(HttpStatus.NOT_FOUND, "error.bad", new IllegalArgumentException());
}
```

실행 결과

http://localhost:8080/api/response-status-ex2

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/233b3fe1-c750-4cc9-9911-d14167feb478/Untitled.png)

# 6. 스프링이 제공하는 ExceptionResolver2

이번에는 `DefaultHandlerExceptionResolver` 을 살펴봅시다.

### DefaultHandlerExceptionResolver

`DefaultHandlerExceptionResolver` 는 스프링 내부에서 발생하는 스프링 예외를 해결합니다.

대표적으로는 파라미터 바인딩 시점에서 타입이 맞지 않으면 내부에서 `TypeMismatchException` 이 발생하는데 이 경우 예외가 발생했기 때문에 그냥 두면 서블릿 컨테이너까지 오류가 올라가고, 결과적으로는 500 오류가 발생합니다.

그런데 파라미터 바인딩은 대부분 클라이언트가 HTTP 요청 정보를 잘못 호출해서 발생하는 문제입니다. HTTP 에서는 이런 경우 HTTP 상태 코드 400 을 사용하도록 되어 있습니다.

`DefaultHandlerExceptionResolver` 는 이것을 500 오류가 아니라 HTTP 상태 코드 400 오류로 변경합니다.

스프링 내부 오류를 어떻게 처리할지 수많은 내용이 정의되어 있습니다.

### 코드 확인

`DefaultHandlerExceptionResolver.handleTypeMismatch` 을 보면 다음과 같은 코드를 확인할 수 있습니다.

`response.sendError(HttpServletResponse.SC_BAD_REQUEST)` (400)

결국에 `response.sendError()` 을 통해서 문제를 해결하는 것이지요!

`sendError(400)` 을 호출했기 때문에 WAS 에서 다시 오류 페이지 `/error` 을 내부 요청합니다.

`ApiExceptionController` - 추가

```java
@GetMapping("/api/default-handler-ex")
public String defaultException(@RequestParam Integer data) {
    return "ok";
}
```

`Integer data` 에 문자를 입력하면 내부에서 `TypeMismatchException` 이 발생합니다.

실행 결과

http://localhost:8080/api/default-handler-ex

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/413f801c-dad7-4ab8-8f2a-57273702776a/Untitled.png)

실행 결과를 보면 HTTP 상태 코드가 400 인 것을 확인할 수 있습니다.

### 정리

지금까지 아래 `ExceptionResolver` 들에 대해서 알아보았습니다.

`ExceptionHandlerExceptionResolver` 는 다음 챕터에서 알아볼 것입니다!

`ResponseStatusExceptionResolver` 는 HTTP 응답 코드 변경에 사용하고,

`DefaultHandlerExceptionResolver` 는 스프링 내부 예외 처리에서 사용합니다.

HTTP 상태 코드를 변경하고, 스프링 내부 예외의 상태 코드를 변경하는 기능도 알아보았습니다.

그런데 HandlerExceptionResolver 을 직접 사용하기는 복잡합니다. API 오류 응답일 경우에 response 에 직접 데이터를 넣어야 해서 불편하고 번거롭지요. ModelAndView 을 반환해야 하는 것도 JSON 데이터 자체를 반환하는 API 에도 잘 와닿지 않지요.

스프링에서는 이러한 문제들을 해결하기 위해서 `@ExceptionHandler` 라는 매우 혁신적인 예외 처리 기능을 제공합니다. 그것이 `ExceptionHandlerExceptionResolver` 입니다.
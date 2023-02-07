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
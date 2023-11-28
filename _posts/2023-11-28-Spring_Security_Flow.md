---
layout : single
title : "Spring Security 웹 요청 처리 흐름"
categories : Spring Security
tags : [Spring Security]
toc : true
toc_sticky : true 
toc_label : "목차"
sidebar:
    nav: "counts"
---


## 웹 보안의 일반적인 흐름

안녕하세요!! 마음만은 따듯했던, 하지만 지금은 옆구리가 시려워 마음도 차가워진 남자 양재용입니다.

오늘 무엇을 만들어 와야할까 정말 많은 고민을 했습니다,, 다들 아는 것들이 많아서

제가 무엇을 알려줄 수 있을까 and 뭔가 땡기는 게 없어서,,,, ㅎㅠㅎ

그러다가 요즘 제가 OAuth2 연동하면서 필터체인을 조금 만지면서

이 부분을 좀 만들어보면 괜찮지 않을까 생각해서 만들게 되었습니다!!

들어가기에 앞서 보안이 적용된 웹 요청의 일반적인 흐름에 대해 알아보고 갑시다!!

<img src = "https://s3.ap-northeast-2.amazonaws.com/urclass-images/7YWbJIzjmV8X9N2O_Z1VX-1662791689566.png">

그림은 사용자가 보호된 리소스를 요청했을 때 실행되는 흐름입니다.

자세히 보시면 Controller와 같은 엔드포인트에 도달하기 전에 인증관리자, 접근 결정 관리자 같은 컴포넌트가

요청을 가로채 사용자의 크리덴셜과 접근 권한을 확인하는 것을 알 수 있습니다.

이 과정은 어디서 일어나는 것 일까요??

## ServletFilter와 FilterChain

빠밤~ 그것은 바로 서블릿 필터랍니다~~!! 🎉

스프링은 서블릿 기반 어플리케이션이기에 요청이 도달하기 전, 후로 서블릿 필터를 이용하여

응답과 요청에 관여할 수 있답니다!!

서블릿은 여러개의 서블릿을 이어붙여 필터 체인이라는 것을 만들 수 있는데

<img src = "https://s3.ap-northeast-2.amazonaws.com/urclass-images/dHbwtXC2SHXBKFpQLw5f5-1674611186514.png">

이런 식으로 여러개의 필터들을 묶어서 필터체인을 구성 할 수 있으며

각각의 필터들은 doFilter()를 통해 다음 필터가 실행되도록 합니다.

```java
@Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            Claims claims = verifyJws(request);
            setAuthenticationToContext(claims);
        } catch (ExpiredJwtException ee) {
            request.setAttribute("exception", ee);
        } catch (MalformedJwtException me) {
            request.setAttribute("exception", me);
        } catch (SignatureException se) {
            request.setAttribute("exception", se);
        } catch (Exception e) {
            request.setAttribute("exception", e);
        }

        filterChain.doFilter(request, response); // 다음 필터를 실행
    }
```

위에 코드는 JWT 토큰 검증을 하는 필터에서 doFilterInternal 부분을 추출한 부분입니다!!

(doFilterInternal은 Spring MVC에서 제공하는 메소드로 **HttpServletRequest, HttpServletResponse**를 사용한다)

보시면 filterChain.doFilter()를 통해 다음 필터를 실행하는 것을 볼 수 있습니다.

지금까지 저희가 서블릿 필터를 공부한 이유는 Spring Security에서 이 필터 부분을 통하여

인증, 검증 보안과 관련된 다양한 추가 작업을 실행하기 때문입니다!!

한번 자세히 알아볼까요~?~?

## Spring Security Filter

<img src = "https://s3.ap-northeast-2.amazonaws.com/urclass-images/qkev4vmV5-DMXpcv4MdBk-1674611246060.png">

자 Spring Security의 필터는 이와 같이 구성되어 있는데요(DelegatingFilterProxy, FilterChainProxy)

DelegatingFilterProxy는 서블릿 필터체인과 ApplicationContext에 Bean으로 등록된 필터들을

연결해 주는 브리지 역할을 합니다!!

으잉?? 갑자기 무슨 소리야!! 라는 말이 나오실텐데요

Spring Security에서는 자기만의 필터를 ApplicationContext에 빈으로 등록한 후에 작업을 처리하기에

DelegatingFilterProxy의 역할이 필요하게 됩니다.

본격적인 작업은 FilterChainProxy에서 실행하게 되는데요, 이미지 자료를 보면서 이해 해보도록 합시다.

<img src = "https://s3.ap-northeast-2.amazonaws.com/urclass-images/UpSQNpe1juvLRg1ohv7kY-1674611275989.png">

FilterChainProxy를 보면 여러개의 Spring Security의 보안을 처리하기 위한 여러개의 필터가 모여있습니다.

Spring Security는 보안을 위한 특정 작업을 수행하기 위한 다양한 Filter를 지원하는데 그 수가 굉장히 많이 있는데

Spring Security의 Filter가 각각 어떤 역할을 수행하는지는 전부 다 알 필요는 없으며, 

필요한 상황이 되었을 때 그때그때 적용해도 상관은 없습니다!!

그리고 Spring Security의 Filter 항상 모든 Filter가 수행되는 것이 아니라 프로젝트 구성 및 설정에 따라 일부의 Filter만 활성화되어 있기 때문에 직접적으로 개발자가 핸들링할 필요가 없는 Filter들이 대부분입니다.

따라서 개발자가 Custom Filter를 작성하고 등록할 경우 기존 필터들 사이에서 우선순위를 적용해 수행되어야 할 필요가 있는 경우에 참고해서 적용하면 됩니다.

아 그리고 Filter에서 발생하는 예외는 Spring 밖에서 실행되기 때문에 저희가 흔히 사용하는 GlobalExceptionHandler(`@RestControllerAdvice`)에  잡히지 않기 때문에 

따로 성공 핸들러, 실패 핸들러를 작성해서 추가해주셔야합니다!!

혹시 이 부분에 대해 잘 아시는 분이 있다면 자신은 어떻게 작성했는지 알려주세요!!

다들 따듯한 연말 보내봐요 ㅎㅡㅎ
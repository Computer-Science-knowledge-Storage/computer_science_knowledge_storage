## Google 소셜 로그인 [안드로이드-스프링] 연동에 대해 알아보자

### 1. 소셜로그인 원리 및 절차

#### 개요

1. 사용자가 소셜로그인 버튼을 누르면 구글 로그인 페이지로 redirect
2. 이때 이 로그인 페이지로 가게 하기 위해 android app과 google 사이에 상호작용이 일어나는데, android 개발자는 이 상호작용을 위해 미리 OAuth라는 서비스를 사용함
3. 로그인 성공 시, google은 사용자가 기존에 보던 서비스 페이지로 redirect 시켜줌

=> 로그인 중개 역할: Google

=> 중개 역할 가능하게 만드는 기술적 서비스: OAuth

즉, 사용자가 소셜로그인에 로그인 시, 사용자의 비밀번호를 android app에 주는게 아니라 OAuth를 거쳐서 android app에게 Access Token을 제공하고, android app은 이 token을 이용해 google social에 접근하여 사용자에게 로그인 페이지를 제공할 수 있는 것



#### 용어 정리

- Client: Resource Server를 사용하는 주체로, 내가 구현할 Android Application
- Resource Owner: Resource Server에 인증되는, 내가 구현한 Application 쓰는 사용자
- Resource Server: Google, Kakao같이 데이터 가진 서버



#### 절차

1. 등록

   1. Client가 Resource Server를 사용하기 위해 Google Developer 사이트 등에서 진행하는 과정

      => 이 과정을 거치면 Client와 Resource Server는 아래 3가지를 공유하게 된다.

      1. Client ID
         - 내가 구현할 application을 식별할 수 있는 ID
      2. Client Secret
         - 내가 구현할 application을 식별할 수 있는 PW (절대 노출되면 안됨)
      3. Authorized Redirect URL
         - Resource Server가 권한을 부여하는 과정에서 Authorized code를 전달해줄 경로

2. 인증

   1. Client가 Resource Owner에게 인증화면 전송

   2. Sign in with Google 버튼을 Resource Owner가 클릭 시, Resource Server는 Resource Owner가 인증되어 있는지 확인하고, 인증이 안되어 있으면 로그인 화면을 전송함

   3. (Resource Owner가 로그인 성공 시) Client가 Resource Server에게 client id 전송

   4. Resource Server가 client로 부터 받은 client id와 갖고있던 client id 확인해서 맞으면 redirect URL까지 확인

   5. Resource Server는 Resource Owner에게 scope (client가 Resource Server에서 사용할 자원들) 확인 창 띄움

   6. Resource Owner가 scope 허용 시, Resource Server는 아래 정보를 추가로 가짐

      - User id: Resource Owner를 식별할 수 있는 id (1, 2, ...)
      - Scope: Resource Owner가 허락해 접근할 수 있는 resource 목록 (B, C...)

   7. Resource Server는 Authorization code생성하고 header location에 redirect URL과 Authorzation code를 Resource Owner에게 전달. Resource Owner는 header에 세팅된 location에 의해 redirect 됨

   8. Resource Owner로부터 redirect된 URL뒤의 parameter인 Authorization code를 받은 Client는 아래와 같은 URL로 Resource Server로 직접 접근하게 됨

      ```
      https://[redirect URL]?code=[Authorization Code]
      ```

   9. Resource Server는 전달받은 code를 통해 client id, client secret, redirect URL 검사

   10. 9번 절차 끝나면 Resource Server가 Access Token을 생성하고, 중복 인증 방지를 위해 Authorization code를 제거함

3. Refresh Token

   1. Access Token의 수명은 매우 짧아 만료되면 Resource Server에 접근 불가하므로 Refresh Token을 사용해 위와 같은 인증과정을 생략한다. 

      => 즉, Access Token 만료 시 이를 다시 발행하기 위한 용도로, Access Token보다 유효기간이 더 길다 (Refresh나 Access나 둘 다 같은 내용이지만 유효 기간이 차이나는 것)

      => 세션을 유지한다는 말은 만료되지 않는 Access 혹은 Refresh Token 값을 android에서 갖고 있다는 말 (Token을 갖고 있어야 서버에서 Token 만료 여부를 확인하고 만료가 안됐으면 유효한 값들을 보내는 과정이 구현되어야 한다.)

      => API 를 사용하기 위해 android에서 access token을 보냈는데 401 (UnAuthorized error)를 받았다면 Refresh Token을 다시 보내도록 request 설정.. 즉 API 두 번 호출하는 것으로 해결가능하다고 한다. 

      => Refresh Token을 DB에 저장해야 함!!



### 2. 구현

예시 백엔드 로그인 통신 로직

1. 로그인 성공 시

   1. Access Token 및 Refresh Token 발행
   2. Authorization Header에 Access Token 혹은 Refresh Token을 세팅해 API 호출

2. 결과 처리

   2.1 Aceess Token이 유효한 경우

   => API 결과 반환 {Access Token, Access Token expire at, Refresh Token, Refresh Token expire at}을 response로 반환

   2.2 Access Token / Refresh Token이 만료된 경우

   => 401 Error

   2.3 Acess Token / Refresh Token이 변조된 경우

   => 403 Error

   2.4 Access Token이 없는 경우 (1.2에서 token 세팅 안한 경우)

   => 401 Error

   2.5 Refresh Token이 유효한 경우

   => Authorization Header에 Access Token 세팅 및 API 결과 반환

   => Access Token이 만료된 경우 Refresh Token을 넘겼을 때 Access Token 갱신

3. Refresh Token 처리

   1. Refresh Token의 요청 시간과 DB에 저장된 Refresh Token의 만료 시간과 비교하여 72시간 미만으로 차이나면 서버에서 자동으로 Refresh Token의 만료 시간 갱신

<img src="/Users/soeunlee/Library/Application Support/typora-user-images/image-20220723231241323.png" alt="image-20220723231241323" style="zoom: 50%;" />



### Reference

https://velog.io/@hyunju-song/%EC%86%8C%EC%85%9C%EB%A1%9C%EA%B7%B8%EC%9D%B8%EA%B8%B0%EB%8A%A51OAuth

https://velog.io/@parkoon/OAuth-%EC%99%84%EB%B2%BD%ED%95%98%EA%B2%8C-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0

https://velog.io/@mraz3068/Refresh-Token%EC%9D%B4-%ED%95%84%EC%9A%94%ED%95%9C-%EC%9D%B4%EC%9C%A0
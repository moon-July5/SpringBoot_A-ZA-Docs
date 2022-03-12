![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
바로 전에 구현했던 기능은 회원가입 후, 이메일 인증 완료 후 자동 로그인 기능과 사용자 정보를 참조하는 애너테이션을 생성했습니다.  
이를 이용하여 회원가입 후 이메일 인증을 하도록 유도하는 메시지를 출력하도록 하도,  
이메일 인증 메일을 재전송하도록 구현해보도록 하겠습니다.  

`localhost:8080/` 경로는 홈이며, 전에 `@CurrentMember`를 이용하여 홈 경로에 사용자 정보를 참조하도록 했습니다.   
이를 이용하도록 하겠습니다.  

## 홈(home.html) 수정
```html
  ...생략
  <div class="home">
        <div class="alert alert-warning" role="alert" style="margin : 60px 0px 0 200px;" th:if="${member != null && !member.isValid()}">
                        회원 가입을 완료하려면 <a th:href="@{/member/email-check}" class="alert-link">계정 인증 이메일을 확인</a>하세요.
        </div>
        <div class="py-5 text-center" style="margin : 60px 40px 0 200px;">
               <h2>A-ZA Home!!</h2>
        </div>           
   </div>
  ...생략
```
회원가입 후 자동로그인 되면서 홈 화면으로 이동하게 됩니다. 그때 `Thymeleaf` 문법을 이용하여 이메일 인증을 유도하도록 수정했습니다.  
그러면 아래와 같이 메시지가 나타나는 것을 확인할 수 있습니다.  
![email-4](https://user-images.githubusercontent.com/60730405/158018070-4dc3e4e6-9a85-4d75-8b83-eb8bf72d2be5.JPG)  

이제 `계정 인증 이메일을 확인`을 클릭하게 되면 나타나는 페이지를 구현해보도록 하겠습니다.  

## 계정 인증 이메일 확인(email-check.html) 페이지 구현
```html
  ...생략
<div class="email__Auth">
    <div class="py-5 text-center" th:if="${error != null}">
          <p class="email-check" style="font-weight: bold;">A-ZA 회원가입</p>
          <div  class="alert alert-danger" role="alert" th:text="${error}"></div>
          <p class="email-check" th:text="${email}"></p>
    </div>
        
    <div class="py-5 text-center" th:if="${error == null}">
          <p class="email-check" style="font-weight: bold;">A-ZA 회원가입</p>
        
          <h2>회원가입을 완료하기 위해 인증 이메일을 확인하세요.</h2>
        
          <div>
               <p class="email-check" th:text="${email}"></p>
               <a class="btn btn-outline-info" th:href="@{/member/email-resend}">인증 이메일 다시 보내기</a>
          </div>
 </div>
  ...생략
```
에러가 존재하면 에러를 출력합니다. 그게 아니라면 이메일 인증을 확인하라는 메시지가 나타납니다.  
이 페이지는 인증이 된 상태에서만 접근해야 하기 때문에 `SecurityConfig`에서 아무나 접근을 허용하도록 하면 안됩니다.  

이제, 이 페이지가 나타나도록 `MemberController`에서 추가하고 이메일 인증 메일을 다시 보낼 수 있도록 `/member/email-resend` 경로로  
요청이 올 때 기능을 구현합니다.  

## MemberCOntroller 수정
```java
package com.moon.aza.controller;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.MemberService;
import com.moon.aza.support.CurrentMember;
import com.moon.aza.validator.SignUpFormValidator;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.Errors;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;

import javax.mail.MessagingException;
import javax.validation.Valid;
import java.util.Map;

@Log4j2
@RequiredArgsConstructor
@Controller
public class MemberController {
  ...생략

    /* 이메일 인증 페이지*/
    @GetMapping("/member/email-check")
    public String emailCheck(@CurrentMember Member member, Model model){
        model.addAttribute("email", member.getEmail());
        return "/aza/email-check";
    }
    /* 이메일 재전송 */
    @GetMapping("/member/email-resend")
    public String emailResend(@CurrentMember Member member, Model model) throws MessagingException {
        if(!member.enableToSendEmail()){
            model.addAttribute("error", "인증 이메일은 5분에 한 번만 전송할 수 있습니다.");
            model.addAttribute("email", member.getEmail());
            return "/aza/email-check";
        }
        memberService.sendVerificationEmail(member);
        return "redirect:/";
    }
    ...생략

}
```
이메일 인증을 확인하라는 메시지 링크를 클릭시 나타나는 경로는 `/member/email-check`이며, 사용자의 이메일을 나타낼 수 있도록  
`@CurrentMember`를 이용하여 인증된 사용자의 `email` 정보를 넘겨줍니다.  

`인증 이메일 다시 보내기`를 클릭 시 요청하는 경로는 `/member/email-resend`로 악용하지 못하도록 5분에 한 번만 보낼 수 있도록 기능을 구현합니다.  
다시 보낼 수 있는 시간이면 이메일을 보내는 메서드인 `sendVerificationEmail()`를 호출합니다.  

## Member 수정
사용자가 이메일을 보낼 수 있는지 확인하는 메서드인 `enableToSendEmail()`를 `Member`에 구현합니다.  

```java
package com.moon.aza.entity;

import lombok.*;

import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.UUID;

@ToString
@Builder
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member extends BaseEntity {
    
    ...생략
  
    // 이메일 토큰 발급 시간 저장
    private LocalDateTime emailTokenGeneratedAt;


    // 이메일을 보낸 지 5분이 지났는지 확인
    public boolean enableToSendEmail(){
        return this.emailTokenGeneratedAt.isBefore(LocalDateTime.now().minusMinutes(5));
    }
}

```
`isBefore()` 메서드를 이용하여 지금 현재 시간보다 토큰 발급 시간이 5분 전이면 true를 return하도록 합니다.  


## 결과
`계정 인증 이메일을 확인` 링크를 클릭 시 나타나는 페이지  
![email-5](https://user-images.githubusercontent.com/60730405/158018717-045ba565-ed80-4302-92ce-39be9db955ba.JPG)  

`인증 이메일 다시 보내기`를 5분이 지나기 전에 클릭시 나타나는 경고 메시지  
![email-6](https://user-images.githubusercontent.com/60730405/158019699-b8002c3a-b499-47be-a5cc-5c0bf34a7061.JPG)

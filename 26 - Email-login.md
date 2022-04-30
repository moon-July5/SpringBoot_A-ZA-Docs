![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***

이번에 구현할 것은 비밀번호를 잊어버렸을 때, 회원가입을 위해 작성했던 이메일을 통해 로그인한 후에  
비밀번호 변경을 유도하는 기능을 구현해 보겠습니다.  

일단 `Member` Entity에 메서드 하나를 추가하겠습니다.

## Member Entity 
```java
package com.moon.aza.entity;

import lombok.*;
import org.hibernate.Hibernate;

import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

@ToString
@Builder
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member extends BaseEntity {
  ...생략
    public boolean isValid(String token){
        return this.emailToken.equals(token);
    }

}

```
`Member` Entity에 `isValid()` 메서드를 구현합니다.  
이것은 나중에 회원가입할 때 작성한 이메일에 메일을 전송할 것인데, 그 메일의 내용에는 링크가 하나 주어지며, 그 링크를 클릭 시 자동으로 로그인이 되도록 구현할 것입니다.  
그때 링크 주소에는 `이메일 토큰`이 포함되어 있는데, `Database에 저장되어 있는 이메일 토큰`과 `링크 주소의 이메일 토큰`이 일치하는지 확인하는 메서드입니다. 비교하여 같다면 True를, 아니면 False를 반환합니다.  

## MemberService
```java
package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import com.moon.aza.support.UserMember;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.validation.Errors;
import org.springframework.validation.FieldError;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

@RequiredArgsConstructor
@Transactional
@Service
public class MemberService implements UserDetailsService {
    private final MemberRepository memberRepository;
    private final JavaMailSender mailSender;
    private final PasswordEncoder passwordEncoder;
    @Value("${spring.mail.username}")
    private String from;

    public void sendLoginLink(Member member) throws MessagingException {
        member.generateToken();
        // 메일 보내기
        MimeMessage mailMessage = mailSender.createMimeMessage();
        MimeMessageHelper messageHelper = new MimeMessageHelper(mailMessage,"UTF-8");
        StringBuilder body = new StringBuilder();
        body.append("<html> <body> <span style=\"font-size:16px; font-weight:bold;\">");
        body.append(member.getNickname()+"님</span> 안녕하세요.<br>");
        body.append("<p> 이메일을 통해 로그인을 하고 싶으시면 아래의 링크를 클릭하시길 바랍니다. </p><br>");
        body.append("<a href=\"http://localhost:8080/member/login-by-email?token="
                +member.getEmailToken()+"&email="+member.getEmail()+"\">");
        body.append("로그인하기</a></body></html>");

        messageHelper.setFrom(from); // 보내는 사람
        messageHelper.setTo(member.getEmail()); // 받는 사람
        messageHelper.setSubject("A-ZA 로그인 링크"); // 메일 제목
        messageHelper.setText(body.toString(),true); // 메일 내용

        mailSender.send(mailMessage);

    }
    
  ...생략
    
}

```
이메일을 통해 로그인을 위한 링크를 보내기 위한 `sendLoginLink()`를 구현합니다.  
이것은 사실 저번에 이메일 인증을 보내는 메서드인 `sendVerificationEmail()`와 매우 흡사합니다.  
차이점은 `generateToken()` 메서드로 이메일 토큰을 생성하고 사용자에게 메일을 보냅니다.  

## MemberController
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
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import javax.mail.MessagingException;
import javax.validation.Valid;
import java.util.Map;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/member")
@Controller
public class MemberController {
    private final SignUpFormValidator signUpFormValidator;
    private final MemberService memberService;

    /*이메일 로그인 페이지*/
    @GetMapping("/email-login")
    public String emailLogin(){
        log.info("/member/email-login");
        return "/aza/email-login";
    }
    @PostMapping("/email-login")
    public String sendLinkForEmailLogin(String email, Model model, RedirectAttributes redirectAttributes) throws MessagingException{
        Member member = memberService.findMemberByEmail(email);
        if(member==null){
            model.addAttribute("error","유효한 이메일 주소가 아닙니다.");
            return "/aza/email-login";
        }
        if(!member.enableToSendEmail()){
            model.addAttribute("error","이메일 전송은 5분에 한 번만 전송할 수 있습니다.");
            return "/aza/email-login";
        }
        memberService.sendLoginLink(member);
        redirectAttributes.addFlashAttribute("message","로그인 가능한 링크를 성공적으로 이메일로 전송하였습니다.");
        return "redirect:/member/email-login";
    }
    @GetMapping("/login-by-email")
    public String loginByEmail(String token, String email, Model model){
        Member member = memberService.findMemberByEmail(email);
        if(member==null || !member.isValid(token)){
            model.addAttribute("error","로그인 할 수 없습니다.");
            return "/aza/logged-in-by-email";
        }
        memberService.login(member);
        return "/aza/logged-in-by-email";
    }

  ...


}
```
`MemberController`에는 로그인 화면에 `이메일로 로그인하기` 버튼을 클릭 시 `/member/email-login`으로 요청하여 이동할 페이지를 `@GetMappin`을 통해 구현합니다.  

그리고 사용자 이메일을 입력하고 전송할 때 `/member/email-login`경로의 `POST` 방식으로 요청이 올 때, 호출한 메서드 `sendLinkForEmailLogin()`를 구현합니다.  
이 메서드는 사용자의 이메일을 전달받아 Database에서 그 사용자가 있는지 확인합니다. 사용자가 존재하지 않거나 전송한 지 5분도 지나지 않아 또 다시 전송할 시 error를 담아 전달합니다.  

성공적으로 전송이 완료되었으면 사용자가 작성한 이메일에 메일이 도착했을 것이며, 이 메일의 내용에는 링크를 클릭 시 `/member/login-by-email` 경로로 요청이 가며,  
parameter로 이메일 토큰과 사용자 이메일을 전달합니다. 이메일을 조회하여 사용자가 존재하지 않거나 실제 Database의 이메일 토큰과 일치하지 않으면 error를 담아 전달하도록 합니다.  
error가 없다면 `login()` 메서드를 통해 자동으로 로그인하도록 하며, 비밀번호를 변경하도록 유도합니다.  

## 로그인(login.html) 페이지 수정
```html
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{/layout/basic}">
    <head>
        <link rel="stylesheet" th:href="@{/css/login.css}">
    </head>
    <th:block layout:fragment="content">
        <div class="login">
            <div class="login__top">
                <h1>로그인</h1>
            </div>
            <div th:if="${param.error}" class="ui-icon-alert alert-danger" role="alert">
                <p class="text-center">로그인 정보가 정확하지 않습니다.</p>
            </div>
            <form th:action="@{/login}" class="needs-validation" method="post">
                <div class="form-group">
                    <label style="font-weight:bold; margin-bottom:5px;">이메일</label>
                    <input type="text" class="form-control" name="username" placeholder="이메일을 입력해주세요">
                </div><br>
                <div class="form-group">
                    <label style="font-weight:bold; margin-bottom:5px;">비밀번호</label>
                    <input type="password" class="form-control" name="password" placeholder="비밀번호를 입력해주세요" required>
                </div>
                <div class="form-group">
                    <input type="checkbox" class="form-check-input" id="rememberMe"  name="remember-me" checked> 
                    <label class="form-check-label" for="rememberMe">로그인 유지</label>
                </div><br>
              <!-- 추가 -->
                <div class="form-group">
                    <small id="forgotPasswordHelp" class="form-text text-muted">
                        패스워드를 잊어버렸다면? <a th:href="@{/member/email-login}">이메일로 로그인하기</a>
                    </small>
                </div><br>
              <!-- 추가 -->
                <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}">홈 화면으로</button>
                <button type="submit" class="btn btn-primary loginButton"><img class="button__icon" th:src="@{/imgs/login-icon-white.png}">로그인</button>
                <button type="button" th:onclick="|location.href='@{/signup}'|" class="btn btn-info" style="color: white;"><img class="button__icon" th:src="@{/imgs/signup-icon-white.png}">회원가입</button>
            
            </form>
        </div>
    </th:block>
</html>
```
비밀번호를 잊어버렸을 경우, `이메일로 로그인하기`를 클릭 시 `/member/email-login`으로 요청하도록 구현합니다.  

## 이메일 로그인(email-login) 페이지 구현
```html
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{/layout/basic}">
    <th:block layout:fragment="content">
        <div class="email-login" style="margin: 60px 40px 0 250px; padding: 20px 0 0 20px;">
            <div class="email-login__top" style="font-family: 'cookie'; font-size: 1.8rem; margin-bottom: 15px; display: flex; justify-content: space-between;">
                <h1>패스워드 없이 이메일로 로그인하기</h1>
            </div>
            <div th:if="${error}" class="alert alert-danger alert-dismissible fade show mt-3" role="alert">
                <span th:text="${error}">error</span>
                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
            </div>  
            <div th:if="${message}" class="alert alert-info alert-dismissible fade show mt-3" role="alert">
                <span th:text="${message}">success</span>
                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
            </div>
            <form class="needs-validation col-sm-6" th:action="@{/member/email-login}" method="post" novalidate>
                <div class="form-group mb-3">
                    <label for="email">가입 할 때 사용한 이메일</label>
                    <input id="email" type="email" name="email" class="form-control"
                           placeholder="example000@email.com" aria-describedby="emailHelp" required>
                    <small id="emailHelp" class="form-text text-muted">
                        가입할 때 사용한 이메일을 입력하세요.
                    </small>
                    <small class="invalid-feedback">이메일을 입력하세요.</small>
                </div>
    
                <div class="form-group">
                    <button class="btn btn-success btn-block" type="submit" aria-describedby="submitHelp">
                        로그인 링크 보내기
                    </button>
                </div>
            </form>
        </div>
    </th:block>
</html>
```
사용자 이메일을 입력받도록 구현하며, 유효한 이메일 주소가 아니거나 이메일 전송을 한 지 5분이 지나지 않아 재전송할 시 error가 나오도록 합니다.  
또한 성공적으로 이메일 전송이 완료되면 성공했다는 알림이 나오도록 합니다. 

## 이메일 링크 클릭 시 (logged-in-by-email.html) 페이지 구현
```html
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{/layout/basic}">

    <th:block layout:fragment="content">
        <div class="logged-in-by-email" style="margin: 60px 40px 0 250px; padding: 20px 0 0 20px;">
            <div class="py-5 text-center" th:if="${error}">
                <p class="logged-email" style="font-weight: bold;">이메일 로그인</p>
                <div  class="alert alert-danger" role="alert" th:text="${error}">
                    로그인할 수 없습니다.
                </div>
            </div>
        
            <div class="py-5 text-center" th:if="${error == null}">
                <p class="logged-email" style="font-weight: bold;">이메일 로그인</p>
                <h3>이메일로 로그인 했습니다. <a th:href="@{/settings/password}">패스워드를 변경</a>하세요.</h3>
            </div>
        </div>
    </th:block>
</html>
```
유효한 이메일이 아니거나 토큰이 일치하지 않을 시 error를 출력하도록 구현하며,  
error가 없다면 자동으로 로그인이 되고 패스워드를 변경하도록 유도합니다.  

## 결과
1. `로그인 페이지(login.html)`에서 링크 확인  
![1](https://user-images.githubusercontent.com/60730405/166094094-49ede907-6199-43dc-9c01-8a023cc7aba2.png)

2. `이메일로 로그인하기` 클릭 시 나타나는 페이지(email-login.html)  
![2](https://user-images.githubusercontent.com/60730405/166094093-90d757f2-19ab-4b85-8e5c-900a600625b3.png)  

3. 사용자 이메일 작성  
![3](https://user-images.githubusercontent.com/60730405/166094092-2540e7d0-2ae5-4be6-b0d5-b341edfdbb1a.JPG)

4. 이메일 링크 확인
![4](https://user-images.githubusercontent.com/60730405/166094089-16f8f91f-c78f-4a56-b6d3-a023655a8b38.png)  

5. `이메일 링크(/member/logged-in-by-email)` 클릭 시 나타나는 페이지(logged-in-by-email.html)
![5](https://user-images.githubusercontent.com/60730405/166094085-b593e7a1-6fae-4845-a8f6-3dcb7b50654f.png)

6. 패스워드 변경 페이지 유도
![6](https://user-images.githubusercontent.com/60730405/166094082-2abb4bf2-f494-44ff-866b-d7ef0625afbe.JPG)

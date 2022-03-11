![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 할 것은 회원가입할 때 작성했던 이메일에 회원가입 인증을 위한 메일을 전송할 것입니다.  
전송된 메일의 내용에는 인증 링크가 있으며, 이 인증 링크는 Database에 저장된 `emailToken`과 `email`이 포함되어 있습니다.  
해당 링크를 클릭했을 때 일치하면 가입 완료 처리를 합니다.  

그전에 저같은 경우는 회원가입 사용자에게 이메일을 전송하기 위해 `NAVER`를 사용했으며, `NAVER`에서 메일에 들어가서  
`환경설정` -> `POP3/IMAP 설정`에서 SMTP 설정을 해줘야 합니다.  

## build.gradle 의존성 추가
```
dependencies {
  ... 생략
	implementation 'org.springframework.boot:spring-boot-starter-mail'
  ... 생략
}
```
`spring-boot-starter-mail` 패키지는 이메일과 관련된 프로세스를 담당합니다.  

## application.properties 정보 추가
```
# email Authentication(naver)
spring.mail.host = smtp.naver.com
spring.mail.port = 465
spring.mail.protocol = smtp
spring.mail.username = NAVER 이메일
spring.mail.password = NAVER 비밀번호
spring.mail.properties.mail.smtp.auth = true
spring.mail.properties.mail.transport.protocol = smtp
spring.mail.properties.mail.smtp.ssl.trust=smtp.naver.com
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.ssl.enable = true
```
mail 전송을 위해서 `application.properties`파일에 정보를 작성합니다.  
`username`과 `password`는 본인의 NAVER 계정을 작성해야 합니다.  

## Member 수정
`Member` entity에서 필드 `emailToken`이 발급한 시기를 저장할 있는 필드변수를 하나 더 추가시켜야 합니다.  
왜냐하면 나중에 이메일 토큰을 다시 발급할 수 있는 상황이 올 수 있는데, 이때 계속 발급할 수 없게 시간 텀을 둬서 발급하기 위함입니다.  

그리고 이메일 인증 완료 후 인증 날짜를 저장할 수 있는 필드 변수 하나를 더 추가시키고 필드 변수 `isValid`를 함께 갱신해주는 메서드를 구현합니다.  
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
    private LocalDateTime joinedAt; // 추가

    private LocalDateTime emailTokenGeneratedAt; // 추가

    // 이메일 인증 완료 후
    public void verified(){
        this.isValid = true;
        joinedAt = LocalDateTime.now();
    }

    public void generateToken(){
        this.emailToken = UUID.randomUUID().toString();
        this.emailTokenGeneratedAt = LocalDateTime.now();
    }
}
```

다음에 할 것은 `MemberService`를 수정하는 것입니다.  

## MemberService 수정 
```java
package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
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
public class MemberService {
    private final MemberRepository memberRepository;
    private final JavaMailSender mailSender; // 추가
    private final PasswordEncoder passwordEncoder;
    
    // 추가
    @Value("${spring.mail.username}")
    private String from;
    
    ...생략

    /* 회원정보 & 이메일 인증 보내기(실질적으로 컨트롤러에서 호출)  */
    public Member signUp(SignUpForm signUpForm) throws MessagingException {
        Member newMember = saveMember(signUpForm);
        // 이메일 인증용 토큰 생성
        newMember.generateToken();
        sendVerificationEmail(newMember);
        return newMember;
    }


    /* 이메일 인증 보내기 */
    public void sendVerificationEmail(Member member) throws MessagingException {
        // 메일 보내기
        MimeMessage mailMessage = mailSender.createMimeMessage();
        MimeMessageHelper messageHelper = new MimeMessageHelper(mailMessage,"UTF-8");

        StringBuilder body = new StringBuilder();
        body.append("<html> <body> <span style=\"font-size:16px; font-weight:bold;\">");
        body.append(member.getNickname()+"님</span> 안녕하세요.<br>");
        body.append("<p> 회원가입 완료를 위해 아래 이메일 인증을 해주시길 바랍니다.</p><br>");
        body.append("<a href=\"http://localhost:8080/member/email-check-token?token="
                +member.getEmailToken()+"&email="+member.getEmail()+"\">");
        body.append("이메일 인증 하기</a></body></html>");

        messageHelper.setFrom(from); // 보내는 사람
        messageHelper.setTo(member.getEmail()); // 받는 사람
        messageHelper.setSubject("A-ZA 회원 가입 인증"); // 메일 제목
        messageHelper.setText(body.toString(),true); // 메일 내용, true = html 형식

        mailSender.send(mailMessage);
    }
    // 이메일 인증 완료 되었다는 표시
    public void verify(Member member){
        member.verified();
    }
    // 사용자 이메일로 회원 조회
    public Member findMemberByEmail(String email){
        return memberRepository.findByEmail(email);
    }
  ...생력
}
```
`MailSender` 인터페이스를 상속받은 `JavaMailSender`를 사용하여 이메일을 전송하도록 합니다.  

이메일 인증을 보내는 기능을 하는 `snedVerificationEmail()` 메서드를 구현합니다.  
`MimeMessage` 객체를 생성하여 메일을 보냅니다. `SimpleMailMessage`는 단순한 텍스트 데이터만 보낼 수 있기때문에  
저는 `MimeMessage`와 `MimeMessageHelper`를 사용하여 html 형식으로 보냈습니다. 메일의 내용은 가입한 `닉네임`과 `인증 링크`가 있습니다.  
인증 링크는 Database에 저장된 `email`과 `emailToken`이 포함되어 있어 이 링크를 클릭하게 되면 이메일 토큰이 일치하는지 확인 후 인증이 완료됩니다.  
그리고 메일의 정보(보내는 사람, 받는 사람, 제목)도 포함하여 보내도록 합니다.  
보내는 사람(setFrom)에는 `application.properties`에 저장된 이메일을 사용합니다.  

`verify()`는 전송된 메일의 인증 링크를 클릭하게 되면 발생되는 메서드입니다.  
아까 `Member`에 구현한 `verified()` 메서드로 인증 날짜와 `isValid` 변수를 갱신해줍니다.  

`findMemberByEmail()` 메서드는 이메일로 사용자 정보를 조회하는 기능을 합니다.  

마지막으로 회원가입 로직인 `signup()` 메서드에 이메일을 보내는 메서드인 `sendVerificationEmail()`를 추가합니다.  

전송된 메일에 인증 링크를 클릭하게 되면 `/member/email-check-token`으로 요청을 하게 됩니다.  
이제 `MemberController`에서 이 요청을 처리하는 로직을 구현하겠습니다.  

## MemberRepository 수정
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Member;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
  ...생략

    public Member findByEmail(String email);
}
```
인증 링크에 포함된 `email`로 사용자를 조회하기 위해 `MemberRepository`에 `findByEmail()` 메서드를 추가합니다.  


## MemberController 수정  
```java
package com.moon.aza.controller;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.MemberService;
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
    private final SignUpFormValidator signUpFormValidator;
    private final MemberService memberService;
  
  ...생략

    // 이메일 인증 확인
    @GetMapping("/member/email-check-token")
    public String verifyEmail(String token, String email, Model model){
        Member member = memberService.findMemberByEmail(email);
        log.info("token : "+token);
        log.info("email : "+email);
        if(member == null){ // 계정정보가 없으면
            model.addAttribute("error", "wrong.email");
            return "/aza/email-verify";
        }
        if (!token.equals(member.getEmailToken())) {
            model.addAttribute("error", "wrong.token");
            return "/aza/email-verify";
        }

        memberService.verify(member);
        model.addAttribute("nickname",member.getNickname());
        return "/aza/email-verify";
    }
  ...

}
```
전송된 메일에 인증 링크를 클릭했을 때 요청하는 메서드인 `verifyEmail()`를 구현합니다.  
`MemberRepository`에 구현한 `findByEmail()`메서드로 인증 링크에 포함된 이메일로 등록된 사용자를 조회합니다.  
사용자의 계정이 없거나 이메일 토큰이 일치하지 않으면 에러를 발생시켜 이메일 인증 확인을 표시하는 페이지로 에러를 보냅니다.  
정상적으로 인증이 완료되면 사용자의 닉네임을 `Model`에 담아서 이메일 인증이 완료되었다는 페이지로 리다이렉트를 합니다.  

## 이메일 인증 확인(email-verify.html) 구현
```html
  ...생략
  <div class="py-5 text-center" style="margin : 60px 0px 0 200px;" th:if="${error}">
            <p class="email-verify" style="font-weight: bold;">A-ZA 이메일 확인</p>
            <div class="alert alert-danger" role="alert">
                이메일 확인 링크가 정확하지 않습니다.
            </div>
        </div>
        <div class="py-5 text-center" style="margin : 60px 0px 0 200px;" th:if="${error == null}">
            <p class="email-verify" style="font-weight: bold;">A-ZA 이메일 확인</p>
            <h3>
                이메일을 확인했습니다. <span style="font-weight: bold;" th:text="${nickname}"></span>님 회원가입을 축하합니다.
            </h3><br>
            <small class="alert alert-info">이제부터 가입할 때 사용한 이메일과 패스워드를 통해 로그인할 수 있습니다.</small>
        </div>
  ...생략
```
역시 design 측면은 부트스트랩을 이용하였습니다.  
여기서 `thymeleaf` 문법인 `th:if`를 이용하여 error가 존재 유무에 따라 나오는 내용이 다릅니다.  
에러가 존재하면 에러메시지가 출력되고 존재하지 않으면 회원가입 축하메시지가 출력이 됩니다.  
이제 결과를 확인해 보겠습니다.  

## 결과 
회원가입(signup.html)페이지에서 회원정보를 작성 후 회원가입버튼을 누르게 되면,   
자신이 `application.properties`에서 설정한 이메일로 전송된 것을 확인할 수 있습니다.  
![email-1](https://user-images.githubusercontent.com/60730405/157884622-112a7159-664c-4e87-9656-7d75dac25be3.JPG)  

이제 `이메일 인증 하기`를 클릭하게 되면 아래와 같은 이미지처럼 이메일 인증이 완료됩니다.  
![email-2](https://user-images.githubusercontent.com/60730405/157884929-f437fe63-2740-483b-8cb6-7c86f84d926a.JPG)  

만약, 계정정보가 없거나 인증 토큰이 일치하지 않으면 아래와 같이 에러 메시지가 출력이 됩니다.  
![email-3](https://user-images.githubusercontent.com/60730405/157885243-3ca1cba1-e366-4cd4-95b5-bed60ff7f526.JPG)  

***
다음에는 자동로그인과 이메일 인증 메시지를 재전송하는 기능을 구현해보겠습니다.  



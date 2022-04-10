![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
저번에는 간단한 게시글 검색 기능을 구현하였습니다.  
이번에는 자신의 `닉네임(Nickname)`과 `패스워드(Password)`를 변경할 수 있게하는 기능을 구현해보도록 하겠습니다.  
그 중에서 일단 `닉네임(Nickname)`을 변경하는 기능부터 구현하겠습니다.  

## NicknameForm 구현
이전에 회원가입을 위해 정보를 전달받을 `SignUpForm` 클래스를 구현했듯이,  
이번에도 `닉네임(Nickname)`을 전달받을 'NicknameForm` 클래스를 구현합니다.  
```java
package com.moon.aza.dto;

import lombok.AccessLevel;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;


@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Data
public class NicknameForm {
    @NotBlank(message = "닉네임은 필수 입력 값입니다.")
    private String nickname;

    public NicknameForm(String nickname){
        this.nickname = nickname;
    }
}

```
생성자는 자신의 현재 닉네임을 전달해주기 위해 사용합니다.  

## NicknameFormValidator 구현
변경할 `닉네임(Nickname)`이 유효한지 검사하기 위해 구현합니다.  
바꾸려고 하는 `닉네임(Nickname)`이 다른 사용자가 사용하는지 중복 검사를 할 필요가 있습니다.  
```java
package com.moon.aza.validator;

import com.moon.aza.dto.NicknameForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;

@RequiredArgsConstructor
@Component
public class NicknameFormValidator implements Validator {
    private final MemberRepository memberRepository;

    @Override
    public boolean supports(Class<?> clazz) {
        return NicknameForm.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        NicknameForm nicknameForm = (NicknameForm) target;
        Member member = memberRepository.findByNickname(nicknameForm.getNickname());
        if(member != null){
            errors.rejectValue("nickname", "wrong.value","이미 사용중인 닉네임입니다.");
        }
    }
}

```
`@Component`를 사용하여 Bean으로 등록하고 객체의 유효성 검증을 위한 `Validator`를 상속받습니다.  
어떤 객체에 대해 검사할지 `supports()`를 재정의하고 `validate()`로 실제 Database에 동일한 `닉네임(Nickname)`이 있는지 검사합니다.  
이미 존재하는 경우 에러 메시지를 전달하도록 합니다.  

이제 실제 `닉네임(Nickname)`을 변경하기 위해 `MemberService`에 동작하도록 구현합니다.  
그전에 `Member` Entity에 변경 메서드를 추가합니다.  

## Member Entity
```java
package com.moon.aza.entity;

import lombok.*;

import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@ToString
@Builder
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member extends BaseEntity {
  
  ...생략

    public void changeNickname(String nickname){
        this.nickname = nickname;
    }
   

}

```
`닉네임(Nickname)`을 변경할 수 있도록 `changeNickname()` 메서드를 추가합니다.  

이제 `MemberService`에 `닉네임(Nickname)`을 변경하도록 구현합니다.  

## MemberService 구현
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

  
    public void changeNickname(Member member, String nickname){
        member.changeNickname(nickname);
        memberRepository.save(member);
        login(member);
    }
    
  ...생략
}
```

`Member` Entity에 구현한 `changeNickname()` 메서드를 호출하여 `닉네임(Nickname)` 필드를 변경합니다.  
그리고 수정된 필드를 `MemberRepository`에 저장합니다.  
여기서 변경 후에 `login()` 메서드를 호출하여 자동 로그인을 하도록 했는데, 이것은 해당 `닉네임(Nickname)`이 변경되지 않을 수 있어  
재로그인을 하여 정보를 갱신하도록 합니다.  

## SettingController 구현
개인정보 설정을 위한 `SetiingController`를 구현합니다.  
`NicknameForm` 전달과 객체 검증, 닉네임 변경 메서드를 구현합니다.  

```java
package com.moon.aza.controller;

import com.moon.aza.dto.NicknameForm;
import com.moon.aza.dto.PasswordForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.MemberService;
import com.moon.aza.support.CurrentMember;
import com.moon.aza.validator.NicknameFormValidator;
import com.moon.aza.validator.PasswordFormValidator;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.Errors;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.InitBinder;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import javax.validation.Valid;
import java.util.Map;

@RequiredArgsConstructor
@Controller
public class SettingController {
    private final NicknameFormValidator nicknameFormValidator;
    private final MemberService memberService;

    // 닉네임 변경
    @GetMapping("/settings/nickname")
    public void nicknameForm(@CurrentMember Member member, Model model){
        model.addAttribute(member);
        model.addAttribute(new NicknameForm(member.getNickname()));

    }
    // 닉네임 변경
    @PostMapping("/settings/nickname")
    public String changeNickname(@CurrentMember Member member, @Valid NicknameForm nicknameForm,
                                 Errors errors, Model model, RedirectAttributes redirectAttributes){
        if(errors.hasErrors()){
            model.addAttribute(member);
            return "/settings/nickname";
        }
        memberService.changeNickname(member, nicknameForm.getNickname());
        redirectAttributes.addFlashAttribute("message","닉네임을 수정하였습니다.");
        return "redirect:/settings/nickname";
    }

    @InitBinder("nicknameForm")
    public void nicknameFormInitBinder(WebDataBinder webDataBinder){ webDataBinder.addValidators(nicknameFormValidator);}

}

```
닉네임의 유효성 검증을 위한 `nicknameFormInitBinder()`를 구현합니다.  
그리고 `/setting/nickname` 경로로 `GET`과 `POST` 방식으로 요청이 올 시, 동작하는 메서드를 구현합니다.  
`GET` 방식이면 뷰(View)를 요청하면서 현재 계정정보와 `NicknameForm`에 현재 자신의 `닉네임(Nickname)`을 전달합니다.  
`POST` 방식이면 `닉네임(Nickname)`을 수정하는 동작이 발생하며, 에러 발생시 에러 메시지를 전달합니다.  

이제 `닉네임(Nickname)` 수정을 위한 뷰(View)를 구현합니다.  

## 닉네임 수정(nickname.html) 페이지 구현
```html
  ...생략
        <div class="nickname" style="margin: 60px 40px 0 250px; padding: 20px 0 0 20px;">
            <div th:if="${message}" class="alert alert-info alert-dismissible fade show mt-3" role="alert">
                <span th:text="${message}">수정 완료</span>
                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
            </div>  
            <div class="nickname__top" style="font-family: 'cookie'; font-size: 1.8rem; margin-bottom: 15px; display: flex; justify-content: space-between;">
                <h1>닉네임 변경</h1>
            </div>
            <form class="needs-validation col-12 mb-3" th:object="${nicknameForm}" th:action="@{/settings/nickname}" method="POST">
                <div class="mb-3">
                    <lable for="nickname" class="form-label" hidden>닉네임</lable>
                    <input type="text" th:field="*{nickname}" class="form-control" placeholder="변경할 닉네임을 입력해주세요.">
                    <div class="invalid-feedback">닉네임을 입력하세요.</div>
                    <div class="form-text text-danger" th:if="${#fields.hasErrors('nickname')}" th:errors="*{nickname}">nickname Error</div>
                </div>
                <button type="submit" class="btn btn-outline-primary">변경하기</button>
            </form>
        </div>
  ...생략
```

## 결과
자신의 현재 닉네임 확인  
![nickname-1](https://user-images.githubusercontent.com/60730405/162615321-552b5dc1-1742-4d3f-b6b3-9e9c8ad6b9a3.JPG)  

닉네임 중복 검사 에러 메시지 확인  
![nickname-2](https://user-images.githubusercontent.com/60730405/162615322-17857d93-d10c-465f-ae19-4cd6faab58d4.JPG)  

성공적으로 닉네임 변경 확인  
![nickname-3](https://user-images.githubusercontent.com/60730405/162615325-d5e44e20-1d4a-4848-b7e5-297b45047e41.JPG)

***
지금까지 `닉네임(Nickname)`을 수정하는 기능을 구현했습니다.  
다음에는 `패스워드(Password)`를 수정하는 기능을 구현해 보겠습니다.  

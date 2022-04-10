![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이전에는 `닉네임(Nickname)`을 수정하는 기능을 구현했습니다.  
이번에는 `패스워드(Password)`를 수정하는 기능을 구현해 보도록 하겠습니다.  

어떻게 보면 `닉네임(Nickname)`을 수정하는 방식과 유사합니다.  

## PasswordForm 구현
변경할 `패스워드(Password)`를 전달받을 클래스를 구현합니다.  
여기서 패스워드를 한번 더 입력하기 위해 `newPasswordConfirm` 필드를 생성합니다.  

```java
package com.moon.aza.dto;

import lombok.Data;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Pattern;

@Getter
@NoArgsConstructor
@Data
public class PasswordForm {
    @Pattern(regexp = "(?=.*[0-9])(?=.*[a-zA-Z])(?=.*\\W)(?=\\S+$).{8,16}",
            message = "비밀번호는 8~16자 영문, 숫자, 특수문자를 사용하세요.")
    @NotBlank(message = "비밀번호는 필수 입력 값입니다.")
    private String newPassword;

    @Pattern(regexp = "(?=.*[0-9])(?=.*[a-zA-Z])(?=.*\\W)(?=\\S+$).{8,16}",
            message = "비밀번호는 8~16자 영문, 숫자, 특수문자를 사용하세요.")
    @NotBlank(message = "비밀번호는 필수 입력 값입니다.")
    private String newPasswordConfirm;
}

```

## PasswordFormValidator 구현
`PasswordForm` 객체의 유효성 검증을 위한 클래스인 `PasswordFormValidator`를 구현합니다.  
`validate()` 메서드를 재정의하여 입력한 새로운 패스워드와 확인을 위해 한번 더 입력한 패스워드와 동일한 패스워드인지 검사합니다.  
만약 같지 않다면 에러메시지를 전달합니다.  

```java
package com.moon.aza.validator;

import com.moon.aza.dto.PasswordForm;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;

@Component
public class PasswordFormValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return PasswordForm.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        PasswordForm passwordForm = (PasswordForm) target;
        if(!passwordForm.getNewPassword().equals(passwordForm.getNewPasswordConfirm())){
            errors.rejectValue("newPassword","wrong.value","입력한 새 패스워드가 일치하지 않습니다.");
        }
    }
}

```

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
    
    public void changePassword(String password) { this.password = password; }

}

```
`Member` Entity에 패스워드 변경을 위한 메서드인 `changePassword()` 메서드를 구현합니다.  

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

    public void changePassword(Member member, String password){
        member.changePassword(passwordEncoder.encode(password));
        memberRepository.save(member);

    }
  ...생략
}

```
`MemberService`에 전달받은 새로운 패스워드를 `PasswordEncoder`로 암호화하여 `Member` Entity의 `changPassword()` 메서드로 변경 후,  
`MemberRepository`에 실제 Database에 수정된 내용을 반영하도록 합니다.  

## SettingController 구현
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
    private final PasswordFormValidator passwordFormValidator;
    private final MemberService memberService;

    // 패스워드 변경
    @GetMapping("/settings/password")
    public void passwordForm(@CurrentMember Member member, Model model){
        model.addAttribute(member);
        model.addAttribute(new PasswordForm());
    }

    // 패스워드 변경
    @PostMapping("/settings/password")
    public String changePassword(@CurrentMember Member member, @Valid PasswordForm passwordForm,
                                 Errors errors, Model model, RedirectAttributes redirectAttributes){
        if(errors.hasErrors()){
            model.addAttribute(member);
            return "/settings/password";
        }

        memberService.changePassword(member, passwordForm.getNewPassword());
        redirectAttributes.addFlashAttribute("message","패스워드를 변경했습니다.");
        return "redirect:/settings/password";
    }

    @InitBinder("passwordForm")
    public void passwordFormInitBinder(WebDataBinder webDataBinder) { webDataBinder.addValidators(passwordFormValidator);}
  
  ...생략
  
}

```
입력한 새로운 패스워드를 검증하기 위한 `passwordFormInitBinder()` 메서드를 구현합니다.  
그리고 `/settings/password` 경로로 `GET`과 `POST` 방식으로 요청이 올 시, 동작하도록 구현합니다.  
`GET` 방식의 요청은 현재 로그인된 사용자 계정과 `PasswordForm` 객체를 전달합니다.  
`POST` 방식의 요청은 전달받은 `PasswordForm`에 에러가 발생 시, 에러 메시지를 전달합니다.   
그리고 성공적으로 수정이 완료되면 메시지와 함께 리다이렉트합니다.  

## 패스워드 수정(password.html) 페이지 구현
패스워드 수정 페이지를 구현합니다.  

```html
  ...생략
     <div class="newPassword" style="margin: 60px 40px 0 250px; padding: 20px 0 0 20px;">
      <div th:if="${message}" class="alert alert-info alert-dismissible fade show mt-3" role="alert">
        <span th:text="${message}">수정 완료</span>
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
      </div>    
        <div class="newPassword__top" style="font-family: 'cookie'; font-size: 1.8rem; margin-bottom: 15px; display: flex; justify-content: space-between;">
            <h1>비밀번호 변경</h1>
        </div>
        <form class="needs-validation col-12" th:object="${passwordForm}" th:action="@{/settings/password}" method="POST">
            <div class="form-group mt-3">
                <lable for="newPassword" class="form-label">새 패스워드</lable>
                <input type="password" th:field="*{newPassword}" class="form-control" required>
                <div id="newPasswordHelp" class="form-text text-muted"> 새 패스워드를 입력하세요.</div>
                <div class="invalid-feedback">패스워드를 입력하세요.</div>
                <div class="form-text text-danger" th:if="${#fields.hasErrors('newPassword')}" th:errors="*{newPassword}">new password error</div>
            </div> 
            <div class="form-group mt-3">
              <lable for="newPasswordConfirm" class="form-label">새 패스워드</lable>
              <input type="password" th:field="*{newPasswordConfirm}" class="form-control" required>
              <div id="newPasswordConfirmHelp" class="form-text text-muted"> 새 패스워드를 다시 한번 입력하세요.</div>
              <div class="invalid-feedback">패스워드를 다시 입력하세요.</div>
              <div class="form-text text-danger" th:if="${#fields.hasErrors('newPasswordConfirm')}" th:errors="*{newPasswordConfirm}">new password confirm error</div>
            </div>
          <div class="form-group mt-3">
            <button type="submit" class="btn btn-outline-primary">변경하기</button>
          </div>
        </form>
    </div>
  ...생략
```

## 결과
초기 패스워드 수정 페이지(password.html)  
![password-1](https://user-images.githubusercontent.com/60730405/162618391-46ba5d8e-856f-4f98-b41b-36ef58aee118.JPG)  

입력한 새로운 패스워드와 한번 더 입력한 패스워드가 일치하지 않을 시 나타나는 에러메시지  
![password-2](https://user-images.githubusercontent.com/60730405/162618392-d9b9e21e-91a2-4d0b-8614-34ea2ce2e657.JPG)  

입력한 새로운 패스워드가 8~16자 영문, 숫자, 특수문자를 사용하지 않을 경우 나타나는 에러메시지  
![password-3](https://user-images.githubusercontent.com/60730405/162618393-e7756a8e-7340-4c2a-a19f-5f002463e221.JPG)  

입력한 새로운 패스워드가 성공적으로 수정이 완료된 경우  
![password-4](https://user-images.githubusercontent.com/60730405/162618395-fc88c2db-1a45-4ff0-8ff9-5a13cd595993.JPG)  



![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에는 이메일 재전송 기능을 구현하기 전에 회원가입 후 자동 로그인 기능과  
현재 로그인된 사용자의 정보를 참조하는 애너테이션을 작성해보도록 하겠습니다.  

## 자동 로그인 기능
흔히 로그인 기능은 인증을 처리하기 위해 `UserDetailsService`를 구현하고 Database에서 회원 정보를 조회하도록 합니다.  
사용자가 입력한 ID와 Password와 생성된 `UserDeatils` 정보와 일치하는지 비교하고 처리합니다.  
실제 이러한 사용자에 대해서 검증하는 행위는 `AuthenticationManager(인증 매니저)`를 통해 이루어집니다.  
자동 로그인 기능을 구현할 때는 `AuthenticationManager`를 이용하지 않고 `SecurityContext`를 이용해서 구현해보도록 하겠습니다.  

### MemberController와 MemberService 수정
일단, `회원가입(/member/signup)` 요청과 `이메일 인증 확인(/member/email-check-token)` 요청에 대해 자동적으로 로그인할 수 있게 수정합니다.  
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
  ...생략
   
    /* 회원가입 로직 */
    @PostMapping("/member/signup") /* @Valid - 타입에 대한 검증 */
    public String memberRegister(@Valid @ModelAttribute SignUpForm signUpForm,
                                 Errors errors, Model model) throws MessagingException { /* Errors - 에러를 담을 수 있는 객체 */
        // 회원가입 유효성 검사
        if(errors.hasErrors()) {

            Map<String, String> validatorResult = memberService.validateHandling(errors);

            for(String key : validatorResult.keySet()){
                model.addAttribute(key, validatorResult.get(key));
            }
            return "/aza/signup";
        }

        Member newMember = memberService.signUp(signUpForm);
        // 회원가입 즉시 로그인
        memberService.login(newMember); // 추가

        return "/aza/home";
    }
  ...생략
}
```

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
public class MemberService implements UserDetailsService {
  ...생략

    // 이메일 인증 완료 되었다는 표시 후, 자동 로그인
    public void verify(Member member){
        member.verified();
        login(member); // 추가
    }
  ...생략

}

```

이제 `MemberService`에 로그인 기능을 구현해보도록 하겠습니다.  

## MemberService login() 추가
```java
package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
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
    ...생략

    /* 로그인 */
    public void login(Member member){
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(member.getEmail(),
                member.getPassword(), Collections.singleton(new SimpleGrantedAuthority("ROLE_USER")));
        SecurityContextHolder.getContext().setAuthentication(token);
    }
    ...생략
  
}

```
`UsernamePasswordAuthenticationToken`은 `AuthenticationManager`를 통한 인증을 실행하며,  
username, password를 쓰는 form기반 인증을 처리하는 필터입니다. 만약 성공하게 되면, `Authentication` 객체를 `SecurityContext`에 저장 후  
`AuthenticationSuccessHandler`를 실행합니다. 
`SecurityContextHolder`는 인증된 정보를 가지고 있는 holder이며, `setAuthentication()`에 사용자의 정보가 담긴 토큰을 전달합니다.  

이제, 회원가입하거나 이메일 인증 확인을 하게되면 자동적으로 로그인을 된 것을 확인할 수 있습니다.  
다음은 로그인 후 사용자 정보를 참조하는 애너테이션을 구현해보도록 하겠습니다.  

## Custom Annotation 구현(사용자 정보 참조)
Annotation을 구현하여 쉽게 사용자 정보를 가져올 수 있습니다.  
`support` 패키지(폴더)를 생성하여 안에 `CurrentMember` 애너테이션 형식으로 생성합니다.  

### CurrentMember 구현
```java
package com.moon.aza.support;

import org.springframework.security.core.annotation.AuthenticationPrincipal;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME) // 런타임시 유지
@Target(ElementType.PARAMETER) // 파라미터로 사용
@AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : member")
public @interface CurrentMember {
}
```
`@Retention(RetentionPolicy.RUNTIME)`은 런타임시 유지하겠다는 의미이고 `@Target(ElementType.PARAMETER)`은 parameter에 사용하겠다는 의미입니다.  
`@AuthenticationPrincipal(expression ="#this == 'anonymousUser' ? null : member")`은 인증 정보가 존재하지 않으면 `null`을    
아니면 `member`라는 값을 반환합니다.  

### UserMember 구현  
`Member`를 가져오기 위해 중간 `Adaptor` 역할을 하는 객체가 필요합니다.  

```java
package com.moon.aza.support;

import com.moon.aza.entity.Member;
import lombok.Getter;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;

import java.util.List;

public class UserMember extends User {
    @Getter
    private final Member member;

    public UserMember(Member member) {
        super(member.getEmail(), member.getPassword(), List.of(new SimpleGrantedAuthority("ROLE_USER")));
        this.member = member;
    }
}

```
아까 `CurrentMember`를 구현할 때 `member`를 반환하도록 하였기 때문에 필드 변수 이름을 `member`로 설정해야 합니다.  
그리고 `User`를 상속받고 이 `User` 객체를 생성하기 위해서는 `username`, `password`, `authority`가 필요하기 때문에  
`Member`에서 각각 얻어와서 설정해 줍니다.  

### MemberService 수정
아까 자동 로그인 기능을 구현할 때 작성했던 `login()` 메서드를 수정합니다.  
```java
package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
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
    ...생략

    /* 로그인 */
    public void login(Member member){
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(new UserMember(member),
                member.getPassword(), Collections.singleton(new SimpleGrantedAuthority("ROLE_USER")));
        SecurityContextHolder.getContext().setAuthentication(token);
    }
    ...생략

```
기존에는 `member.getEmail()`을 통해 이메일을 전달했던 parameter를 방금 작성했던 `UserMember`로 대체합니다.  

## MainController 수정
```java
package com.moon.aza.controller;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.support.CurrentMember;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Log4j2
@RequiredArgsConstructor
@Controller
public class MainController {

    /* 홈(메인) 페이지 */
    @GetMapping("/")
    public String home(@CurrentMember Member member, Model model) {
        log.info("/");
        if(member != null)
            model.addAttribute(member);

        return "/aza/home";
    }

  ...생략
}

```
`@CurrentMember`를 사용하여 현재 인증된 사용자 정보를 객체에 할당하게 됩니다.  
`MainController`에서 `@CurrentMember`에 의해 `@AuthenticationPrincipal`이 적용되어 인증 여부에 따라 `member`를 반환해서 넘겨줍니다.  


***
지금까지 자동 로그인 기능과 사용자 정보를 참조하는 애너테이션을 생성하였습니다.  
다음에는 이를 이용하여 이메일 인증 메일을 재전송하는 기능을 구현해보도록 하겠습니다.  


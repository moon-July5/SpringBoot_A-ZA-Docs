![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에는 회원가입을 완료하기 전에 회원정보가 유효한지를 검사하는 기능을 추가할 것입니다.  
유효하지 않으면 회원가입 처리가 완료되지 않으며, 모든 회원 정보가 유효해야지 회원가입이 됩니다.  

## Dependencies
`build.gradle`파일의 `dependencies`에서 `spring-boot-starter-validation` 패키지를 추가합니다.
```
...생략
  implementation 'org.springframework.boot:spring-boot-starter-validation'
...생략
```

## MemberController 수정
```java
package com.moon.aza.controller;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.MemberService;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.Errors;

import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.Map;

@Log4j2
@RequiredArgsConstructor
@Controller
public class MemberController {
    private final MemberService memberService;
    
    
    /* 회원가입 로직 */
    @PostMapping("/member/signup") /* @Valid - 타입에 대한 검증 */
    public String memberRegister(@Valid @ModelAttribute SignUpForm signUpForm, Errors errors, Model model)  { /* Errors - 에러를 담을 수 있는 객체 */
        // 회원가입 유효성 검사
        if(errors.hasErrors()) {

            Map<String, String> validatorResult = memberService.validateHandling(errors);

            for(String key : validatorResult.keySet()){
                model.addAttribute(key, validatorResult.get(key));
            }
            return "/aza/signup";
        }

        Member newMember = memberService.signUp(signUpForm);

        return "/aza/home";
    }
  }
```
`@Valid`을 `SignUpForm` 객체 앞에 추가하여 타입에 대한 검증을 실시합니다.  
그리고 에러를 담을 수 있는 객체인 `Errors`를 추가합니다.  
에러가 존재할 경우, 유효성을 통과하지 못한 필드와 메시지를 핸들링하고 `회원가입(signup.html)` 페이지로 다시 return 합니다.  
Errors 객체로 에러가 전달되기 때문에, Thymeleaf로 랜더링된 HTML에 해당 에러를 전달하여 업데이트할 수 있습니다.  

유효성 검사에 실패한 필드가 존재한다면, `Service` 계층에게 Errors 객체를 전달합니다.  

# MemberService 수정
```java
package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;

import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.validation.Errors;
import org.springframework.validation.FieldError;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

@RequiredArgsConstructor
@Transactional
@Service
public class MemberService {
    private final MemberRepository memberRepository;
    private final PasswordEncoder passwordEncoder;
    
    ...생략
    
     /* 회원가입 시, 유효성 체크*/
    public Map<String, String> validateHandling(Errors errors){
        Map<String, String> validatorResult = new HashMap<>();
        for(FieldError error : errors.getFieldErrors()){
            String validKeyName = String.format("valid_%s", error.getField());
            validatorResult.put(validKeyName, error.getDefaultMessage());
        }
        return validatorResult;
    }
    
    ...생략
}
```
유효성 검사에 실패한 필드들은 `Map` 자료구조를 통해 Key 값과 Error 메시지를 응답합니다.  
>key : valid_{field}  
Message : SignUpForm에서 작성한 message 값  

## SignUpForm 수정
```java
package com.moon.aza.dto;

import lombok.Data;

import javax.validation.constraints.Email;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Pattern;

@Data
public class SignUpForm {
    @Email(message = "올바른 이메일 형식을 입력해주세요")
    @NotBlank(message = "이메일은 필수 입력 값입니다.")
    private String email;

    // 8~16자 영문, 숫자, 특수문자를 사용
    @Pattern(regexp = "(?=.*[0-9])(?=.*[a-zA-Z])(?=.*\\W)(?=\\S+$).{8,16}",
            message = "비밀번호는 8~16자 영문, 숫자, 특수문자를 사용하세요.")
    @NotBlank(message = "비밀번호는 필수 입력 값입니다.")
    private String password;

    @NotBlank(message = "닉네임은 필수 입력 값입니다.")
    private String nickname;

}
```
`@NotBlank`는 비어있는 값인지 여부를 검사합니다.  
`@Email`은 이메일 형식인지 검사합니다.  
`@Pattern`은 문자열의 패턴을 검사합니다. 정규표현식을 사용했으며, 8~16사이의 길이로 영문, 숫자, 특수문자를 사용해야 한다는 의미입니다.  

마지막으로 회원가입(signup.html) 페이지에서 유효성 검사를 통과하지 못할 경우, 에러를 표시하도록 수정합니다.  

## 회원가입(signup.html) 페이지 수정
```html

... 생략

      <div class="signup">
            <div class="signup__top">
                <h1>회원가입</h1>
            </div>
            <div class="signup__container">
                <form class="needs-validation col-sm-6" th:action="@{/member/signup}" method="post" th:object="${signUpForm}" novalidate>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="email" style="font-weight:bold; margin-bottom:10px;">이메일</label>
                        <input type="email" class="form-control" name="email" th:field="*{email}" placeholder="ex) aaa@exple.com" required>
                        <small class="form-text text-danger" th:text="${valid_email}"></small>
                    </div>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="password" style="font-weight:bold; margin-bottom:10px;">비밀번호</label>
                        <input type="password" class="form-control" name="password" th:field="*{password}" placeholder="비밀번호는 8~16자 영문, 숫자, 특수문자를 사용하세요." required>
                        <small class="form-text text-danger" th:text="${valid_password}"></small>
                    </div>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="nickname" style="font-weight:bold; margin-bottom:10px;">닉네임</label>
                        <input type="text" class="form-control" name="nickname" th:field="*{nickname}" placeholder="닉네임을 입력해주세요." required>
                        <small class="form-text text-danger" th:text="${valid_nickname}"></small>
                    </div>
              
                    <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}"> 홈 화면으로</button>
                    <button type="submit" class="btn btn-primary signupButton"><img class="button__icon" th:src="@{/imgs/signup-icon-white.png}">회원가입</button>
                </form>
            </div>
        </div>

...생략
```
아까 `Service` 계층에서 `vaild_{fieldname}` 형식으로 설정했기 때문에 위와 같이 메시지가 나타나도록 `Thymeleaf` 문법으로 수정합니다.  

그러면 제대로 실행이 되는지 확인해 보겠습니다.  
![signup-2](https://user-images.githubusercontent.com/60730405/157684055-c3353288-8f4c-4769-8105-0effe2cfe5a9.JPG)
![signup-3](https://user-images.githubusercontent.com/60730405/157684070-9469a058-587a-4244-a68c-b654163dd59d.JPG)

회원정보가 유효하지 않기 때문에 각 회원정보 입력창 밑에 에러 메시지가 출력이 되는 것을 확인할 수 있습니다.  

다음에 할 것은 이메일과 닉네임의 중복 여부를 확인하는 것을 해보겠습니다.
***

지금까지는 회원정보 양식에 맞게 작성하지 않으면 에러를 발생시키도록 하였습니다.  
이번에 할 것은 이메일과 닉네임의 중복 여부를 확인하도록 검사해봐야 합니다.  

닉네임이나 이메일같은 경우는 Database에 이미 등록한 사용자가 있는지 찾아봐야 합니다.  
그래서 따로 검증하는 SignUpForm class를 만들겠습니다.  

## SignUpFormValidator 구현
```java
package com.moon.aza.validator;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;

@RequiredArgsConstructor
@Component
public class SignUpFormValidator implements Validator {
    private final MemberRepository memberRepository;

    /* SignUpForm 일때만 검증 수행 */
    @Override
    public boolean supports(Class<?> clazz) {
        return clazz.isAssignableFrom(SignUpForm.class);
    }
    /* 검증 수행  */
    @Override
    public void validate(Object target, Errors errors) {
        SignUpForm signUpForm = (SignUpForm) target;
        // 중복 여부 체크
        if(memberRepository.existsByEmail(signUpForm.getEmail())){
            errors.rejectValue("email", "invalid.email", new Object[]{signUpForm.getEmail()},
                    "이미 사용중인 이메일입니다.");
        }
        if(memberRepository.existsByNickname(signUpForm.getNickname())){
            errors.rejectValue("nickname","invalid.nickname",new Object[]{signUpForm.getNickname()},
                    "이미 사용중인 닉네임입니다.");
        }
    }
}
```
Spring에서 제공하는 객체 검증용 인터페이스인 `Validator`를 상속받습니다.  
`@Component`를 클래스 선언부 위에 설정하여 Spring에서 관리할 수 있게 Spring Bean 객체로 등록합니다.  
여기서 `supports()`와 `validate()`를 `재정의(override)`합니다.  
`supports()` 메서드는 인스턴스가 검증 대상 타입인지 확인하는 역할을 하므로 SignUpForm 일 때만 검증을 수행하도록 합니다.  
그리고 `validate()` 메서드는 실질적인 검증 작업을 수행합니다. 여기서 이메일과 닉네임의 중복 여부흘 확인할 것이기 때문에  
`target`을 검증할 타입인 `SignUpForm`으로 캐스팅해주고 `Repository`를 이용하여 중복 여부를 확인합니다.  
만약, 중복일 경우 errors 객체에 어떤 field에 에러가 있는지 검사합니다.  

## MemberRepository 수정
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Member;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
    /* 이메일 중복 여부 확인 */
    boolean existsByEmail(String email);

    /* 닉네임 중복 여부 확인 */
    boolean existsByNickname(String nickname);
}
```
JPA가 자동으로 만들어주는 `쿼리 메서드`를 통해 이메일과 닉네임 중복 여부를 확인하는 메서드를 구현합니다.  

## MemberController 수정
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

import javax.validation.Valid;
import java.util.Map;

@Log4j2
@RequiredArgsConstructor
@Controller
public class MemberController {
    private final SignUpFormValidator signUpFormValidator;
    private final MemberService memberService;

    ... 생략

    /* 객체 검증*/
    @InitBinder("signUpForm") // attribute로 바인딩할 객체 지정
    public void initBinder(WebDataBinder webDataBinder){
        webDataBinder.addValidators(signUpFormValidator);
    }
    
    ... 생략

}
```
`@InitBinder`로 attribute로 바인딩할 객체를 지정합니다.  
`WebDataBinder`를 이용하여 전에 구현한 `SignUpFormValidator` 클래스를 추가하면 해당 클래스가 들어왔을 때  
검증하는 로직을 추가할 필요가 없습니다.  
즉, `@InitBinder`에 객체 `signUpform`를 지정했으며, 이렇게 되면 `WebDataBinder`가 `signUpform`라는 객체에 binding할 때, 검증을 수행합니다.  

## 결과 
이미 회원가입한 사용자와 같은 이메일과 닉네임으로 가입을 시도하게되면 메시지가 출력이 됩니다.  
![signup-4](https://user-images.githubusercontent.com/60730405/157863576-53f67418-9e1e-475a-ad7c-e77ae63478e4.JPG)


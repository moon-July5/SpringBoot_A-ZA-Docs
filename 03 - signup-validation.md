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




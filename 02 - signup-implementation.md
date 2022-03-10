![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***

바로 전에는 회원가입을 위한 **Member Entity**를 생성하고 **회원가입(signup.html) 페이지**를 구현했습니다.  
그리고 **Security**를 설정하여 회원가입 각 경로에 접근을 허용하도록 했습니다.

이번에는 회원가입을 처리하는 방법에 대해 설명하겠습니다.

## DTO 구현(SignUpForm)
`DTO`라는 것은 **Data Transfer Object**의 약자로, `Entity`객체와 달리 각 계층끼리 주고받는 우편물이나 상자의 개념입니다.  
**순수하게 데이터를 담고 있다는 점**에서 `Entity` 객체와 유사하지만,  목적 자체가 `데이터의 전달`이므로 읽고, 쓰는 것이  
허용되는 점이 가능하고 일회성으로 사용되는 성격이 강합니다.  
```java
package com.moon.aza.dto;

import lombok.Data;


@Data
public class SignUpForm {

    private String email;

    private String password;

    private String nickname;

}
```
회원가입할 때 필요한 정보들, `email`, `password`, `nickname`을 필드로 줬습니다.  
`@Data`는 `@Getter`, `@Setter`, `@RequiredArgsConstructor`, `@ToString`, `@EqualsAndHashCode`을 한꺼번에 설정해주는 기능입니다.  
이제 이 클래스를 가지고 `Controller`를 수정해보겠습니다.  

## MainController 수정
```java
package com.moon.aza.controller;

import com.moon.aza.dto.SignUpForm;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Log4j2
@RequiredArgsConstructor
@Controller
public class MainController {

    /* 회원가입 페이지 */
    @GetMapping("/signup")
    public String signup(Model model) {
        log.info("/signup");
        model.addAttribute(new SignUpForm());
        return "/aza/signup";
    }
}
```
위와 같이 수정해주며, 웹 브라우저에서 `/signup`을 요청 시, ccontroller가 호출되면서 `SignUpForm`이라는 객체를 생성해서 `Model`을 통해 전달해줍니다.
그리고 `/aza/signup` 페이지로 redirect 해줍니다.  
이제 `signup.html`도 수정해줍니다.  

## 회원가입(signup.html) 페이지 수정
```html
... 
        <div class="signup">
            <div class="signup__top">
                <h1>회원가입</h1>
            </div>
            <div class="signup__container">
                <form class="needs-validation col-sm-6" th:action="@{/member/signup}" method="post" th:object="${signUpForm}" novalidate>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="email" style="font-weight:bold; margin-bottom:10px;">이메일</label>
                        <input type="email" class="form-control" name="email" th:field="*{email}" placeholder="ex) aaa@exple.com" required>
                    </div>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="password" style="font-weight:bold; margin-bottom:10px;">비밀번호</label>
                        <input type="password" class="form-control" name="password" th:field="*{password}" placeholder="비밀번호는 8~16자 영문, 숫자, 특수문자를 사용하세요." required>
                    </div>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="nickname" style="font-weight:bold; margin-bottom:10px;">닉네임</label>
                        <input type="text" class="form-control" name="nickname" th:field="*{nickname}" placeholder="닉네임을 입력해주세요." required>
                    </div>
              
                    <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}"> 홈 화면으로</button>
                    <button type="submit" class="btn btn-primary signupButton"><img class="button__icon" th:src="@{/imgs/signup-icon-white.png}">회원가입</button>
                </form>
            </div>
        </div>
  ...생략
 ```
여기서 수정된 것은 form에 `th:object`와 `th:field`가 생긴 것을 확인할 수 있습니다.  
`thymeleaf` 문법으로 `th:object`는 form Submit할 때, form 안에 있는 데이터가 `th:object`에 설정해준 객체로 받아지는 것입니다.  
`th:field`는 각각 field들을 매핑해주는 역할을 합니다. 설정해준 값으로, `th:object`에 설정해준 객체의 내부와 매칭해줍니다.  

여기까지 수정이 완료된 후, 웹 브라우저에서 `/signup` 경로로 회원가입 페이지를 확인해도 별 다른 점을 없을 것입니다.  
이제 회원가입을 처리하는 로직에 대해 설명하겠습니다.  
***

## 회원가입 처리(MemberController) 
회원에 대한 처리를 해주는 Controller인 `MemberController`를 생성해 줍니다.
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
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;


@Log4j2
@RequiredArgsConstructor
@Controller
public class MemberController {
    private final MemberService memberService;

    /* 회원가입 로직 */
    @PostMapping("/member/signup") 
    public String memberRegister(@ModelAttribute SignUpForm signUpForm, Model model)  { 


        Member newMember = memberService.signUp(signUpForm);
 
        return "/aza/home";
    }
```
`@PostMapping`은 `post`방식으로 `RequestMapping`을 하며 `/member/signup` 경로로 요청 시, 회원가입을 처리하게 됩니다.  
`SignUpForm` 객체에는 회원의 정보가 저장되어 있으며, 이를 나중에 구현할 `Service` 계층인 `MemberService`에서 `signUp()` 메서드로 회원가입을 처리하게 됩니다.  
다음은 `Service` 계층인 `MemberService`를 구현해 보겠습니다.  

## 회원가입 처리(MemberService) 구현
`Service` 계층은 `Controller` 계층과 그 다음에 구현할 `Repository` 계층을 연결하는 역할을 합니다.  
```java
package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;


@RequiredArgsConstructor
@Transactional
@Service
public class MemberService {
    private final MemberRepository memberRepository;
    private final PasswordEncoder passwordEncoder;
    
    /* 회원정보 보내기(실질적으로 컨트롤러에서 호출)  */
    public Member signUp(SignUpForm signUpForm) {
        Member newMember = saveMember(signUpForm);
        // 이메일 인증용 토큰 생성
        newMember.generateToken();
        return newMember;
    }

    /* 회원정보 저장 로직*/
    public Member saveMember(SignUpForm signUpForm){
        Member member = Member.builder()
                .email(signUpForm.getEmail())
                .nickname(signUpForm.getNickname())
                .password(passwordEncoder.encode(signUpForm.getPassword()))
                .build();

        return memberRepository.save(member);
    }
 }
 ```
 `@Service`로 `Service` 계층임을 나타냅니다.  
 `saveMember()` 메서드는 `SignUpForm` 객체에 저장되어 있는 회원정보들을 얻어와서 `DTO`형태를 `빌더 패턴`을 통해 `Member` 객체를 생성합니다.  
 이는 `Repository` 계층에는 `Entity` 형태로 전달하기 위해서입니다.  
 여기서 `PasswordEncoder`라는 것이 사용되었는데, 이것은 비밀번호를 평문 그대로 사용할 수 없기 때문에 `Spring Security`에서 제공해주는  
 `PasswordEncoder`를 사용하여 비밀번호를 암호화합니다. `PasswordEncoder`는 기본값으로 `BCrypt` 알고리즘을 사용합니다.  
 그리고 이렇게 저장된 `Member` 객체를 `MemberRepository`로 전달해줍니다.  
 
 `signUp()` 메서드는 `Controller` 계층에서 실제로 호출하는 함수입니다. 이 함수에는 회원정보를 저정하는 `saveMember()` 메서드를 호출하여    
 새로운 회원을 나타내는 `Member` 객체를 생성하고 나중에 구현할 이메일 인증을 위한 토큰도 생성하여 `Member` 객체를 return 해줍니다.  
 
 다음은 `Repository` 계층을 구현해 보겠습니다.  
 
 ## 회원가입 처리(MemberRepository) 구현
 `repository` 계층은 `service` 계층에서 데이터를 받아 실제 Database에 접근하는 메서드들을 사용하기 위한 인터페이스입니다.  
 ```java
 package com.moon.aza.repository;

import com.moon.aza.entity.Member;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
  
}
```
`@Repository`로 `Repository` 계층임을 나타냅니다.  
상속받은 `JpaRepository`는 인터페이스이며, 이것은 상속받은 것만으로 모든 작업이 끝납니다. 
상속받을 때 형식은 `JpaRepository<객체, @ID 타입>` 입니다.  

다음은 아까 `Service` 계층에서 회원정보를 저장할 때, 이메일 인증용 토큰도 저장했는데, 이에 대해서 `Member` Entity를 조금 수정하겠습니다.  

## 회원가입 처리(Member Entity 수정) 구현
```java
package com.moon.aza.entity;

import lombok.*;

import javax.persistence.*;
import java.util.UUID;

@ToString
@Builder
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member extends BaseEntity {
    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @Column(unique = true)
    private String email;

    @Column(unique = true)
    private String nickname;

    private String password;

    private boolean isValid;

    private String emailToken;

    // 이메일 인증용 토큰 생성 함수 추가
    public void generateToken(){
        this.emailToken = UUID.randomUUID().toString();
    }
}
```
이메일 인증용 토큰을 생성하기 위해 **범용 고유 식별자**인 `UUID(Universally Unique Identifier)`로 생성합니다.  
실제 사용시에 중복될 가능성이 없기 때문에 사용합니다.  

이것으로 회원가입 처리를 구현했습니다.  




 

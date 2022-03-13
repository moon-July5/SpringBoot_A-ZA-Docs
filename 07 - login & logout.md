![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 것은 로그인 기능과 로그아웃 기능을 구현해 보겠습니다.  

## Login 기능 구현  
`SecurityConfig` 클래스에 로그인과 로그아웃에 대한 설정을 추가합니다.  
```java
package com.moon.aza.config;

import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

import javax.sql.DataSource;

@Log4j2
@RequiredArgsConstructor
@EnableWebSecurity // spring security 설정 클래스 선언
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
  ...생략
  
    // http 요청에 대한 웹 기반 보안
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()    // 접근 제한
                .antMatchers("/","/board","/login","/signup",
                        "/member/email-check-token"
                        ).permitAll(); // 특정 경로 지정, 접근을 설정
        http.formLogin() // login
                .loginPage("/login")
                .permitAll();
        http.logout() // logout
                .logoutRequestMatcher(new AntPathRequestMatcher("/logout"))
                .logoutSuccessUrl("/");
    
    }
    ...생략

}
```
`formLogin()`을 설정하여 form 기반 인증을 지원하도록 합니다. `.loginPage()`를 지정하지 않으면 Spring이 기본 로그인 페이지를 설정해줍니다.  
`.permitAll()`를 지정하여 로그인 페이지에 누구든지 접근할 수 있도록 합니다.    
`logout()`을 이용하여 로그아웃을 설정합니다.  
로그아웃 설정 중에서 `.logoutRequestMatcher(new AntPathRequestMatcher("/logout"))`을 지정하여 `/logout` 경로에 요청이 가면  
로그아웃이 되도록 설정했으며, `.logoutSuccessUrl("/")`으로 로그아웃 성공 시 홈 화면으로 이동하도록 했습니다.  

## MainController 수정
로그인 시 로그인 페이지로 이동하도록 `MainController`에 추가합니다.  
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
  ...생략

    /* 로그인 페이지 */
    @GetMapping("/login")
    public String login(){
        log.info("/login");
        return "/aza/login";
    }
  ...생략
}

```
## 로그인(login.html) 페이지 구현  
```html
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
      
                <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}">홈 화면으로</button>
                <button type="submit" class="btn btn-primary loginButton"><img class="button__icon" th:src="@{/imgs/login-icon-white.png}">로그인</button>
                <button type="button" th:onclick="|location.href='@{/signup}'|" class="btn btn-info" style="color: white;"><img class="button__icon" th:src="@{/imgs/signup-icon-white.png}">회원가입</button>
            </form>
```
로그인 화면에서 `form` 태그 작성 후, 요청하게 되면 `/login`을 호출합니다.  
`Spring Security`가 로그인 핸들러에서 사용하는 변수 이름인 `username`, `password`로 동일하게 작성해야 하며, 만약 변수 이름을 바꾸고 싶으면  
`SecurityConfig` 설정 파일에서 `formLogin()`에 `.usernameParameter()'와 `.passwordParameter()`에 값을 설정해줍니다.  

여기서 `form` 태그의 입력 값들이 유효하지 않을 때 Submit 되지 않도록 아래와 같이 맨 밑단에 추가합니다.  
`form`의 클래스 이름 `needs-validation`이 들어간 경우 유효성 검사를 합니다.  
```html
<script type="application/javascript" th:fragment="form-validation">
        (function () {
            'use strict';
    
            window.addEventListener('load', function () {
                // Fetch all the forms we want to apply custom Bootstrap validation styles to
                const forms = document.getElementsByClassName('needs-validation');
    
                // Loop over them and prevent submission
                Array.prototype.filter.call(forms, function (form) {
                    form.addEventListener('submit', function (event) {
                        if (form.checkValidity() === false) {
                            event.preventDefault();
                            event.stopPropagation();
                        }
                        form.classList.add('was-validated')
                    }, false)
                })
            }, false)
        }())
    </script>
```

로그인을 하는 기능은 `UserDetailsService`를 구현해야 합니다.  
## MemberService 수정(UserDetatilsService 구현)
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
  ...생략
    /* 사용자 정보 조회 */
    @Override
    @Transactional(readOnly = true) // 조회 용도로만 사용
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberRepository.findByEmail(username);
        if(member == null)
            throw new UsernameNotFoundException(username);

        return new UserMember(member);
    }
  ...생략
}

```
`UserDetatilsService`를 상속받아 이 인터페이스가 제공하는 메서드인 `loadUserByUsername()`를 재정의합니다.  
Database에서 회원 정보가 존재하는지 확인하여 사용자 정보를 반환해 줍니다. 여기서 `email`로 로그인을 진행할 것이기 때문에  
`findByEmail()`를 통해 회원 정보를 조회한 후, 존재하지 않으면 `Exception`을 발생시킵니다.  
존재하면, 전에 사용자 정보를 참조하는 애너테이션을 만들기 위해 구현했던 `UserMember`에 `UserDetails` 인터페이스를 구현했기 때문에  
객체를 반환해줍니다.  

## 결과  
로그인을 성공적으로 했을 경우 홈 화면으로 이동
![login-1](https://user-images.githubusercontent.com/60730405/158057160-fe3baf17-4dcc-415b-bb48-d6c91f1c2c00.JPG)  
![login-2](https://user-images.githubusercontent.com/60730405/158057173-4fff6e66-4413-4606-a153-a17c219f5025.JPG)  

로그인에 실패했을 경우 에러 메시지 출력  
![login-3](https://user-images.githubusercontent.com/60730405/158057203-397fe5c6-fb28-49ee-893a-a62e18a40c8b.JPG)  

로그아웃을 성공적으로 했을 경우 홈 화면으로 이동  
![logout-1](https://user-images.githubusercontent.com/60730405/158057235-790dde5d-3df4-4ba6-b805-15ba490d0f87.png)  
![logout-2](https://user-images.githubusercontent.com/60730405/158057241-25dd9679-85c5-40f9-a425-d29d683d9744.JPG)

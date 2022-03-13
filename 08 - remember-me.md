![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에는 로그인을 유지하는 기능을 구현해 보겠습니다.  

로그인한 상태에서 개발자 도구 -> Application -> Cookies를 확인해보면 `JSESSIONID`가 존재하는 것을 확인할 수 있습니다.  
이는 로그인 후 서버에서 `JSESSIONID`를 발급해주고 클라이언트(웹 브라우저)에서 정보를 `쿠키`에 저장합니다.  
클라이언트(웹 브라우저)에서 `JSESSIONID`를 서버에 같이 요청하여 서버에서는 로그인 되었다고 생각하고 요청에 응답해줍니다.  
여기서 쿠키에 저장된 `JSESSIONID`를 삭제시키고 페이지를 새로고침하게 되면 로그아웃이된 상태인 것을 확인할 수 있습니다.  

`Spring Boot`에서는 로그인 유지 시간을 기본적으로 30분동안 세션을 유지하도록 되어있습니다.  
따라서, 30분후 다시 로그인을 해야하는 상태가 된다는 것입니다.  
세션 시간을 긴 시간 동안 설정할 수 있지만 그렇다고 그렇게 설정하면 서버 메모리에 영향을 줄 수 있습니다.  

로그인 유지(Remember-Me)기능을 구현하기 위해 단순히 쿠키만을 이용하는 방법이 있지만  
랜덤한 토큰 값과 Unique한 식별자(Series)를 같이 생성하여 사용하는 방식을 사용하여 `Remember-Me` 기능을 구현할 것입니다.  
일단, Unique한 식별자 값은 첫 발급된 이후 고정 값입니다.  
만약, 침입자가 도중에 토큰을 도용해서 인증을 성공하게 되면, 사용자는 잘못된 토큰을 요청하게 될 것입니다.  
기존에 사용하던 모든 토큰을 삭제하여 침입자쪽에서도 접근하지 못하도록 합니다.  
즉, Database에 저장된 토큰값을 이용해 쿠키를 이중 인증하는 방법입니다.  


## SecurityConfig 수정  
```java
package com.moon.aza.config;

import com.moon.aza.service.MemberService;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.rememberme.JdbcTokenRepositoryImpl;
import org.springframework.security.web.authentication.rememberme.PersistentTokenRepository;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

import javax.sql.DataSource;

@Log4j2
@RequiredArgsConstructor
@EnableWebSecurity // spring security 설정 클래스 선언
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    private final MemberService memberService; // 추가
    private final DataSource dataSource; // 추가

   ...생략

    // http 요청에 대한 웹 기반 보안
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()    // 접근 제한
                .antMatchers("/","/board","/login","/signup",
                        "/member/email-check-token"
                        ).permitAll(); // 특정 경로 지정, 접근을 설정
        http.formLogin()
                .loginPage("/login")
                .permitAll();
        http.logout()
                .logoutRequestMatcher(new AntPathRequestMatcher("/logout"))
                .logoutSuccessUrl("/");
        http.rememberMe() // 추가
                .userDetailsService(memberService)
                .tokenRepository(tokenRepository());
    }
    // 추가
    @Bean
    public PersistentTokenRepository tokenRepository(){
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        jdbcTokenRepository.setDataSource(dataSource);
        return jdbcTokenRepository;
    }
  ...생략
}

```
토큰을 관리하기 위한 `tokenRepository()` 메서드를 구현합니다.  
`JdbcTokenRepositoryImpl`은 Database의 `persistent_logins` 테이블을 대상으로 한 SQL들이 들어있는 구현체입니다.  
이것을 통해 Database에 토큰을 저장하고 갱신하고 삭제합니다. 여기에 토큰 저장소를 위한 `DataSource`를 주입합니다.  

`http.rememberMe()`에 `userDetailsService`를 설정하고 토큰을 관리할 `tokenRepository()`를 설정합니다.  

이제 `PersistentLogins` 클래스를 구현합니다.  
## PersistentLogins 구현
```java
package com.moon.aza.entity;

import lombok.Getter;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;
import java.time.LocalDateTime;

@Getter
@Table(name = "persistent_logins")
@Entity
public class PersistentLogins {
    @Id
    @Column(length = 64)
    private String series;

    @Column(length = 64)
    private String username;

    @Column(length = 64)
    private String token;

    @Column(length = 64, name="last_used")
    private LocalDateTime lastUsed;

}
```
아까 `SecurityConfig`에 구현한 `JdbcTokenRepositoryImpl` 클래스에 내부적으로 사용하는 테이블이 있기 때문에  
그에 따라 필드를 생성합니다.  

이제 `로그인(login.html)` 페이지에 `로그인 유지 체크박스`를 생성합니다.  
## 로그인(login.html) 수정  
```html
  ...생략
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
                <!--로그인 유지 체크박스 추가-->
                <div class="form-group">
                    <input type="checkbox" class="form-check-input" id="rememberMe"  name="remember-me" checked> 
                    <label class="form-check-label" for="rememberMe">로그인 유지</label>
                </div><br>
                <!--로그인 유지 체크박스 추가-->
                <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}">홈 화면으로</button>
                <button type="submit" class="btn btn-primary loginButton"><img class="button__icon" th:src="@{/imgs/login-icon-white.png}">로그인</button>
                <button type="button" th:onclick="|location.href='@{/signup}'|" class="btn btn-info" style="color: white;"><img class="button__icon" th:src="@{/imgs/signup-icon-white.png}">회원가입</button>
            </form>
        </div>
  ...생략
```
여기까지 구현한 후 Application을 실행하게 되면 error가 발생할 것입니다.  
이는 `MemberService`와 `SecurityConfig`가 서로 순환 참조를 하고 있기 때문입니다.  
`MemberService`에서 `PasswordEncoder`를 사용하고 있는데, 이 `PasswordEncoder`를 사용하기 위해서는 `SecurityConfig` Bean이 먼저 생성되어야 합니다.  
하지만 `SecurityConfig`를 생성하려면 `MemberService`를 주입받아야 하므로 서로 순환 참조가 발생하는 것입니다.  
그래서 이 `PasswordEncoder`를 따로 다른 클래스에 옮겨 설정해줍니다.  

## AppConfig 구현
```java
package com.moon.aza.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;

// MemberService와 SecurityConfig가 PasswordEncoder로 인해 서로 순환 참조되고 있기 때문에
// 그 원인이 되는 PasswordEncoder를 이렇게 따로 옮겨서 설정
@Configuration
public class AppConfig {

    /* 비밀번호 암호화 객체 */
    @Bean
    public PasswordEncoder passwordEncoder(){
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}

```

## 결과  
로그인을 할 때, `로그인 유지`를 체크하고 로그인을 합니다.  
![remember-me-1](https://user-images.githubusercontent.com/60730405/158059575-00554c37-1169-4dfd-875e-807b34ab81fc.png)  

쿠키를 확인해보면 `remember-me`가 생성된 것을 확인할 수 있습니다.  
![remember-me-2](https://user-images.githubusercontent.com/60730405/158059578-a7fde6ad-c619-4095-b20e-d30c067595c2.png)  

여기서 로그인이 잘 유지되는지 `JSESSIONID`를 삭제하고 다시 새로고침을 하게되면, 추가 인증 필요없이 `JSESSIONID`가 다시 생성된 것을  
확인할 수 있습니다.  

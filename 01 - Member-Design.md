![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
게시판 웹을 만들기 전에 게시판을 등록하거나 삭제, 수정하기 위해서는 먼저 회원을 등록하고 로그인을 해줘야 합니다.  
그렇기 때문에 회원가입, 로그인 기능을 개발하기 위해 **회원(Member) 도메인 설계**와 **회원가입 폼**을 구현해 보겠습니다.  

# Member Entity 구현
`Member` Entity를 구현하기에 앞서, 먼저 회원가입을 할 때 작성할 항목들을 정리해 보겠습니다.

* email : 로그인을 하기위해 사용, Unique 해야한다.
* nickname : 다른 사용자들에게 노출되기 위해 사용, Unique 해야한다.
* password : 비밀번호, 8 ~ 16 영문, 숫자, 특수문자 사용
* isValid : 이메일 인증 여부
* emailToken : 이메일 인증을 위한 토큰



이를 토대로 `Member` Entity를 구현해보겠습니다.  
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
}
```
클래스위에 `Annotation`들이 있는데, 먼저 아래서부터 차례대로 설명하겠습니다. 


`@Entity`은 클래스 위에 선언하게 되면, 이 클래스는 Entity임을 알려줍니다. 이렇게 되면 JAP가 관리하게 되며,   
정의된 필드들을 바탕으로  Database에서 테이블을 만들어 줍니다.  

`@AllArgsConstructor(access = AccessLevel.PROTECTED)`와 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`은 각각 선언된 모든 필드를 parameter로
갖는 생성자를 자동으로 만들어주고 parameter가 아예 없는 기본생성자를 자동으로 만들어 줍니다.  
여기서 `@Entity`는 기본 생성자가 반드시 존재해야 하는데, 외부에서 new 할 수 없도록 `PROTECTED`레벨로 설정하였습니다.  

`@Getter`는 각 필드 값을 조회할 수 있는 Getter를 자동으로 생성해줍니다.  

`@Builder`는 해당 클래스에 해당하는 Entity 객체를 만들 때 빌더 패턴을 이용해서 만들 수 있도록 지정해줍니다.  

`@ToString`는 해당 클래스에 선언된 필드들을 모두 출력할 수 있는 toString() 메서드를 자동으로 생성해줍니다.  


이제 필드 안에 있는 `Annotation`들을 설명하겠습니다.  

`@Id`와 `@GeneratedValue`는 해당 Entity의 주요 키(Primary Key)가 될 값을 지정해주는 것이 `@Id`이며, `@GeneratedValue`는 PK가 자동으로 1씩  
증가하는 형태로 생성될지 결정해줍니다.  

`@Column`의 속성 중에서 **Unique** 값을 추가해 필드에 고유의 값만 추가할 수 있도록 해줍니다.  

`Member` 클래스에서 `BaseEntity`라는 것을 상속받았습니다.  
이 `BaseEntity`라는 것은 데이터의 등록 시간과 수정 시간과 같이 자동으로 추가되고 변경되어야 하는 Column들이 있을 것인데,  
이를 매번 처리하는 일은 번거롭기 떄문에 자동으로 처리할 수 있도록 설정해줍니다.  


## BaseEntity 구현
```java
package com.moon.aza.entity;

import lombok.Getter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.Column;
import javax.persistence.EntityListeners;
import javax.persistence.MappedSuperclass;
import java.time.LocalDateTime;

@Getter
@MappedSuperclass
@EntityListeners(value = {AuditingEntityListener.class })
public class BaseEntity {
    @CreatedDate
    @Column(name = "regdate", updatable = false)
    private LocalDateTime regDate;

    @LastModifiedDate
    @Column(name = "moddate")
    private LocalDateTime modDate;
}

```
`@EntityListeners(value = {AuditingEntityListener.class })`는 필드 위에 선언된 `@CreateDate`와 `@LastModifiedDate`를 탐색해 `Entity`변경 시
생성 시간을 나타내는 `redDate`와 최종 수정 시간을 자동으로 처리하는 용도인 `modDate`에 적절한 값이 지정됩니다.  
즉, JPA 내부에서 `Entity` 객체가 생성/변경되는 것은 감지하는 역할을 `AuditingEntityListener`가 합니다.  

`@MappedSuperclass`는 이 `Annotation`이 적용된 클래스는 테이블로 생성되지 않습니다. 실제 테이블은 `BaseEntity 클래스를 상속한  
`Entity`의 클래스로 Database 테이블이 생성됩니다.  

필드 `regDate`의 속성 `updatable = false`로 설정했는데, 이는 Database에 반영할 때 `regDate` 칼럼값은 변경되지 않습니다.  

## AazApplication 수정
JPA를 이용하면서 `AuditingEntityListener`를 활성화시키기 위해서는 프로젝트에 `@EnableJpaAuditing` 설정을 아래와 같이  
수정해야 합니다. 
```java
package com.moon.aza;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

@EnableJpaAuditing
@SpringBootApplication
public class AzaApplication {

	public static void main(String[] args) {
		SpringApplication.run(AzaApplication.class, args);
	}

}
```

# 회원가입(signup.html) 페이지 구현
```html
...생략
        <div class="signup">
            <div class="signup__top">
                <h1>회원가입</h1>
            </div>
            <div class="signup__container">
                <form class="needs-validation col-sm-6" th:action="@{/member/signup}" method="post"novalidate>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="email" style="font-weight:bold; margin-bottom:10px;">이메일</label>
                        <input type="email" class="form-control" name="email"  placeholder="ex) aaa@exple.com" required>
                    </div>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="password" style="font-weight:bold; margin-bottom:10px;">비밀번호</label>
                        <input type="password" class="form-control" name="password" placeholder="비밀번호는 8~16자 영문, 숫자, 특수문자를 사용하세요." required>
                    </div>
                    <div class="form-group required" style="margin-bottom: 15px;">
                        <label for="nickname" style="font-weight:bold; margin-bottom:10px;">닉네임</label>
                        <input type="text" class="form-control" name="nickname" placeholder="닉네임을 입력해주세요." required>
                    </div>
              
                    <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}"> 홈 화면으로</button>
                    <button type="submit" class="btn btn-primary signupButton"><img class="button__icon" th:src="@{/imgs/signup-icon-white.png}">회원가입</button>
                </form>
            </div>
        </div>
...생략
```
이와 같이 `회원가입(signup.html)` 페이지를 구현했습니다. Design 측면은 `Bootstrap`을 이용했습니다.  
저같은 경우는 이 페이지를 `resource/templates/aza/signup.html` 경로에 생성했습니다.  
이제 `회원가입(signup.html)` 페이지로 redirect 시켜줄 `Controller`를 구현해보겠습니다.  

# MainController 구현
이 `MainController`는 View를 반환하는 역할을 하며, 여기에는 `home`, `signup`, `login` 페이지 등을 반환하도록 하였습니다.
```java
package com.moon.aza.controller;

import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Log4j2
@RequiredArgsConstructor
@Controller
public class MainController {


    /* 회원가입 페이지 */
    @GetMapping("/signup")
    public String signup(Model model) {
        log.info("/signup");
        
        return "/aza/signup";
    }
 
}
```
`@Controller`로 이 클래스는 `Controller`임을 나타내고,   
`@@RequiredArgsConstructor`는 나중에 `final`이 붙거나 `@NotNull`이 붙은 필드의 생성자를 자동으로 생성해주는 역할을 합니다.  
그리고 `@Log4j2`로 요청한 페이지로 이동시 로그에 기록하도록 했습니다.  
마지막으로 `회원가입(signup.html)` 페이지로 이동하도록 경로를 반환하도록 하였습니다.  

# 회원가입(signup.html) 페이지
웹 브라우저에서 `http://localhost:8080/signup` 로 요청 시 아래와 같이 회원가입 페이지가 나오게 됩니다.











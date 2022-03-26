![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에는 게시판 웹에서 가장 중요한 부분인 게시판 부분에 대해서 구현해 보겠습니다.  
이 게시판 기능은 대략 게시글 작성, 조회, 수정, 삭제의 기능이 존재합니다.  
이 기능들을 구현하기 전에 게시판(Board) 도메인 설계와 게시판 페이지를 구현할 필요가 있습니다.  
또한 게시판 작성 페이지에서 글꼴 크기, 스타일, 이미지 업로드 등을 편리하게 구현하게 해주는 것을 위지윅 에디터라고 하며,    
그 중에서 **CKEditor** 를 설정하도록 하겠습니다. 
먼저 게시판(Board) 도메인 설계를 해보겠습니다.  

## Board Entity 구현
`Board` Entity를 구현하기에 앞서, 먼저 글을 작성할 때 필요한 항목들을 정리해보겠습니다.  
* **id** : 글 번호(PK)를 나타냅니다. 이를 이용해 조회, 수정, 삭제 기능을 수행합니다.  
* **title** : 글 제목을 나타냅니다.  
* **contents** : 글 내용을 나타냅니다.  
* **writer** : 글 작성자를 나타냅니다.
* **member** : 개체(Entity) 간의 관계(Relation)를 표현합니다.  

이를 토대로 `Board`를 구현해 보겠습니다.  
```java
package com.moon.aza.entity;


import lombok.*;

import javax.persistence.*;


@Builder
@ToString
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Board extends BaseEntity {
    @Id
    @GeneratedValue
    @Column(name = "board_id")
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false)
    private String contents;

    private String writer;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    // 수정 기능을 구현할 때 사용할 메서드
    public void modifyTitle(String title){this.title = title;}
    public void modifyContents(String contents){this.contents = contents;}

}

```
몇몇 부분은 `Member`를 구현할 때와 틀은 비슷하지만 하나 다른 점은 바로 `Member`객체가 있다는 것입니다.  
이는 관계형 데이터베이스에서 개체와 관계에 대한 것입니다.  

관계형 데이터베이스에서는 **일대일(1:1)**, **일대다(1:N)**, **다대일(N:1)**, **다대다(M:N)** 의 관계를 이용해서 데이터가 서로 간에  
어떻게 구성되었는지를 표현합니다. 중요한 것은 `PK(기본키, Primary Key)`와 `FK(외래키, Foreign Key)`를 어떻게 설정해서 사용하는가에 대한 설정입니다. 

여기서는 `회원(Member)`과 `게시글(Board)`의 관계를 살펴보면, **한 명의 회원은 여러 개의 게시글을 작성할 수 있습니다.**  
그렇기 때문에 `회원(Member)`쪽의 `Id`를 `게시글(Board)`에서는 **FK**로 참조하는 구조입니다.  
먼저 **FK** 를 사용하는 Board 쪽의 관계를 살펴보며, 이 경우 Board와 Member와의 관계는 **다대일(N:1)** 의 관계가 되므로 JPA에서는  
이를 의미하는 `@ManyToOne`을 적용하여 연관관계를 설정합니다.  

또한 `fetch = FetchType.LAZY`는 두 개이상의 Entity 간의 관계를 맺고 나면 쿼리를 실행하는 데이터베이스 입장에서는   
위와 같이 `@ManyToOne`의 경우 FK쪽의 Entity를 가져올 때 PK 쪽의 엔티티도 같이 가져옵니다.  
이때, 특정 Entity를 조회할 때 연관관계를 가진 모든 Entity를 같이 로딩하는 것을 `Eager Loading`이라고 합니다.  
이러한 `Eager Loading`은 연관관계가 복잡할수록 join으로 인한 성능 저하를 피할 수 없기 때문에 그와 반대되는 개념인 `Lazy Loading`으로  
처리합니다.  

`Member` 쪽에도 `Board`와의 연관관계를 설정합니다.  
## Member 수정(연관관계 설정)
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
  
    @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
    private List<Board> boards = new ArrayList<>();
  
  ...생략
}

```
`회원(Member)`에서는 많은 게시글을 생성할 수 있으니 `@OneToMany`를 설정하며 boards 필드는 `Member`에 의해 매핑되므로,  
`mappedBy="member"`로 작성했습니다. 여기서 "member"는 **Board Entity에서 Member Entity를 참조할 때 작성한 필드명입니다.**

## DTO 구현(BoardForm)
이번에는 글을 작성할 때 필요한 정보들, `사용자의 id`, `title`, `writer`, `contents`, `regDate, modDate`를 필드로 구현했습니다.  
```java
package com.moon.aza.dto;


import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Builder
@NoArgsConstructor
@AllArgsConstructor
@Data
public class BoardForm {
    private Long id;

    private String title;

    private String writer;

    private String contents;

    private LocalDateTime regDate, modDate;

}

```


## BoardController 구현
게시판에서 `글쓰기(/board/register)`버튼을 클릭했을 때, 요청을 받아 글 작성페이지를 호출하도록 구현합니다.  

```java
package com.moon.aza.controller;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.entity.Member;
import com.moon.aza.support.CurrentMember;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/board")
@Controller
public class BoardController {

    /* 글 작성 페이지 */
    @GetMapping("/register")
    public void boardRegister(@CurrentMember Member member, Model model){
        log.info("/board/register");
        model.addAttribute("member",member);
        model.addAttribute(new BoardForm());
    }

}

```
여기서 저번에 구현한 커스텀 애너테이션인 `@CurrentMember`로 로그인한 사용자의 정보를 가져옵니다.  
왜냐하면 글 작성 후 저장을 누를 때, 어떤 사용자가 저장했는지 알기 위해서 입니다.  
**DTO**의 역할을 담당할 `BoardForm` 객체를 생성하여 `Model`을 통해 전달하도록 합니다.

## 글쓰기(register.html) 페이지 구현
그 전에 `글쓰기(register.html)`에는 로그인한 사용자만 접근하도록 해야합니다. 그렇기 때문에  
글쓰기로 접근하도록 하는 버튼에는 `Thymeleaf` 문법으로 `Spring Security`를 적용합니다.  
```html
 ...생략
 <button sec:authorize="isAuthenticated()" class="content__button"
          th:onclick="|location.href='@{/board/register}'|">글쓰기</button>
 ...생략
```

```html

...생략

<div class="board__register">
            <div class="register__top">
                <h1>게시글 작성</h1>
            </div>
            <form th:action="@{/board/register}" method="post" th:object="${boardForm}">
                <input type="hidden" id="id" name="id" th:value="${member?.id}"/>
                <div class="form-group" style="font-weight:bold; margin-bottom:15px;">
                    <label>제목</label>
                    <input type="text" class="form-control" name="title" th:field="*{title}" required>
                </div>
                <div class="form-group" style="font-weight:bold; margin-bottom:15px;">
                    <label>작성자</label>
                    <input type="text" class="form-control" name="writer"  th:value="${member?.nickname}" readonly>
                </div>
                <div class="form-group" style="font-weight:bold; margin-bottom:15px;">
                    <label>내용</label>
                    <textarea class="form-control" row="5" name="contents" id="contents" th:field="*{contents}" required></textarea>
                </div><br>
                <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}"> 홈 화면으로</button>
                <button type="submit" class="btn btn-primary"><img class="button__icon" th:src="@{/imgs/register-white.png}">등록하기</button>
            </form>
</div> 

...생략

```
`Thymeleaf` 문법인 `th:object`와 `th:field`로 각각의 field들을 `boardForm`객체에 매핑시켜줍니다.  
그리고 로그인한 사용자의 `nickname`을 `writer` 필드의 값으로 설정합니다. 그리고 input의 hidden 속성으로  
사용자의 id 값을 넣어주게 되면 글 작성 후 버튼을 눌러 `/board/register`의 경로로 작성 데이터와 함께 보냅니다.  

여기까지 작성하게 되면 노멀한 글 작성 페이지 구현이 완료됩니다.  
여기서 추가적으로 텍스트 편집기인 **CKEditor** 를 설정해보도록 하겠습니다.  

## CKEditor4 설치 및 설정
[https://ckeditor.com/ckeditor-4/download/](https://ckeditor.com/ckeditor-4/download/) 에 접속하게 되면 아래와 같은 페이지로 이동하게 됩니다.  
![ckeditor-1](https://user-images.githubusercontent.com/60730405/160228299-702cdf46-3e55-44b0-bfb1-0636cb48c399.JPG)
여기서 다양한 패키지들을 다운받을 수 있고 전 여기서 **Standard Package** 를 다운받았습니다.  

다운을 받고 압축을 풀게되면 한 폴더가 생성되며 이 폴더를 아래와 같이 `/resources/static/js/`에 위치 시켜줍니다.  
(사용자에 따라 폴더 위치가 다를 수 있습니다.)  
![ckeditor-2](https://user-images.githubusercontent.com/60730405/160228441-8f633098-cf57-480e-b187-40362b4c3881.JPG)

이제 여기서 폴더 안에 있는 ckeditor.js를 게시글 작성 페이지에 import 시켜줍니다. 
```html
  
  ...생략
 <head>
        ...
        <script th:src="@{/js/ckeditor/ckeditor.js}"></script>
 </head>
  ...생략
```

그리고 페이지 밑 부분에 아래와 같이 글의 내용을 작성하는 <textarea> 태그에 지정한 id 속성 값을 통해 CKEditor 설정합니다.   
```js
  <script type="text/javascript" th:inline="javascript">
            $(document).ready(function(){
                CKEDITOR.replace('contents',{ 
                    filebrowserUploadMethod :'form',
                    filebrowserUploadUrl: '/image/upload'
           
                });
                CKEDITOR.on('dialogDefinition',function(e){
                    var dialogName = e.data.name;
                    var dialogDefinition = e.data.definition;

                    switch(dialogName){
                        case 'image':
                            //dialogDefinition.removeContents('info');
                            dialogDefinition.removeContents('Link');
                            dialogDefinition.removeContents('advanced');
                            break;
                    }
                });

            });
        </script>  
```
여기서 `이미지 업로드 주소(/image/upload)`를 정하고, CKEditor4.9.x 이상부터는 업로드는 JSON 응답을 해야하는데, 저는 그렇지 않을 것이기 때문에 `form` 속성을 설정하였습니다.  
 밑에 `CKEDITOR.ON(~`부분은 이미지 업로드를 위한 버튼을 누르게 되면 업로드 창이 하나 나올 것입니다.  
 그 창에서 업로드와 이미지 정보를 나타내는 버튼외에 나머지 버튼을 없애겠다는 의미입니다.  
 
 그러면 아래와 같이 글 작성 페이지에서 CKEditor가 적용된 것을 확인할 수 있습니다.  
 ![ckeditor-3](https://user-images.githubusercontent.com/60730405/160229007-d133587d-df87-4df5-9fd2-56dc73003d86.JPG)
  
 (이미지 업로드 창)
 ![ckeditor-4](https://user-images.githubusercontent.com/60730405/160229010-80229b92-9d1e-4b52-ad52-aeb50479e51b.JPG)


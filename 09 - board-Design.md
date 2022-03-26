![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에는 게시판 웹에서 가장 중요한 부분인 게시판 부분에 대해서 구현해 보겠습니다.  
이 게시판 기능은 대략 게시글 작성, 조회, 수정, 삭제의 기능이 존재합니다.  
이 기능들을 구현하기 전에 게시판(Board) 도메인 설계와 게시판 페이지를 구현할 필요가 있습니다.  
먼저 게시판(Board) 도메인 설계를 해보겠습니다.  

## Board Entity 구현
`Board` Entity를 구현하기에 앞서 

## DTO 구현(BoardForm)
먼저 글을 작성할 때 필요한 정보들, `사용자의 id`, `title`, `writer`, `contents`, `regDate, modDate`를 필드로 구현했습니다.  
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
그리고 나중에 구현할 **DTO**의 역할을 담당할 `BoardForm` 객체를 생성하여 `Model`을 통해 전달하도록 합니다.  



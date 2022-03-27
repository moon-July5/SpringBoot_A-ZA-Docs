![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***

이번에 구현할 것은 저번에 구현할 페이징 처리 기능을 가지고 게시글에서 목록 처리를 하고  
게시글 조회 기능을 구현해 보겠습니다.  

## MainController(게시판 목록 처리) 구현
실제 게시판 페이지에서 저번에 작성한 페이징 처리를 이용하여 글 목록이 화면에 나타나도록 합니다.  
```java
package com.moon.aza.controller;

import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.BoardService;
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
    private final BoardService boardService;

  ...생략

    /* 게시판 페이지 */
    @GetMapping("/board") // 화면에서 page 와 size를 전달받기 위해
    public String board(PageRequestDTO pageRequestDTO, Model model) {
        log.info("/board");
        model.addAttribute("result", boardService.getList(pageRequestDTO));
        return "/aza/board";
    }

}

```
게시판으로 이동하는 경로(/board)에 parameter로 `PageRequestDTO`를 이용합니다.  
Spring MVC는 parameter를 자동으로 수집해주는 기능이 있으므로, 화면에서 `page`와 `size`라는 parameter를 전달하면 `PageRequestDTO` 객체로 자동으로 수집될 것입니다.  
`Model`을 통해 결과 데이터를 `result`라는 이름으로 화면에 전달합니다.  

다음은 `board.html` 쪽으로 전달된 데이터를 이용하여 화면에 나타나도록 합니다.  

## 목록 출력(board.html)
목록 출력을 위해 테이블을 이용하였으며, 디자인은 `Bootstrap`을 이용했습니다.

```html

  ...생략

<!--게시판 영역-->
      <section class="content">
        <div class="content__top">
          <h1>게시판</h1>
          <button sec:authorize="isAuthenticated()" class="content__button"
          th:onclick="|location.href='@{/board/register}'|">글쓰기</button>
        </div>
        <div class="content__center">
          <table class="table table-hover table-striped">
            <thead>
              <tr>
                <th scope="col">제목</th>
                <th scope="col">작성자</th>
                <th scope="col">날짜</th>
              </tr>
            </thead>
            <tbody>
              <tr th:each="dto : ${result?.dtoList}">
                <th scope="row"><a th:href="@{/board/read(id=${dto.id}, page=${result.page})}">[[${dto.title}]]</a></th>
                <td>[[${dto.writer}]]</td>
                <td>[[${#temporals.format(dto.regDate, 'yyyy/MM/dd')}]]</td>
              </tr>
            </tbody>
          </table>
        </div>
        <!--페이지-->
        <div class="content__end">
          <ul class="pagination h-100 justify-content-center align-items-center">
              <li class="page-item" th:if="${result?.prev}">
                <a class="page-link" th:href="@{/board(page= ${result.start-1})}" tabindex="-1">Prev</a>
              </li>
              <li th:class=" 'page-item '+ ${result?.page == page? 'active' : ''} " th:each="page : ${result?.pageList}">
                <a class="page-link" th:href="@{/board(page= ${page})}">
                  [[${page}]]
                </a>
              </li>
              <li class="page-item" th:if="${result?.next}">
                <a class="page-link" th:href="@{/board(page= ${result.end+1})}" tabindex="-1">Next</a>
              </li>
          </ul>
        </div>
      </section>
  ...생략
```
`Controller`에서 보낸 데이터인 `result`를 가지고 `BoardForm`들을 출력합니다.  
`Thymeleaf` 문법 중에 반복문인 `th:each`로 `PageResultDTO` 안에 들어있는 `dtoList`를 반복 처리합니다. 날짜는 년/월/일의 포맷으로 출력합니다.  
그리고 게시글 제목을 클릭 시 조회할 수 있도록 `/board/read`으로 요청하도록 처리하고 `parameter 값(id, page)`도 전달하도록 합니다.  

또한 화면에 페이지가 출력되도록 `Thymeleaf` 문법으로 구현합니다.  
이전과 다음부분은 `if`를 이용해서 처리하고, 중간에 페이지는 현재 페이지를 체크해서 'active'라는 이름의 클래스가 출력되도록 합니다.  
'active' 클래스가 추가되면 현재 자신이 어떤 페이지 번호에 위치했는지 알 수 있습니다.  

## 게시판(board.html) 확인
아래의 이미지같이 게시글 목록과 페이지 번호들이 출력된 것을 확인할 수 있습니다.  
`Prev`와 `Next`는 게시글이 10개보다 적기 때문에 나타나지 않습니다.  

![board-read-1](https://user-images.githubusercontent.com/60730405/160267178-7e326d01-2449-48ca-af6f-2085a03c16c5.png)

이제는 게시글 제목을 클릭 시 나타나는 조회 화면을 구현해 보겠습니다.  

## BoardService(조회 기능) 구현
```java
package com.moon.aza.service;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.dto.PageResultDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.BoardRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

@RequiredArgsConstructor
@Transactional
@Service
public class BoardService {
    private final BoardRepository boardRepository;

  ...생략
  
    // 게시글 조회
    public BoardForm read(Long id){
        Optional<Board> result = boardRepository.findById(id);

        return result.isPresent()? entityToDto(result.get()):null;
    }

    // entity -> dto
    public BoardForm entityToDto(Board board){
        BoardForm dto = BoardForm.builder()
                .id(board.getId())
                .title(board.getTitle())
                .contents(board.getContents())
                .writer(board.getWriter())
                .regDate(board.getRegDate())
                .modDate(board.getRegDate())
                .build();
        return dto;
    }
}

```
게시글을 조회하는 `read()` 메서드에는 글 번호(id)를 parameter로 받아 `BoardRepository`에서 글 번호(id)를 통해 특정 게시글을 탐색합니다.  
만약 게시글이 존재하면 전에 구현했던 `entityToDto()` 메서드로 얻어온 Entity 객체를 DTO 형태로 변환하여 return 하도록 합니다.  

## BoardController(조회 기능) 구현
```java
package com.moon.aza.controller;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.service.BoardService;
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
    private final BoardService boardService;

  ...생략
  
    @GetMapping({"/read","/modify"})
    public void boardRead(@CurrentMember Member member, Long id, PageRequestDTO pageRequestDTO, Model model){
        log.info("id : "+id);
        BoardForm boardForm = boardService.read(id);
        model.addAttribute("boardForm", boardForm);
        if(member != null)
            model.addAttribute(member);
    }

}

```
`/board/read` 경로와 나중에 수정 화면으로 구현할 `/board/modify` 경로로 요청이 올 시, GET 방식으로 글 번호인 id 값을 받습니다.  
그리고 특정 게시글을 작성한 작성자만 수정과 삭제 할 수 있도록 조회 화면에 로그인한 사용자의 정보인 `Member`도 parameter로 사용합니다.  
또한 나중에 목록 페이지로 돌아가는 데이터를 같이 저장하기 위해서 `PageRequestDTO`를 parameter로 같이 사용합니다.  

`BoardService`에서 구현한 `read()` 메서드에 id 값을 받아 조회 기능을 수행합니다.  
그리고 "boardForm"라는 이름으로 전달합니다.  

## 조회(read.html) 페이지 구현
```html
  ...생략
        <div class="board__read">
        <form th:action="@{}" method="Post">
           <div class="read__top">
               <div class="title">[[${boardForm.title}]]</div>
           </div>
           <hr>
           <div class="read__info">
                <div class="writer">[[${boardForm.writer}]]</div>
                <div class="date">[[${#temporals.format(boardForm.regDate, 'yyyy/MM/dd HH:mm')}]]</div>
           </div>
           <hr>
           <div class="read__contents" style="margin-bottom: 5%;">
               <div th:utext="${boardForm.contents}"></div>
           </div>
           <div class="read__button">
               <button type="button" class="btn btn-secondary modifyButton">수정</button>
               <button type="submit" class="btn btn-danger removeButton">삭제</button>
               <button type="button" class="btn btn-info" th:onclick="|location.href='@{/board(page=${pageRequestDTO.page})}'|">목록</button>
           </div>
        </form>
        </div>
        
        <script type="text/javascript" th:inline="javascript">
            $(document).ready(function(){
                var currentMember = [[${member?.nickname}]];
                var writer = [[${boardForm.writer}]];

                function showButton(){
                    if(currentMember != writer){
                        $('.modifyButton').hide();
                        $('.removeButton').hide();
                    }
                }
                showButton();
            });
        </script>
  ...생략
```
`BoardController`에서 보낸 "boardForm"이라는 이름으로 글의 내용을 출력합니다.  
그리고 수정과 삭제기능도 나중에 구현할 것이기 때문에 밑에 버튼을 만들어 놓고 `javascript`로 현재 로그인한 사용자의 정보와 작성자의 정보가 일치하면  
수정과 삭제버튼이 나타나도록 하고 아니면 숨기도록 구현합니다.  

목록화면으로 돌아가는 버튼 또한 생성하여 page라는 parameter로 이동하도록 합니다.  

## 조회 화면 결과 
로그인하지 않고 게시글 조회할 시 나타나는 화면은 아래의 이미지와 같습니다.  
![board-read-2](https://user-images.githubusercontent.com/60730405/160267771-1ea85165-a97e-40c2-9cfb-08e8bd4775ca.JPG)

현재 로그인한 사용자와 작성자와 같은 사용자이면 나타나는 화면은 아래의 이미지와 같습니다.  
![board-read-3](https://user-images.githubusercontent.com/60730405/160267775-c6e91b14-8b45-4b5d-b303-565375d29bde.JPG)

***
다음에는 게시글을 수정하고 삭제하는 기능을 구현해 보겠습니다.  


![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 기능은 **내가 쓴 게시글 확인** 기능입니다.    
자신이 게시판에 쓴 게시글을 확인하는 기능으로 여기에 추가적으로 각 게시글마다 체크박스가 존재하여 체크한 게시글만 **선택 삭제**가  
가능하도록 기능을 구현해보도록 하겠습니다.  

일단 이번에는 내가 쓴 게시글만 출력이 되도록 구현해 보겠습니다.  

저번에 [11 - Paging]([http://www.naver.com/](https://github.com/moon-July5/SpringBoot_A-ZA-Docs/blob/main/11%20-%20Paging.md))에서 구현한 페이징 처리를 이용하여 구현해 보겠습니다.  

## BoardService 구현
```java
package com.moon.aza.service;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.dto.PageResultDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.entity.QBoard;
import com.moon.aza.repository.BoardRepository;
import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.dsl.BooleanExpression;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;
import java.util.stream.Collectors;

@RequiredArgsConstructor
@Transactional
@Service
public class BoardService {
    private final BoardRepository boardRepository;

  ...생략
 
    // 페이징 처리
    public PageResultDTO<BoardForm, Object[]> getMyList(PageRequestDTO requestDTO, Long id){
        Pageable pageable = requestDTO.getPageable(Sort.by("id").descending());

        BooleanBuilder booleanBuilder = new BooleanBuilder();
        QBoard qBoard = QBoard.board;
        BooleanExpression expression1 = qBoard.id.gt(0L);
        BooleanExpression expression2 = qBoard.member.id.eq(id);
        booleanBuilder.and(expression1);
        booleanBuilder.and(expression2);

        Page<Object[]> result = boardRepository.searchBoard(booleanBuilder, pageable);

        Function<Object[], BoardForm> fn = (arr -> entityToDto(
                (Board) arr[0],
                (Long) arr[1],
                (Long) arr[2]
        ));
        return new PageResultDTO<>(result, fn);
    }

    // entity -> dto
    public BoardForm entityToDto(Board board, Long commentCnt, Long likesCnt){
        BoardForm dto = BoardForm.builder()
                .id(board.getId())
                .title(board.getTitle())
                .contents(board.getContents())
                .writer(board.getWriter())
                .regDate(board.getRegDate())
                .modDate(board.getRegDate())
                .build();
        dto.setCommentCnt(commentCnt.intValue());
        dto.setLikesCnt(likesCnt.intValue());
        return dto;
    }
}

```
페이징 처리를 위해 `getList()` 라는 메서드를 구현했습니다.  
이번에는 자신이 쓴 게시글만 나타나도록 `getMyList()` 메서드를 구현합니다.  

`getList()` 메서드와 다른 점은 저 같은 경우에 검색기능을 이용할 때 사용했던 `BooleanBuilder`에 조건들을 미리 추가했습니다.  
어떤 조건들이냐면 게시글의 `ID 값`이 0보다 크고 `Member`의 `ID 값`이 동일한 게시글만 출력하도록 처리했습니다.  
이렇게 되면 본인이 작성했던 게시글만 출력이 될 것입니다.  

## MainController 구현
```java
package com.moon.aza.controller;

import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.BoardService;
import com.moon.aza.service.LikesService;
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
  
    /* 내가 쓴 게시글 */
    @GetMapping("/myboard")
    public String myBoard(PageRequestDTO pageRequestDTO,@CurrentMember Member member ,Model model){
        log.info("/myboard");
        model.addAttribute("result", boardService.getMyList(pageRequestDTO, member.getId()));
        return "/aza/myboard";
    }
}

```
`MainController`에 `/myboard` 경로로 요청이 올 경우, `뷰(View)`를 반환하도록 구현합니다.  
여기서 아까 구현했던 `BoardService`의 `getMyList()`를 호출하여 `result`라는 이름으로 전달하며, 화면에서 `page`와 `size`를 전달받기 위해  
`PageRequestDTO`와 본인이 작성한 게시글만 보도록 `Member`를 전달합니다.  

이제 `뷰(View)`를 구현합니다.  

## 내가 쓴 게시글(myboard.html) 페이지 구현  
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
        layout:decorate="~{/layout/basic}">
    <head>
        <link rel="stylesheet" th:href="@{/css/index.css}">
    </head>
    <th:block layout:fragment="content">
      <!--게시판 영역-->
      <section class="content">
        <div class="content__top">
          <h1>게시판</h1>
          <button class="btn btn-danger deleteBtn">삭제</button>
        </div>
        <div class="content__center">
          <table class="table table-hover table-striped">
            <thead>
              <tr>
                <th scope="col">제목</th>
                <th scope="col">작성자</th>
                <th scope="col">날짜</th>
                <th scope="col">추천</th>
                <th scope="col"><input type="checkbox" id="checkAllDelete" name="checkAllDelete"></th>
              </tr>
            </thead>
            <tbody>
              <tr th:if="${result.totalPage}==0">
                <td colspan="5" style="text-align: center;">작성한 게시글이 없습니다.</td>
              </tr>
              <tr th:each="dto : ${result?.dtoList}">
                <th scope="row"><a th:href="@{/board/read(id=${dto.id}, page=${result.page})}">[[${dto.title}]] ([[${dto.commentCnt}]])</a></th>
                <td>[[${dto.writer}]]</td>
                <td>[[${#temporals.format(dto.regDate, 'yyyy/MM/dd')}]]</td>
                <td style="color: red;"> [[${dto.likesCnt}]]</td>
                <td><input type="checkbox" name="checkDelete" th:value="${dto.id}"></td>
              </tr>
            </tbody>
          </table>
         
        </div>
   
        <!--페이지-->
        <div class="content__end">
          <ul class="pagination h-100 justify-content-center align-items-center">
              <li class="page-item" th:if="${result?.prev}">
                <a class="page-link" th:href="@{/myboard(page= ${result.start-1})}" tabindex="-1">Prev</a>
              </li>
              <li th:class=" 'page-item '+ ${result?.page == page? 'active' : ''} " th:each="page : ${result?.pageList}">
                <a class="page-link" th:href="@{/myboard(page= ${page})}">
                  [[${page}]]
                </a>
              </li>
              <li class="page-item" th:if="${result?.next}">
                <a class="page-link" th:href="@{/myboard(page= ${result.end+1})}" tabindex="-1">Next</a>
              </li>
          </ul>
        </div>
      </section>
      <script type="text/javascript" th:inline="javascript">
        $(document).ready(function(){
            $("#checkAllDelete").click(function() {
                if($("#checkAllDelete").is(":checked")) $("input[name=checkDelete]").prop("checked", true);
                else $("input[name=checkDelete]").prop("checked", false);
            });

            $("input[name=chk]").click(function() {
                var total = $("input[name=checkDelete]").length;
                var checked = $("input[name=checkDelete]:checked").length;

                if(total != checked) $("#checkAllDelete").prop("checked", false);
                else $("#checkAllDelete").prop("checked", true); 
            });
        });
      </script>
    </th:block>
</html>
```
`MainController`에서 전달한 `result`를 이용하여 데이터를 출력합니다.  
`게시글(board.html)`와 거의 유사합니다. 다른 점은 행마다 `CheckBox`가 존재하며, `Javascript`의 `JQuery`를 이용하여 `CheckBox`를 전체 선택하는 기능과 전체 해제하는 기능을 구현합니다.  
또한 본인이 작성한 게시글이 하나도 없을 경우, `작성한 게시글이 없습니다.`이 없다는 문구를 표시하도록 합니다.  

## 결과
`내가 쓴 게시글` 확인
![1](https://user-images.githubusercontent.com/60730405/168981083-533de504-166c-4c2b-992d-890b859bb756.png)  

`MySQL Workbench`에서 내가 쓴 게시글만 출력이 됐는지 확인  
![2](https://user-images.githubusercontent.com/60730405/168981089-74c471c1-102a-4a80-b3d5-f5bb0859bb5d.JPG)


![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 기능은 `내가 쓴 게시글`에서 `CheckBox`를 선택하여 선택된 게시글만 삭제하도록 기능을 구현해 보겠습니다.  

일단 Database에서 게시글과 연관된 댓글이나 추천 또한 같이 연쇄적으로 삭제가 되어야 하기 때문에 각 `Repository`에서  
**Delete**하는 `Query` 문을 작성하도록 합니다.  

## Repository 구현(Comment, Likes)
각 Repository에 구현할 메서드들은 각 게시글마다 존재하는 댓글, 추천을 삭제하는 기능을 합니다.  
이 메서드들은 `@Query`를 통해 SQL 언어로 작성하며, 각 Repository의 메서드들은 테이블의 이름만 다르고 틀은 똑같습니다.  

<details>
<summary>CommentRepository</summary>
  
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import com.moon.aza.entity.Comment;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;


@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
  ...생략

    @Modifying
    @Transactional
    @Query("delete from Comment c where c.board.id in :ids")
    void deleteAllByBoardIdIn(@Param("ids") List<Long> ids);

}

```
</details>

<details>
<summary>Likesepository</summary>
  
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Likes;
import com.moon.aza.repository.likes.LikesCustomRepository;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;


@Repository
public interface LikesRepository extends JpaRepository<Likes, Long>, LikesCustomRepository {
   ...생략

    @Modifying
    @Transactional
    @Query("delete from Likes l where l.board.id in :ids")
    void deleteAllByBoardIdIn(@Param("ids") List<Long> ids);
}

```
</details>

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
import com.moon.aza.repository.CommentRepository;
import com.moon.aza.repository.LikesRepository;
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
    private final CommentRepository commentRepository;
    private final LikesRepository likesRepository;

    public void checkRemove(List<String> ids){
        List<Long> result = ids.stream().map(Long::parseLong).collect(Collectors.toList());
        likesRepository.deleteAllByBoardIdIn(result);
        commentRepository.deleteAllByBoardIdIn(result);
        boardRepository.deleteAllByIdIn(result);
    }

  ...생략
}

```
`BoardService`에는 체크된 게시물만을 삭제하는 메서드인 `checkRemove()`를 구현합니다.  
`List<String>` 형태로 전달받기 때문에 이를 `List<Long>` 형태로 변환할 필요가 있습니다.  
`List<Long>` 형태로 변환된 것을 전에 각 Repository에 구현한 특정 게시글의 댓글, 추천을 삭제하는 `deleteAllByBoardIdIn()`을 호출합니다.  
또한 마지막으로 게시글을 삭제하기 위해 `BoardRepository`에는 저번에 `회원 탈퇴` 기능을 구현할 때 구현한 `deleteAllByIdIn()` 메서드를 재사용합니다.

## MyBoardController 구현
```java
package com.moon.aza.controller;

import com.moon.aza.service.BoardService;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/myboard")
@Controller
public class MyBoardController {
    private final BoardService boardService;

    @ResponseBody
    @DeleteMapping("/remove")
    public ResponseEntity<String> remove(@RequestBody List<String> ids){
        log.info("ids : {}",ids);
        boardService.checkRemove(ids);
        return new ResponseEntity<>("success",HttpStatus.OK);
    }
}

```
`MyBoardController`에는 삭제 버튼을 클릭 시 `/myboard/remove` 경로로 요청이 올 때, 동작하는 메서드를 구현합니다.  
이 메서드에는 삭제할 게시글들의 `ID 값`을 `List<String>` 형태로 받습니다. 
그리고 `ID 값`들을 전에 구현한 `BoardService`의 `checkRemove()`에 전달하여 호출합니다.  


## 내가 쓴 게시글(myboard.html) 페이지 수정
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

            $('.deleteBtn').click(function(){
                var ids = [];
                $("input[name=checkDelete]:checked").each(function(){
                    ids.push($(this).val());
                });

                console.log(ids);

                if(ids==""){
                    alert("선택된 항목이 존재하지 않습니다.");
                    return false;
                }

                if(confirm("정말 삭제하시겠습니까?")){
                    $.ajax({
                        url : '/myboard/remove',
                        type : 'DELETE',
                        contentType : 'application/json; charset=utf-8',
                        data : JSON.stringify(ids),
                    success : function(result){
                        alert("성공적으로 삭제되었습니다.");
                        self.location.reload();
                    },
                    error : function(request, status, error){
                            console.log("code:"+request.status+"\n"+"message:"+request.responseText+"\n"+"error:"+error);
                    }
                    });
                }
            });
        });
      </script>
    </th:block>
</html>
```
여기서 삭제 버튼(deleteBtn)을 눌렀을 때 동작하도록 구현합니다.  
`CheckBox`에는 게시글의 `ID 값`이 저장되어 있기 때문에 체크하고 버튼을 누르게 되면 `ID 값`들이 배열 형태로 저장되게 됩니다.  
이 저장된 값들을 `MyBoardController`에 구현한 `/myboard/remove` 경로로 전달합니다.  
그러면 특정 게시글의 삭제가 동작하며, 성공적으로 삭제가 완료되면 삭제되었다는 알림창이 출력하고 새로고침이 됩니다.  
  
## 결과
`내가 쓴 게시글`에서 특정 게시글들의 `CheckBox`를 체크하고 삭제 버튼을 눌렀을 때
![3](https://user-images.githubusercontent.com/60730405/169020805-53b51b3f-fc89-4ff4-926f-6826e4ea0b4c.JPG)

성공적으로 삭제 처리가 완료된 것을 확인
![4](https://user-images.githubusercontent.com/60730405/169020803-e4851f62-3fbb-4a2d-81d9-8e1c2b26ba4a.JPG)

게시판(board.html)을 확인해보면 특정 게시글이 삭제된 것을 확인
![5](https://user-images.githubusercontent.com/60730405/169020802-1b4616e9-1dbc-467b-984d-31db24de6cb2.JPG)

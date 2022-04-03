![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이전에는 입력한 댓글을 Database에 저장하는 것까지 했습니다.  
이번에는 Database에 저장된 댓글들을 게시글 댓글 부분에 출력하는 것까지 해보겠습니다.  

## CommentRepository 구현
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import com.moon.aza.entity.Comment;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;


@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
    @EntityGraph(attributePaths = {"member"}, type = EntityGraph.EntityGraphType.FETCH)
    List<Comment> findByBoard(Board board);

}
```
특정 게시글에 대한 댓글은 `findByBoard()` 메서드로 추출할 수 있습니다. 하지만 그냥 `findByBoard()`메서드만 사용하면 문제가 발생할 것입니다.     
왜냐하면 `Comment` 클래스의 `Member`에 대한 Fetch 방식이 `Lazy`이기 때문에  
한 번에 `Comment` 객체와 `Member` 객체를 조회할 수 없기 때문에 발생하는 문제입니다.  

이 문제를 해결할 수 있는 방법 중에 `@EntityGraph`를 이용하여 `Comment` 객체를 가져올 때 `Member` 객체를 로딩하는 방법이 있습니다.  
기본적으로 JPA를 이용하는 경우 연관관계의 fetch 속성 값은 `Lazy`로 지정합니다.  
`@EntityGraph`는 이러한 상황에서 특정 기능을 수행할 때만 `Eager` 로딩을 하도록 지정할 수 있습니다.  

`@EntityGraph`에서 `attributePaths` 속성은 로딩 설정을 변경하고 싶은 속성의 이름을 배열로 명시하도록 합니다.  
`type` 속성은 `@EntityGraph`를 어떤 방식으로 적용할 것인지를 설정합니다. 거기서 `FETCH`는 `attributePaths`에 명시한 속성은  
`Eager`로 처리하고 나머지는 `Lazy`로 처리합니다.  

이렇게해서 `Comment`를 처리할 때 `@EntityGraph`를 적용해서 `Member`도 같이 로딩할 수 있도록 변경합니다.  

## CommentService 구현
```java
package com.moon.aza.service;

import com.moon.aza.dto.CommentDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Comment;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.CommentRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@RequiredArgsConstructor
@Service
public class CommentService {
    private final CommentRepository commentRepository;

    // 각 게시글 댓글 출력
    public List<CommentDTO> getListOfComment(Long boardId){
        Board board = Board.builder().id(boardId).build();
        List<Comment> result = commentRepository.findByBoard(board);
        return result.stream().map(comment -> entitiesToDTO(comment)).collect(Collectors.toList());
    }
  
    // Entity -> DTO
    public CommentDTO entitiesToDTO(Comment comment){
        CommentDTO commentDTO = CommentDTO.builder()
                .commentNum(comment.getCommentNum())
                .text(comment.getText())
                .boardId(comment.getBoard().getId())
                .memberId(comment.getMember().getId())
                .nickname(comment.getMember().getNickname())
                .regDate(comment.getRegDate())
                .modDate(comment.getModDate())
                .build();

        return commentDTO;
    }
    ...생략
}

```
댓글 출력을 위해 `Entity`를 `DTO` 객체로 변환할 필요가 있습니다.  
그래서 `entitiesToDTO()`를 이용하여 `Comment`를 `CommentDTO`로 변환하고 반환합니다.  

그리고 `entitiesToDTO()` 메서드와 `CommentRepository`에서 작성한 `findByBoard()` 메서드를 이용하여 댓글 목록들을 출력하도록 합니다.  
댓글 목록은 `getListOfComment()`에서 게시글의 번호를 전달받으면 특정 게시글에 속한 댓글들을 탐색할 것이고  
탐색한 댓글들을 `CommentDTO` 객체로 변환하도록 합니다.  

## CommentController 구현
```java
package com.moon.aza.controller;

import com.moon.aza.dto.CommentDTO;
import com.moon.aza.service.CommentService;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/comments")
@RestController
public class CommentController {
    private final CommentService commentService;

    // 댓글 목록
    @GetMapping("/{boardId}/all")
    public ResponseEntity<List<CommentDTO>> getList(@PathVariable("boardId") Long boardId){
        log.info("boardId : "+boardId);
        List<CommentDTO> commentList = commentService.getListOfComment(boardId);

        return new ResponseEntity<>(commentList, HttpStatus.OK);
    }
  ...생략
}

```
`CommentController`에서 `/comments/{게시글 번호}/all` 경로로 요청을 받으면 댓글 목록을 출력하는 함수인 `getList()` 메서드를 호출합니다.  
게시글 번호(boardId)를 전달받으면 여태까지 작성한 `commentService`의 `getListOfCommenet()`로 보내줘서 얻은 결과를 반환합니다.  

## 게시글(read.html) 페이지 구현(javascript)
```javascript
 <script type="text/javascript" th:inline="javascript">
            $(document).ready(function(){
                var currentMember = [[${member?.nickname}]];
                var writer = [[${boardForm.writer}]];
                var memberId = [[${member?.id}]];
                var boardId =[[${boardForm.id}]];
                var text = $('textarea[name="text"]');
              
              ...생략
           
            });
            // 날짜 포맷 변경
            function formatDate(regDate){
                var date = new Date(regDate);
                return date.getFullYear() + '.'+ ('0'+(date.getMonth()+1)).slice(-2) + '.' + ('0'+date.getDate()).slice(-2) + '.  ' 
                + ('0'+date.getHours()).slice(-2) + ':'+('0'+date.getMinutes()).slice(-2)+' ';

            }
           // 게시판 댓글 목록
           function getBoardComments() {
                var boardId =[[${boardForm.id}]];
                var currentMember = [[${member?.nickname}]];
                var writer = [[${boardForm.writer}]];
                    $.getJSON("/comments/"+boardId+"/all", function(arr){
                        var str="";
                        $.each(arr, function(idx, comment){
                            str += '  <li id="comment__'+ comment.commentNum+'">';
                            str += '    <div class="commentWriter d-flex">'+comment.nickname;
                            if(writer == comment.nickname){
                                str += '    <div class="sign">작성자</div>';
                            }
                            str += '        <div class="date">'+formatDate(comment.regDate)+'</div></div>';
                            str += '    <div class="d-flex">';
                            str += '    <div class="text-monospace mr-1 flex-fill">'+comment.text+'</div>';
                            if(currentMember == comment.nickname){
                            str += '    <button id="modify" class="badge btn-warning">수정</button>';
                            str += '    <button class="badge btn-danger">삭제</button>';
                            }
                            str += '    </div>';
                            str += '  </li>';
                        });
                        $(".commentList ul.list-group").html(str);
                    });
            }
        
            getBoardComments();
        </script>
```
`getBoardComments()`메서드에 `$.getJSON`을 이용하여 JSON 형태롤 받아서 출력하도록 합니다.  
`$.each`로 각 댓글 리스트를 반복하면서 동작합니다. 여기에 각 댓글 정보들(작성자, 내용, 작성날짜 등)도 같이 출력합니다.  
여기서 현재 로그인한 사용자와 댓글 작성자를 비교하여 나중에 구현할 댓글 수정, 삭제 버튼을 나타나도록 합니다.  
그리고 전에 작성했던 `ul 태그에 commentList`에 댓글 목록을 작성합니다.  

여기서 중요한 점은 `$(document).ready(function(){});` 바깥 쪽에 작성하도록 합니다.  
왜냐하면 `DOM(document)` 구조가 동적으로 변경이 될 경우 변경된 DOM에서 onclick등으로 jQuery에서 정의한 함수(function)를 실행하면  
**functionName is not defined** 오류출력과 함께 동작하지 않는 경우가 있습니다.  
그 원인은 jQuery는 처음 DOM 구조가 완료되고 난뒤에 동작하기때문에 ( $(document).ready )  
이후 구조가 변경된 DOM에서 onclick등으로 실행되는 함수는 작동이 되지 않습니다.  
나중에 댓글 수정/삭제 버튼을 구현할 것인데 이 버튼들이 동작하지 않을 수 있습니다.  
그래서 이렇게 바깥쪽에 함수를 작성합니다.  

코드 내용을 정리하자면 코드 마지막에 `getBoardComments()`를 호출하여 페이지가 열리면  
JQeury의 `getJSON()`을 이용해서 `CommentController`를 호출하게 되고, `commentList`라는 클래스 속성으로 지정된 `<div>` 태그의 내용물을 채우게 됩니다. 위의 코드가 반영되면 게시글과 함께 댓글들도 같이 출력됩니다.  

한번 정상적으로 댓글이 출력되는지 확인해 보겠습니다.  

## 결과
게시글에 댓글이 출력되는지 확인  
![comment-list-1](https://user-images.githubusercontent.com/60730405/161430336-0884a155-a40c-4f8a-b7b0-7e4354b11f52.png)  

***
여태까지 댓글 목록을 출력하는 기능을 구현했습니다.  
다음에는 댓글을 수정/삭제하는 기능을 구현해 보겠습니다.  



  

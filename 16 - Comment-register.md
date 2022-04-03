![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
전에는 댓글(Comment) 도메인을 설계했습니다.  
이번에 구현할 것은 작성한 댓글을 Database에 등록하는 것까지 구현해 보겠습니다.  

## CommentRepository 구현
실제 Database에 접근하기 위해 `CommentRepository` 인터페이스를 구현합니다.  
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

}
```

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


@RequiredArgsConstructor
@Service
public class CommentService {
    private final CommentRepository commentRepository;

    // 댓글 저장
    public Long save(CommentDTO commentDTO){
        Comment comment = dtoToEntity(commentDTO);
        commentRepository.save(comment);
        return comment.getCommentNum();
    }
 

    // DTO -> Entity
    private Comment dtoToEntity(CommentDTO commentDTO){
        Comment comment = Comment.builder()
                .text(commentDTO.getText())
                .member(Member.builder().id(commentDTO.getMemberId()).build())
                .board(Board.builder().id(commentDTO.getBoardId()).build())
                .build();

        return comment;
    }
    
}

```
일단 먼저 전달받은 `CommentDTO` 객체를 `Entity` 객체로 변환할 필요가 있습니다.  
그래서 `dtoToEntity()` 메서드를 이용하여 `DTO`를 `Entity`객체로 변환하고 반환합니다.  
이 `dtoToEntity()` 메서드로 댓글을 Database에 저장하도록 동작하는 `save()` 메서드에 활용하여 댓글을 저장합니다.  
여기서 저같은 경우는 Database에 저장 후 `댓글 번호(commentNum)`를 반환하도록 했습니다.  

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

    // 댓글 저장
    @PostMapping("/{boardId}")
    public ResponseEntity<Long> addComment(@RequestBody CommentDTO commentDTO){
        log.info("CommentDTO : "+commentDTO);
        Long commentNum = commentService.save(commentDTO);
        return new ResponseEntity<>(commentNum, HttpStatus.OK);
    }
}

```
`@Controller`와 `@ResponseBody`를 합친 기능을 하는 `@RestContoller`로 `CommentController`를 구현합니다.  
`@PostMapping`으로 `/comments/{boardId(게시글 번호)}`로 요청이 올 경우 호출하는 `addComment()` 메서드로 댓글 저장 기능을 수행하도록 합니다.  
`@RestController`는 응답할 때 `JSON` 형태로 Body에 담겨 응답합니다.  
여기에 `ResponseEntity`로 이 Body와 header 정보, 상태 코드 등을 담을 수 있도록 합니다.  
그리고 매개변수에 `@RequestBody`로 Body에 담긴 값을 Java 객체로 변환하도록 합니다.  

## 게시글(read.html) 페이지 수정(Javascript)
이번에는 댓글 등록 처리를 위해 `등록` 버튼을 눌렀을 때 동작하도록 `Javascript JQuery`로 Event 처리를 구현합니다.  
```html
  ...생략
            <div class="commentList">
                <h4>댓글 </h4>
                <ul class="list-group"></ul>
            </div>
            <div class="comment">
                <div class="comment__input">
                    <textarea class="form-control text"  name="text" placeholder="로그인을 하셔야 댓글을 작성하실 수 있습니다."></textarea>
                    <button type="button" class="btn btn-dark commentSave">등록</button>
                </div>
            </div>
  ...생략
```

```javascript
<script type="text/javascript" th:inline="javascript">
            $(document).ready(function(){
                var currentMember = [[${member?.nickname}]];
                var writer = [[${boardForm.writer}]];
                var memberId = [[${member?.id}]];
                var boardId =[[${boardForm.id}]];
                var text = $('textarea[name="text"]');
                
                ...생략
       
                // 댓글 저장
                $('.commentSave').click(function(e){
                    var data = {boardId : boardId, text : text.val(), memberId : memberId };
                    console.log(data);
                    
                    if(data.text==""){
                        alert("댓글을 작성해주세요!");
                        return;
                    }

                    $.ajax({
                        url : '/comments/'+boardId,
                        type : 'POST',
                        contentType : 'application/json; charset=utf-8',
                        dataType : 'json',
                        data : JSON.stringify(data),
                    success : function(result){
                        console.log("success: "+result);
                        self.location.reload();
                    },
                    error : function(error){
                        console.log("error : "+error);
                    }
                    });
                });
</scprit>
```
`등록(commentSave)` 버튼을 클릭하면 `게시글의 번호(boardId)`, `사용자 번호(memberId)`, `댓글 내용(text)`을 `JSON` 형태의 데이터로  
`Post` 타입으로 만들어서 비동기 통신인 `ajax`로 전송하게 합니다. `경로(url)`는 아까 `CommentController`에서 구현한 `@PostMapping` 주소를 저정합니다.    
만약 전송이 성공하면 `self.location.reload()`를 이용하여 URL을 다시 호출합니다. 이를 통해서 나중에 구현할 `댓글 목록`이 변화여 갱신하게 됩니다.  

이제 댓글이 정상적으로 Database의 `comment` 테이블에 저장되는지 확인해 보도록 하겠습니다.  

## 결과
입력창에서 댓글 작성  
![comment-register-1](https://user-images.githubusercontent.com/60730405/161427355-cf1a0b97-42fd-4c64-8468-5d1dd4ebf684.JPG)  

`MySQL Workbench`에서 데이터가 정상적으로 저장되었는지 확인  
![comment-register-2](https://user-images.githubusercontent.com/60730405/161427357-80092dfb-a8c2-4042-b572-c839711b6e04.JPG)

***
여기까지 댓글을 작성하여 실제 Database에 저장하는 것까지 구현했습니다.  
다음에는 이 Database에 저장된 댓글을 불러와서 게시글에 출력하는 것을 구현해 보도록 하겠습니다.  

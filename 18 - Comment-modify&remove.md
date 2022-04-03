![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
저번에는 Database에 저장된 댓글 목록들을 게시글 페이지에 출력하는 것까지 했습니다.  
이번에는 댓글을 수정과 삭제 기능을 구현해 보겠습니다.  
동작하는 로직은 간단하기 때문에 같이 설명하겠습니다.  

## CommentService(댓글 수정 & 삭제) 구현
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
import java.util.Optional;
import java.util.stream.Collectors;

@RequiredArgsConstructor
@Service
public class CommentService {
    private final CommentRepository commentRepository;

  ...생략
  
    // 댓글 수정
    public void modify(CommentDTO commentDTO){
        Optional<Comment> result = commentRepository.findById(commentDTO.getCommentNum());

        if(result.isPresent()){
            Comment comment = result.get();
            comment.changeText(commentDTO.getText());

            commentRepository.save(comment);
        }
    }
    // 댓글 삭제
    public void remove(Long commentNum){
        commentRepository.deleteById(commentNum);
    }
    
  ...생략

}

```
`CommentService`에 댓글 수정 메서드인 `modify()`와 댓글 삭제 메서드인 `remove()` 메서드를 구현합니다.  

`modify()` 메서드에는 `댓글 번호(commentNum)`로 특정 댓글을 탐색합니다.  
그리고 저번에 구현한 `changeText()` 메서드로 수정한 내용을 전달받아 `commentRepository`의 `save()` 메서드로 저장합니다.  

`remove()` 메서드는 정말 간단하게 `commentRepository`의 `deleteById()` 메서드로 `특정 댓글의 번호(commentNum)`로 탐색 후 삭제를 수행합니다.  

## CommentController(수정 & 삭제) 구현
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

    ...생략
    
    // 댓글 수정
    @PutMapping("/{boardId}/{commentNum}")
    public ResponseEntity<Long> modifyComment(@PathVariable("commentNum") Long commentNum, @RequestBody CommentDTO commentDTO){
        log.info("CommentDTO : "+commentDTO);
        commentService.modify(commentDTO);
        return new ResponseEntity<>(commentNum, HttpStatus.OK);
    }

    // 댓글 삭제
    @DeleteMapping("/{boardId}/{commentNum}")
    public ResponseEntity<Long> removeComment(@PathVariable("commentNum") Long commentNum){
        log.info("commentNum : "+commentNum);
        commentService.remove(commentNum);
        return new ResponseEntity<>(commentNum, HttpStatus.OK);
    }

}

```
댓글 수정 메서드인 `modifyComment()`에는 `@PutMapping`으로 `/comments/{게시글 번호}/{댓글 번호}`로 요청이 오면 호출하도록 합니다.  
`댓글 번호(commentNum)`와 `댓글 정보들(CommentDTO)`을 전달받아 `commentService`의 `modify()` 메서드로 전달하여 댓글을 수정하도록 수행합니다.  

댓글 삭제 메서드인 `removeComment()`에는 `@DeleteMapping`으로 댓글 수정 메서드와 같은 경로로 요청이 오면 호출하도록 합니다.  
`댓글 번호(commentNum)`만 전달받아 댓글 삭제를 수행하도록 합니다.  

## 게시글(read.html) 페이지(댓글 수정 & 삭제) 구현
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
                            str += '    <button id="modify" class="badge btn-warning" onclick="modifyForm('+comment.commentNum+',\''+comment.text+'\')">수정</button>';
                            str += '    <button class="badge btn-danger" onclick=commentRemove('+comment.commentNum+')>삭제</button>';
                            }
                            str += '    </div>';
                            str += '  </li>';
                        });
                        $(".commentList ul.list-group").html(str);
                    });
            }
        
            // 댓글 수정 폼 생성 함수
            function modifyForm(commentNum, text) {
                var str = "";
                str += '  <li id="comment__' + commentNum + '">';
                str += '    <div class="d-flex">';
                str += '    <textarea class="form-control flex-fill modifyText" name="modifyText" id="' +
                        'modifyText">' + text + '</textarea>';
                str += '    <button class="badge btn-warning" onclick=commentModify('+commentNum+')>수정</button>';
                str += '    <button class="badge btn-danger" onclick=getBoardComments()>취소</button>';
                str += '    </div>';

                str += '  </li>';

                $('#comment__' + commentNum).replaceWith(str);
                $('#comment__' + commentNum + ' #modifyText').focus();
            
            }
            // 댓글 수정 함수
            function commentModify(commentNum){
                var memberId = [[${member?.id}]];
                var boardId =[[${boardForm.id}]];
                var text = $('#modifyText').val();
                var data = {commentNum : commentNum, boardId : boardId, text : text, memberId : memberId};
                
                $.ajax({
                    url : '/comments/'+boardId+"/"+commentNum,
                    type : 'PUT',
                    contentType : 'application/json; charset=utf-8',
                    dataType : 'JSON',
                    data : JSON.stringify(data),
                success : function(result){
                    console.log("success : "+result);
                    self.location.reload();
                },
                error : function(err){
                    console.log(err);
                }
                });

            }
            // 댓글 삭제 함수
            function commentRemove(commentNum) {
                var data = {commentNum: commentNum};
                var boardId = [[${boardForm.id}]];
                if (confirm("댓글을 삭제하시겠습니까?")) {
                    $.ajax({
                        url: '/comments/' + boardId + "/" + commentNum,
                        type: 'DELETE',
                        contentType: 'application/json; charset=utf-8',
                        dataType: 'json',
                        data: JSON.stringify(data),
                        success: function (result) {
                            console.log("success : " + result);
                            self.location.reload();
                        },
                        error: function (error) {
                            console.log("error : " + error);
                        }
                    });
                }
            }
            getBoardComments();
        </script>
```
여기서 댓글에 있는 수정 버튼을 클릭 시 발생하는 메서드인 `modifyForm()`에는 수정할 댓글이 수정할 수 있게 입력창으로 변경될 것입니다.  
`replaceWith()`가 그 역할을 담당합니다.  
그리고 변경된 폼에는 `수정`버튼이 생성되어 클릭 시 `commentModify()` 메서드가 호출되어 `$.ajax`로 아까 `CommentController`에서 구현한  
주소로 전송합니다.  

삭제 버튼 클릭 시 발생하는 메서드인 `commentRemove()` 메서드 또한 `$.ajax`로 전송합니다.  

그러면 정상적으로 작동하는지 확인해 보겠습니다.  

## 결과

### 댓글 수정
수정할 댓글의 수정 버튼 확인  
![comment-modify-1](https://user-images.githubusercontent.com/60730405/161432056-cc6c96d6-d3c8-4793-9d14-7521e41047ad.png)  

수정 버튼 클릭 시 나타나는 입력창 확인  
![comment-modify-2](https://user-images.githubusercontent.com/60730405/161432060-f558ecd3-f219-4c75-8456-5073a9ad3a60.JPG)  

수정할 댓글 내용 작성  
![comment-modify-3](https://user-images.githubusercontent.com/60730405/161432062-25c8e34b-ca54-4980-bf37-3910e6e3cb47.JPG)  

수정 버튼 클릭 후 수정된 댓글 확인  
![comment-modify-4](https://user-images.githubusercontent.com/60730405/161432064-433a0022-23df-4555-a17f-6ba90b56ba3e.JPG)

### 댓글 삭제
삭제할 댓글의 삭제 버튼 확인  
![comment-remove-1](https://user-images.githubusercontent.com/60730405/161432276-1c1df2dd-f42f-4c70-a4c8-8b37bc586177.png)  

삭제 버튼 클릭 시 나타나는 알림창 확인
![comment-remove-2](https://user-images.githubusercontent.com/60730405/161432278-220adf0d-4ebb-461c-91ac-ca535c84d18a.png)  

댓글 목록에는 삭제되어 없는 것을 확인  
![comment-remove-3](https://user-images.githubusercontent.com/60730405/161432280-8b0b8ba9-33de-4698-ac5f-18a9aa1d0424.png)  

Database의 `Comment` 테이블에도 삭제되어 없는 것을 확인  
![comment-remove-4](https://user-images.githubusercontent.com/60730405/161432281-04c18446-fec8-47f3-8db4-1a4236120686.png)


***
지금까지 댓글 수정 & 삭제 기능을 구현했습니다.  
다음에는 댓글의 개수를 게시판의 게시글 제목 옆과 게시글 조회 페이지에서 댓글 부분에도 나타나도록 구현해 보겠습니다.  


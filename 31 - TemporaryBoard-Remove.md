![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이전에는 임시등록 게시글의 목록과 개수, 그리고 내용을 불러오는 것까지 구현했습니다.  
이번에는 임시등록 게시글을 목록에서 삭제하는 기능을 구현해 보겠습니다.  

## TemporaryService 구현
삭제는 비교적 간단하게 바로 `Service`에서 `JpaRepository`를 이용하여 삭제하도록 합니다.  
```java
  package com.moon.aza.service;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.entity.Member;
import com.moon.aza.entity.TemporaryBoard;
import com.moon.aza.repository.TemporaryBoardRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@RequiredArgsConstructor
@Service
public class TemporaryBoardService {
    private final TemporaryBoardRepository temporaryBoardRepository;

  ...생략

    @Transactional
    public void remove(Long id){
        temporaryBoardRepository.deleteById(id);
    }

}

```
이미 구현된 `deleteById()`를 이용하여 임시등록 게시글의 `ID 값`으로 삭제하도록 합니다.  

## BoardController 구현
```java
package com.moon.aza.controller;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.dto.LikesDTO;
import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.entity.TemporaryBoard;
import com.moon.aza.service.BoardService;
import com.moon.aza.service.LikesService;
import com.moon.aza.service.TemporaryBoardService;
import com.moon.aza.support.CurrentMember;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/board")
@Controller
public class BoardController {
    private final BoardService boardService;
    private final LikesService likesService;
    private final TemporaryBoardService temporaryBoardService;



    @DeleteMapping("/temporary-remove/{id}")
    public @ResponseBody ResponseEntity<Long> temporaryRemove(@PathVariable("id") Long id,
                                                              @CurrentMember Member member){
        temporaryBoardService.remove(id);
        Long count = temporaryBoardService.getCount(member.getId());
        return new ResponseEntity<>(count, HttpStatus.OK);
    }

    @PostMapping("/register")
    public String register(@ModelAttribute BoardForm boardForm, Long tid){
        Board board = boardService.register(boardForm);

        if(tid!=null){
            temporaryBoardService.remove(tid);
        }

        return "redirect:/board";
    }
    
    ...생략
    
}

```
`/board/temporary-remove/{id}` 경로로 요청이 오면 전달된 `ID 값`을 통해 임시등록 게시글을 삭제하도록 합니다.  
성공적으로 삭제 후, 각 사용자의 임시등록 게시글의 개수를 불러와서 `ResponseEntity`로 응답 데이터를 포함하여 전달합니다.  
왜냐하면 삭제 후, 임시등록 게시글의 개수를 반영하기 위함입니다.  

그리고 임시등록한 게시글을 완전히 작성한 후에 게시판에 게시글을 등록(/board/register)하게 되면, 그 임시등록 게시글을 삭제하도록 합니다.  
그렇기 때문에 전에 임시등록 게시글의 `ID 값`인 `tid`를 `input` 태그의 `hidden` 속성으로 숨겨놨습니다.  
그렇게 되면 임시등록 게시글을 `등록하기` 버튼을 눌러 `tid` 값이 전달되어 조회 후 삭제하게 됩니다.  


## 게시글 등록(register.html) 수정
모달창의 임시등록 게시글 목록에서 `X` 버튼을 클릭 시 함수를 호출하도록 `th:attr="onclick=|tempRemove('${temp.id}',this)|"`  
속성을 추가합니다. `ID 값`을 통해 삭제하도록 요청하고 `this`를 전달하여 목록의 특정 `<tr>` 태그를 없애주어 삭제됐다는 표시를 합니다.   
```html
<!DOCTYPE html>
<html lang="en"xmlns:th="http://www.thymeleaf.org"
    xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
    layout:decorate="~{/layout/basic}">
    <head>
        <link rel="stylesheet" th:href="@{/css/register.css}">
        <script th:src="@{/js/ckeditor/ckeditor.js}"></script>

    </head>
    <th:block layout:fragment="content">
        <div class="board__register">
            <div class="register__top">
                <h1>게시글 작성</h1>
            </div>
            <div th:if="${success}" class="alert alert-info alert-dismissible fade show mt-3" role="alert">
                <span th:text="${success}">임시저장 완료</span>
                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
            </div>
            <form name="form" th:action="@{/board/register}" method="post" th:object="${boardForm}">
                <input type="hidden" id="id" name="id" th:value="${member?.id}"/>
                <input type="hidden" id="tid" name="tid" th:value="${tid}"/>
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
                <div class="form-group" style="margin-bottom: 5%;">
                    <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}"> 홈 화면으로</button>
                    <div class="btn-group">
                        <button type="button" class="btn btn-light btn-temp">임시등록</button>
                        <button type="button" data-bs-toggle="modal" data-bs-target="#temporary" class="btn btn-light btn-count" style="color: red;">[[${count}]]</button>
                    </div>
                    <button type="submit" class="btn btn-primary"><img class="button__icon" th:src="@{/imgs/register-white.png}">등록하기</button>
                </div>    
            </form>
        </div>
        <!-- Modal -->
        <div class="modal fade modal-dialog-scrollable" id="temporary" tabindex="-1" role="dialog" aria-labelledby="temporaryLabel">
            <div class="modal-dialog" role="document">
            <div class="modal-content">
                <div class="modal-header">
                <h5 class="modal-title" id="temporaryLabel" style="font-weight: bold;" >임시등록</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
                </div>
                <div class="modal-body">
                    <table class="table table-hover">
                        <tbody>
                            <tr th:each="temp : ${result}" style="font-size: 0.9rem;">
                                <th scope="row"><a th:href="@{/board/register(id=${temp.id})}" onclick='return confirm("임시등록된 제목과 내용을 불러오시겠습니까?\n작성 중인 제목과 내용은 제거됩니다.");' 
                                    style="text-decoration:none; color:black">[[${temp.title}]]</a></th>
                                <td>[[${#temporals.format(temp.regDate, 'yyyy/MM/dd')}]]</td>
                                <td><button type="button" class="btn-close temp-delete" th:attr="onclick=|tempRemove('${temp.id}',this)|"></button></td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
            </div>
        </div>
        <!-- Modal -->

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
                $('.btn-temp').click(function(){
                    
                    let form = document.form;
                    form.action = '/board/temporary-register';
                    form.method = 'POST';

                    if(!confirm("임시등록 하시겠습니까?\n작성 중인 제목과 내용은 제거됩니다.")){
                        return false;
                    } else {
                        form.submit();
                    }
                });
               

            });
            <!-- 추가 -->
            function tempRemove(id, obj){
                var data = {id : id};
                var count = [[${count}]];

                var tr = $(obj).parent().parent();
                if(confirm("삭제하시겠습니까? ")){
                    $.ajax({
                    url : '/board/temporary-remove/'+id,
                    type: 'DELETE',
                    contentType: 'application/json; charset=utf-8',
                    data: JSON.stringify(data),
                    success: function (result) {
                        console.log("result : "+result);
                        tr.remove();
                        $('.btn-count').html(result);
                    },
                    error: function (error) {
                        console.log("error : " + error);
                    }

                    });
                }
            }
        </script>  
    </th:block>
    
</html>
```
또한 `tempRemove()` 메서드에는 `Ajax`로 서버에 요청하며, 요청이 성공적으로 이루어져 특정 임시등록 게시글을 삭제하면서 개수를 반영하도록 합니다.  

제대로 동작하는 지 확인해 보겠습니다.  

## 결과
임시등록 게시글 목록에서 `X`버튼을 눌렀을 때 나타나는 알림창  
![7](https://user-images.githubusercontent.com/60730405/168417377-febe3783-f229-49c8-ad45-630d486eed3c.JPG)

임시등록 게시글을 삭제했을 때 반영되는 목록의 개수  
![8](https://user-images.githubusercontent.com/60730405/168417375-0d8ac8d3-ffae-49de-826b-d2776577f626.png)

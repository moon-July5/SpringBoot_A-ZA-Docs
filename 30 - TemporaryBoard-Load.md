![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이전에는 임시등록 게시글을 Database에 저장하는 것까지 구현했습니다.  
이번에는 임시등록 게시글 목록을 모달창에 나타내고 각 게시글을 클릭 시 불러오는 것까지 구현해 보겠습니다.  

## TemporaryRepository 구현
`TemporaryRepository`에는 각 사용자의 임시등록 게시글 목록을 조회하도록하는 `findBMemberId()` 메서드와 각 사용자의  
임시등록 게시글 목록의 개수를 나타내는 `getTemporaryBoardCount()` 메서드를 `@Query` 애너테이션을 통해 구현합니다.  

```java
package com.moon.aza.repository;

import com.moon.aza.entity.TemporaryBoard;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface TemporaryBoardRepository extends JpaRepository<TemporaryBoard, Long> {
    @Query("select t from TemporaryBoard t where t.member.id = :memberId")
    List<TemporaryBoard> findByMemberId(@Param("memberId") Long memberId);

    @Query("select count(t) from TemporaryBoard t where t.member.id = :memberId group by t.member.id")
    Long getTemporaryBoardCount(@Param("memberId") Long memberId);
}
```

## TemporaryBoardService 구현
아까 `TemporaryRepository`에서 구현한 메서드들을 이용하여 각 사용자의 임시등록 게시글의 목록을 조회하는 `getList()`   
메서드와 개수를 나타내는 `getCount()`를 구현합니다.  
또한 임시등록한 게시글의 내용들을 불러오기 위해 `temporaryBoardLoad()`메서드를 구현합니다.   

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

    public BoardForm temporaryBoardLoad(Long id){
        TemporaryBoard tBoard = temporaryBoardRepository.getById(id);
        BoardForm boardForm = BoardForm.builder()
                .id(tBoard.getId())
                .title(tBoard.getTitle())
                .contents(tBoard.getContents())
                .writer(tBoard.getWriter())
                .regDate(tBoard.getRegDate())
                .modDate(tBoard.getModDate())
                .build();
        return boardForm;
    }
    public Long getCount(Long memberId){
        Long result = temporaryBoardRepository.getTemporaryBoardCount(memberId);
        if(result==null) result = 0L;
        return result;
    }

    public List<TemporaryBoard> getList(Long memberId){
        List<TemporaryBoard> result = temporaryBoardRepository.findByMemberId(memberId);
        return result;
    }

}

```
여기서 `getCount()` 메서드는 임시등록한 게시글의 개수가 없으면 `뷰(View)`에 아무것도 반환하지 않기 때문에 `null일 경우`  
**0**을 반환하도록 합니다. 또한 `temporaryBoardLoad()` 메서드에는 `PK 값`으로 임시등록 게시글을 조회합니다.  
그리고 `BoardForm(DTO)` 형태로 반환하도록 합니다.  

## BoardController 구현
`BoardController`에는 `게시글 작성(/board/register)`을 `GET` 방식으로 요청하면 여태 구현한 것들을 전달합니다.    
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

  ...생략

    /* 글 작성 페이지 */
    @GetMapping("/register")
    public void boardRegister(@CurrentMember Member member, Long id, Model model){
        log.info("/board/register");
        model.addAttribute("member",member);
        model.addAttribute(new BoardForm());
        
        /* 추가 */
        model.addAttribute("result", temporaryBoardService.getList(member.getId()));
        model.addAttribute("count", temporaryBoardService.getCount(member.getId()));
        if(id!=null){
            BoardForm boardForm = temporaryBoardService.temporaryBoardLoad(id);
            model.addAttribute(new BoardForm(boardForm.getTitle(), boardForm.getContents()));
            model.addAttribute("tid",id);
        }
    }

}

```
현재 로그인한 사용자의 임시등록한 게시글 목록과 개수를 `Model`에 담아 전달합니다.  

또한 임시등록한 게시글을 클릭하게 되면 그 `ID 값`을 받아 `TemporaryBoardService`의 `temporaryBoardLoad()`메서드를 통해  
임시등록했던 게시글의 내용을 불러옵니다. 그리고 전에 `BoardForm`에서 구현한 생성자를 통해 데이터를 전달합니다.  

여기에는 임시등록 게시글의 `ID 값` 또한 전달했는데, 이는 임시등록한 게시글을 완전히 작성하여 게시판에 등록할 경우,  
임시등록 게시글 목록에서 삭제하기 위함입니다. 나중에 임시등록 게시글 삭제를 구현할 때 구현할 예정입니다.  

## 게시글 등록(register.html) 수정
이 페이지에 `BoardController`에 전달한 데이터들을 `Thymeleaf`를 통해 출력하도록 합니다.  
그리고 `input`태그의 `hidden`속성으로 임시등록 게시글의 `ID 값`을 보이지 않게 저장하도록 합니다.  

모달창의 임시등록 게시글의 제목을 클릭하게 되면 `ID 값`을 통해 임시등록 했던 게시글의 내용을 불러오도록 합니다.  
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
                <input type="hidden" id="tid" name="tid" th:value="${tid}"/> <!-- 추가 -->
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
                        <!-- 추가 -->
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
                            <!-- 추가 -->
                            <tr th:each="temp : ${result}" style="font-size: 0.9rem;">
                                <th scope="row"><a th:href="@{/board/register(id=${temp.id})}" onclick='return confirm("임시등록된 제목과 내용을 불러오시겠습니까?\n작성 중인 제목과 내용은 제거됩니다.");' 
                                    style="text-decoration:none; color:black">[[${temp.title}]]</a></th>
                                <td>[[${#temporals.format(temp.regDate, 'yyyy/MM/dd')}]]</td>
                                <td><button type="button" class="btn-close temp-delete"></button></td>
                            </tr>
                            <!-- 추가 -->
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
        </script>  
    </th:block>
    
</html>
```

마지막으로 제대로 동작하는 지 확인해 보겠습니다.  

## 결과
`임시등록` 버튼 옆에 개수 출력과 숫자 클릭 시 나타나는 모달창  
![5](https://user-images.githubusercontent.com/60730405/168416119-d6664ab5-8c4e-4e7a-9c36-756f6bbb8d3c.png)

모달창에서 각 목록에서 제목을 클릭 시 알림창으로 확인 후 내용을 불러옵니다.    
![6](https://user-images.githubusercontent.com/60730405/168416121-92aa14ec-1a54-4f37-b419-dc48360f47d4.JPG)


***
여기까지 임시등록 게시글의 목록과 개수, 그리고 불러오는 기능까지 구현했습니다.  
다음에는 임시등록 게시글의 목록에서 `X` 버튼을 클릭 시 삭제하는 기능을 구현해 보겠습니다.  

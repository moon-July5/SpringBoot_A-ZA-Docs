![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
저번에는 **임시등록** 기능을 위해 `TemporaryBoard Entity`를 구현했습니다.  
이번에는 **임시등록**을 누르면 Database에 내용을 저장하도록 구현해 보겠습니다.  

## TemporaryBoardRepository 구현
먼저 실제 Database에 접근하기 위해 `Repository`를 구현합니다.  
저장만을 구현하기 때문에 별다른 메서드는 구현할 필요없습니다.  
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

}

```

## TempoarayService 구현
게시글 내용이 담겨있는 `BoardForm(DTO)`을 `TemporaryBoard Entity`형태로 변환 후 아까 구현한 `TemporaryBoardRepository`를 통해  
실제 Database에 `save`하는 메서드인 `register()`를 구현합니다.   
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

    public TemporaryBoard register(BoardForm boardForm){
        TemporaryBoard temporaryBoard = TemporaryBoard.builder()
                .title(boardForm.getTitle())
                .contents(boardForm.getContents())
                .writer(boardForm.getWriter())
                .member(Member.builder().id(boardForm.getId()).build())
                .build();

        return temporaryBoardRepository.save(temporaryBoard);
    }
}

```

## BoardController 구현
기존에 구현한 게시글에 대한 `Controller`인 `BoardController`에 `POST` 방식으로 `/board/temporary-register` 경로로 요청이 오면  
게시글을 임시등록해주는 메서드인 `temporaryRegister()`를 구현합니다.  

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

    @PostMapping("/temporary-register")
    public String temporaryRegister(@ModelAttribute BoardForm boardForm, @CurrentMember Member member ,
                                    RedirectAttributes redirectAttributes){
        TemporaryBoard temporaryBoard = temporaryBoardService.register(boardForm);
        redirectAttributes.addFlashAttribute("success", "성공적으로 저장되었습니다.");

        return "redirect:/board/register";
    }
    ...생략
}

```
게시글의 내용이 담겨있는 `BoardForm`을 받아 `TemporaryBoardService`의 `reigster()` 메서드를 통해 Database에 저장하도록 합니다.  
그리고 `RedirectAttributes`로 성공 메시지를 담아 `/board/register` 경로로 리다이렉트합니다.  

제가 여기서 좀 엉성하다고 느꼈던 것은 그냥 이렇게 리다이렉트를 하게되면 임시등록은 성공하지만 작성하고 있던 내용들이 없어집니다.  
작성하고 있던 내용들이 없어지지 않도록 구현을 시도했지만 코드도 난잡해지고 사소한 문제들이 존재하여 이대로 리다이렉트하는 방향으로  
구현했습니다. 아직 실력이 부족하다는 의미인 것 같습니다.  

## 게시글 등록(register.html) 수정
임시등록이 성공적으로 완료되면 성공메시지를 출력하도록 구현합니다.  
그리고 `Javascript`로 `임시등록` 버튼을 누르게 되면 나타나는 알림창과 게시글 내용을 `submit`하도록 구현합니다.  

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
            <!-- success 메시지 추가 -->
            <div th:if="${success}" class="alert alert-info alert-dismissible fade show mt-3" role="alert">
                <span th:text="${success}">임시저장 완료</span>
                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
            </div>
            <!-- success 메시지 추가 -->
            <form name="form" th:action="@{/board/register}" method="post" th:object="${boardForm}">
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
                <div class="form-group" style="margin-bottom: 5%;">
                    <button type="button" th:onclick="|location.href='@{/}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/home-icon-white.png}"> 홈 화면으로</button>
                    <div class="btn-group">
                        <button type="button" class="btn btn-light btn-temp">임시등록</button>
                        <button type="button" data-bs-toggle="modal" data-bs-target="#temporary" class="btn btn-light btn-count" style="color: red;"></button>
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
                          임시등록 목록
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
그러면 임시등록을 누르면 Database에 저장이 되는지 확인해 보도록 하겠습니다.  

## 결과
게시글 내용을 작성 후 `임시등록` 버튼을 누를 시  
![1](https://user-images.githubusercontent.com/60730405/168413942-0e12e88c-ad5b-4ca2-b37f-8ee638169bcd.png)

게시글 내용이 성공적으로 임시등록이 되면 나타나는 메시지
![3](https://user-images.githubusercontent.com/60730405/168413969-0ce2bda2-90d5-4441-bd9f-c8320ae2f658.JPG)

`MySQL Workbench`에서 `temporary_board` 테이블에 데이터가 저장되었는지 확인 
![4](https://user-images.githubusercontent.com/60730405/168414016-1f9c2085-ba91-4d58-b2e7-4ea8b75a2c8b.JPG)

***
지금까지 임시등록한 게시물을 Database에 저장하는 것까지 구현했습니다.  
다음에는 임시등록한 게시물 목록을 모달창에 나타나도록하고 임시등록한 게시물의 내용과 개수를 불러오는 것까지 구현해보겠습니다.  



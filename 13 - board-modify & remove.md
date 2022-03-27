![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
바로 전에는 게시글 조회 기능을 구현했습니다.  
이번에는 게시글 수정과 삭제 기능 구현해보겠습니다.  
게시글 수정과 삭제 기능은 간단하기 때문에 함께 설명하도록 하겠습니다.  

## 게시글 수정
게시글 수정은 이미 `Board` 도메인을 설계할 때 Entity에 게시글의 제목과 내용을 수정하는 메서드를 구현했습니다.  
구현했던 메서드를 이용하겠습니다.  

### BoardService(수정 기능) 구현
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

    // 게시글 수정
    @Transactional
    public void modify(BoardForm boardForm){
        Optional<Board> board = boardRepository.findById(boardForm.getId());

        if(board.isPresent()){
            Board modifyBoard = board.get();

            modifyBoard.modifyTitle(boardForm.getTitle());
            modifyBoard.modifyContents(boardForm.getContents());

            boardRepository.save(modifyBoard);
        }
    }
    
    ...생략

}

```
`modify()` 메서드에는 `BoardForm`을 입력받아 `BoardRepository`의 `findById`로 특정 게시글을 조회합니다.  
조회한 게시글이 존재하면 `modifyTitle()`와 `modifyContents()` 메서드로 수정된 내용을 얻어와 저장합니다.  

### BoardController(수정 기능) 구현
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
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/board")
@Controller
public class BoardController {
    private final BoardService boardService;

  ...생략

    @PutMapping("/modify/{id}")
    public String modify(@ModelAttribute BoardForm boardForm, PageRequestDTO pageRequestDTO,
                         RedirectAttributes redirectAttributes){
        boardService.modify(boardForm);

        redirectAttributes.addAttribute("page", pageRequestDTO.getPage());
        redirectAttributes.addAttribute("id", boardForm.getId());

        return "redirect:/board/read";
    }

}
```
`BoardController`에서 `@PutMapping`으로 `/board/modify/{id}` 경로로 요청이 올 시, 호출하는 `modify()` 메서드를 구현합니다.  
여기에 전달된 게시글의 내용들을 `BoardForm`으로 받아 `BoardService`의 `modify()` 메서드를 호출하여 넘겨줍니다.  
그리고 `RedirectAttributes`를 이용하여 `page`와 `id`를 `/board/read`로 리다이렉트하면서 값을 전달합니다.  

### 게시글 수정(modify.html) 페이지 구현
```html
<div class="board__modify">
            <div class="modify__top">
                <h1>게시글 수정</h1>
            </div>
            <form th:action="@{'/board/modify/' +${boardForm.id}}" method="Post">
                <input type="hidden" name="_method" value="put"/>
                <input type="hidden" id="id" name="id" th:value="${boardForm.id}"/>
                <div class="form-group" style="font-weight:bold; margin-bottom:15px;">
                    <label>제목</label>
                    <input type="text" class="form-control" name="title" th:value="${boardForm.title}" required>
                </div>
                <div class="form-group" style="font-weight:bold; margin-bottom:15px;">
                    <label>작성자</label>
                    <input type="text" class="form-control" name="writer"  th:value="${member.nickname}" readonly>
                </div>
                <div class="form-group" style="font-weight:bold; margin-bottom:15px;">
                    <label>내용</label>
                    <textarea class="form-control" row="5" name="contents" id="contents" th:utext="${boardForm.contents}" required></textarea>
                </div><br>
                <button type="button" th:onclick="|location.href='@{/board/read(id=${boardForm.id}, page=${pageRequestDTO.page})}'|" class="btn btn-secondary"><img class="button__icon" th:src="@{/imgs/cancle-icon-white.png}">취소하기</button>
                <button type="submit" class="btn btn-primary"><img class="button__icon" th:src="@{/imgs/register-white.png}">수정하기</button>
            </form>
        </div> 
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
                
            });
        </script>  
```
이 `modify.html`에는 수정된 내용들을 `form`태그로 `Post` 메서드와 `/board/modify/{게시글 id}` 경로로 전달하도록 합니다.  
여기서 주의할 점은 `form` 태그 안에 `<input type="hidden" name="_method" value="put"/>`을 작성할 필요가 있습니다.  
HTTP에서 GET과 POST가 아닌 **PUT**과 **DELETE**를 사용하기 위해 hidden 속성으로 HTTP Method를 전달해야 합니다.  

또한 Spring Boot 2.2 이상 버전에서는 자동으로 구성되지 않기 때문에 `application.properties`에 아래와 같이 설정 값을 넣어줘야 합니다.  
```
# put, delete method
spring.mvc.hiddenmethod.filter.enabled = true
```

그러면 수정이 정상적으로 되는지 확인해 보겠습니다.  

### 게시글 수정 확인
수정할 게시글  
![board-read-3](https://user-images.githubusercontent.com/60730405/160268733-5ec4a2ba-78e8-4d56-a316-2cdd3bd0a980.JPG)

수정 버튼 클릭 후 수정할 내용 작성합니다.  
![board-modify-1](https://user-images.githubusercontent.com/60730405/160268749-ba06a05f-c01f-4957-a654-296765ae00d5.JPG)

수정이 정상적으로 완료된 것을 확인합니다.  
![board-modify-2](https://user-images.githubusercontent.com/60730405/160268767-9965c381-94e0-418d-a10f-0ce2e0906c14.JPG)

`MySQL Workbench`에서 게시글의 수정 시간이 변경된 것을 확인합니다.  
![board-modify-3](https://user-images.githubusercontent.com/60730405/160268781-e6e6afb5-5d6a-4fbe-8564-e6fe2b4fd665.png)

## 게시글 삭제
게시글 삭제는 훨씬 더 간단합니다.  
일단 먼저 `BoardService`에 삭제 기능을 구현합니다.  

### BoardService(삭제 기능) 구현
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
    
    @Transactional
    public void remove(Long id){
        boardRepository.deleteById(id);
    }
    
  ...생략
}

```
`remove()` 메서드에는 `BoardRepository`에서 제공해주는 메서드인 `deleteById()`로 특정 게시글의 id 값을 받아와 실제 Database에서 삭제합니다.  
이런 식으로 정말 간단하게 삭제됩니다.  

### BoardController(삭제 기능) 구현
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
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Log4j2
@RequiredArgsConstructor
@RequestMapping("/board")
@Controller
public class BoardController {
    private final BoardService boardService;

  ...생략
  
    @DeleteMapping("/remove/{id}")
    public String remove(@PathVariable("id") Long id){
        boardService.remove(id);
        return "redirect:/board";
    }

}

```
`BoardController`에서 `@DeleteMapping`으로 `/board/remove/{게시글 id}`로 요청이 오면 호출하는 메서드인 `remove()`를 구현합니다.  
이 메서드에는 게시글 id 값을 얻어와 `BoardService`의 `remove()` 메서드를 호출하여 게시글 삭제를 수행합니다.  
그 후 게시판 목록 화면으로 리다이렉트 됩니다.  

### 게시글(board-read.html) 조회 페이지 수정
전에 구현한 게시글(board-read.html) 페이지를 수정합니다.  
```html
 <div class="board__read">
        <form th:action="'/board/remove/' +${boardForm.id}" method="Post">
            <input type="hidden" name="_method" value="delete"/>
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
               <button type="button" class="btn btn-secondary modifyButton" th:onclick="|location.href='@{/board/modify(id=${boardForm.id}, page=${pageRequestDTO.page})}'|">수정</button>
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
                $('.removeButton').click(function(){
                    if(!confirm("정말 삭제하시겠습니까?")){
                        self.location.reload();
                    }
                });
                showButton();
            });
        </script>
```
`board-read.html`에서 수정된 것은 `form`태그가 생겼으며, 수정했을 때와 유사하게 `Post` 메서드와 `/board/remove/{게시글 id}` 경로로 전달하도록 합니다.  
또한 역시 `form` 태그안에 `<input type="hidden" name="_method" value="delete"/>`을 작성하여 **DELETE** 메서드로 전달할 수 있게 합니다.  
그리고 위의 **게시글 수정** 기능을 설명할 때 언급하지 않았지만 수정 버튼 클릭 시, `게시글 수정(board-modify.html)` 페이지로 이동할 수 있게 합니다.  

`javascript`로 삭제 버튼 클릭 시 `정말 삭제하시겠습니까?`라는 알림창을 나타나게 하고 확인을 누를 시 삭제하도록 합니다.  

정상적으로 게시글이 삭제되는지 확인해 보겠습니다.  

### 게시글 삭제 확인  
삭제할 게시글에서 삭제 버튼 클릭 시  
![board-remove-1](https://user-images.githubusercontent.com/60730405/160269211-870a3c1d-be29-410b-ae75-6d78ae7f83a6.JPG)  

삭제 후 게시글 목록 화면으로 리다이렉트하며, 삭제된 것을 확인합니다.  
![board-remove-2](https://user-images.githubusercontent.com/60730405/160269212-3517232a-6a64-4c7d-a47a-450ab61f4f79.JPG)  

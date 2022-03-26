![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
저번에는 게시판 기능을 구현하기 전에 Board 도메인 설계와 위지윅 에디터인 **CKEditor4** 를 글 작성 페이지를 구현하면서  
적용하는 것까지 했습니다.  

이번에는 구현한 글 작성 페이지에서 내용을 입력 후 실제 Database에 저장하는 것까지 해보겠습니다.    

## BoardRepository 구현
먼저 실제 Database에 접근하기 위해 BoardRepository를 구현합니다.  
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface BoardRepository extends JpaRepository<Board, Long> {
}

```

## BoardService 구현
```java
package com.moon.aza.service;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.BoardRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashMap;
import java.util.Map;

@RequiredArgsConstructor
@Transactional
@Service
public class BoardService {
    private final BoardRepository boardRepository;

    // 게시글 작성 로직
    public Board register(BoardForm boardForm){
        Map<String, Object> entityMap = saveBoard(boardForm);
        Board board = (Board) entityMap.get("board");

        return boardRepository.save(board);
    }
    // 게시글 저장
    public Map<String, Object> saveBoard(BoardForm boardForm){
        Map<String, Object> entityMap = new HashMap<>();
        Board board = Board.builder()
                .title(boardForm.getTitle())
                .contents(boardForm.getContents())
                .writer(boardForm.getWriter())
                .member(Member.builder().id(boardForm.getId()).build())
                .build();
        entityMap.put("board", board);
        return entityMap;
    }

}

```
먼저 `saveBoard()`메서드에는 `boardForm`을 매개변수로 입력받아 `Board` Entity 형태로 변환할 필요가 있습니다.  
그리고 `Map` 형태로 `BoardForm`에서 `Board` Entity로 변환하여 **Value** 값은 Board Entity로 **Key** 값은 "board"로 저장하였는데,  
이는 만약 이미지를 업로드를 하게 되면 이미지를 실제 Database에 저장하기 위해 Image Entity 또한 `Map`형태로 저장하려고 했습니다.  
하지만 위지윅 에디터인 **CKEditor**를 이용하기 때문에 딱히 필요가 없었습니다.  

그 후 `Controller`에서 호출할 `register()` 메서드를 작성하며, 이 메서드에는 `saveBoard()` 메서드를 호출하여 Entity 형태로 변환합니다.  
그리고 `BoardRepository`의 `save()` 메서드로 실제 Database에 접근하여 게시글을 저장합니다.  

## BoardController 구현
```java
package com.moon.aza.controller;

import com.moon.aza.dto.BoardForm;
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

    @PostMapping("/register")
    public String register(@ModelAttribute BoardForm boardForm){
        Board board = boardService.register(boardForm);

        return "redirect:/board";
    }


}

```
`@PostMapping`으로 `/board/register`으로 요청이 올 경우 호출하도록 `register()` 메서드를 구현합니다.  
이 메서드에는 아까 구현한 `BoardService`의 `register()` 메서드를 호출하도록 하고 성공적으로 저장이 되면  
게시판 페이지로 리다이렉트하며 글 목록을 확인합니다.  

## 결과
글 작성 페이지에서 내용을 입력 후 등록하기 버튼을 눌러 저장합니다.  
![board-register-1](https://user-images.githubusercontent.com/60730405/160242673-d66e9619-3e33-46bd-8feb-32e494fb9a98.JPG)

Database에 저장되어있는지 확인합니다. (저 같은 경우는 MySQL을 사용했기 때문에 **MySQL Workbench**를 통해 확인.)  
![board-register-2](https://user-images.githubusercontent.com/60730405/160242677-e2182aea-9a4a-4e1e-93ae-81cc5bbc7df4.JPG)
![board-register-3](https://user-images.githubusercontent.com/60730405/160242681-e79ae360-2fbe-484a-b0bb-3e568d4091b4.JPG)

성공적으로 저장된 것을 확인할 수 있습니다.  

***
다음에는 작성한 글을 게시판에서 조회하는 기능을 구현하기 전에 페이징 처리를 먼저 구현해 보겠습니다.  

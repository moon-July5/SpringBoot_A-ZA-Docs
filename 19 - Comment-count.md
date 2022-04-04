![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
여태까지 댓글을 작성하고 목록 출력, 수정, 삭제하는 기능까지 구현했습니다.  
이번에는 각 게시글 댓글의 개수를 게시판(Board)의 목록에서 표시되도록 하고 각 게시글 조회 시, 댓글의 개수도 나타나도록 구현해 보겠습니다.  

일단 먼저, 게시판(Board)이나 각 게시글을 조회 후 나타나는 페이지에서 댓글의 개수를 출력하기 위해  
`BoardForm(DTO)`를 수정해야 합니다.  

## BoardForm(DTO) 수정  
```java
package com.moon.aza.dto;


import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.List;

@Builder
@NoArgsConstructor
@AllArgsConstructor
@Data
public class BoardForm {
  
  ...생략
    // 댓글의 개수
    private int commentCnt;


}

```
`BoardForm`에 댓글 개수를 나타내는 필드인 `commentCnt`를 선언해줍니다.  

그 다음 `BoardRepository`에 두 개의 메서드들을 추가합니다.  

## BoardRepository 수정
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface BoardRepository extends JpaRepository<Board, Long> {
    @Query("select b, count(c) from Board b " +
            "left outer join Comment c on c.board = b " +
            "group by b")
    Page<Object[]> getListPage(Pageable pageable);

    @Query("select b, count(c) from Board b " +
            "left outer join Comment c on c.board = b " +
            "where b.id = :boardId group by b")
    List<Object[]> getBoardWithAll(@Param("boardId") Long boardId);
}
```
`getListPage()` 메서드는 게시판의 목록들을 출력하도록 합니다.  
여기서 `@Query`로 `Board` 객체와 **left outer join**으로 `Comment` 객체로 댓글의 개수를 `Object[]`로 반환하도록 했습니다.  

`getBoardWithAll()` 메서드는 특정 게시글의 정보들과 댓글의 개수를 반환하도록 했습니다.   

그리고 이러한 메서드들을 이용하여 `BoardService`의 **페이징 처리**와 **특정 게시글을 조회하는 로직**을 수정합니다.  

## BoradService 수정
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

    // 게시글 조회
    public BoardForm read(Long id){
        //Optional<Board> result = boardRepository.findById(id);
        List<Object[]> result = boardRepository.getBoardWithAll(id);
        Board board = (Board) result.get(0)[0];
        Long commentCnt = (Long) result.get(0)[1];

        return entityToDto(board, commentCnt);
    }

    // 페이징 처리
    public PageResultDTO<BoardForm, Object[]> getList(PageRequestDTO requestDTO){
        Pageable pageable = requestDTO.getPageable(Sort.by("id").descending());

        //Page<Board> result = boardRepository.findAll(pageable);
        Page<Object[]> result = boardRepository.getListPage(pageable);
        Function<Object[], BoardForm> fn = (arr -> entityToDto(
                (Board) arr[0],
                (Long) arr[1]
        ));

        return new PageResultDTO<>(result, fn);
    }
    // entity -> dto
    public BoardForm entityToDto(Board board, Long commentCnt){
        BoardForm dto = BoardForm.builder()
                .id(board.getId())
                .title(board.getTitle())
                .contents(board.getContents())
                .writer(board.getWriter())
                .regDate(board.getRegDate())
                .modDate(board.getRegDate())
                .build();
        dto.setCommentCnt(commentCnt.intValue());
        return dto;
    }
}

```
일단 `entityToDto()` 메서드의 매개변수에 댓글의 개수를 나타내는 Long 타입의 `commentCnt` 변수를 추가합니다.  
이를 `DTO` 객체에 `setter`로 댓글 개수를 설정합니다.  

그리고 `게시판(Board)`에 들어가게 되면 `Controller`에서 호출할 때 사용할 `getList()` 메서드를 수정합니다.  
처음에 `BoardRepository`의 `findAll`로 `Board` 객체를 한꺼번에 조회했는데, 이전에 구현한 `getListPage()` 메서드를 이용하여 반환하도록 합니다.  

또한, 특정 게시글을 조회하는 기능을 수행하는 `read()` 메서드에는 `findById()`메서드에서 특정 게시글 번호를 전달받아 조회하였는데,  
이전에 구현한 `getBoardWithAll()` 메서드로 조회하여 `List<Object[]>` 형태로 나타냅니다.  
`조회한 결과(result)`에서 각 객체 또는 Long 타입을 불러와 `entityToDto()` 메서드로 `DTO` 객체로 변환합니다.  

여기까지 게시판이나 특정 게시글 댓글의 개수를 반환하도록 하는 기능은 완료된 상태이며,  
각 `Controller`에서 `게시판(/board)`이나 `조회(/read)` 경로에 요청이 올 때 호출되는 함수에는 이미 게시판 기능들을 구현할 때 작성했기 때문에 따로 수정할 필요는 없습니다.  

이제 이를 이용하여 `뷰(View)`를 수정해 보겠습니다.  

## 게시판(board.html) 페이지 수정
```html
  <!--게시판 영역-->
      <section class="content">
        <div class="content__top">
          <h1>게시판</h1>
          <button sec:authorize="isAuthenticated()" class="content__button"
          th:onclick="|location.href='@{/board/register}'|">글쓰기</button>
        </div>
        <div class="content__center">
          <table class="table table-hover table-striped">
            <thead>
              <tr>
                <th scope="col">제목</th>
                <th scope="col">작성자</th>
                <th scope="col">날짜</th>
              </tr>
            </thead>
            <tbody>
              <tr th:each="dto : ${result?.dtoList}">                                                             // 추가
                <th scope="row"><a th:href="@{/board/read(id=${dto.id}, page=${result.page})}">[[${dto.title}]] ([[${dto.commentCnt}]])</a></th>
                <td>[[${dto.writer}]]</td>
                <td>[[${#temporals.format(dto.regDate, 'yyyy/MM/dd')}]]</td>
              </tr>
            </tbody>
          </table>
        </div>
        
      ...생략
        
      </section>
```
각 게시글의 제목을 나타내는 `[[${dto.title}]]` 바로 옆에 `([[${dto.commentCnt}]])`을 추가합니다.  
그렇게 되면 `게시글 제목 (댓글 개수)` 형식으로 출력이 됩니다.  

## 게시글 조회(read.html) 페이지 수정
```html
 <div class="board__read">
   
          ...생략
   
            <div class="commentList">
                <h4>댓글 ([[${boardForm.commentCnt}]])</h4>
                <ul class="list-group"></ul>
            </div>
            <div class="comment">
                <div class="comment__input">
                    <textarea class="form-control text"  name="text" placeholder="로그인을 하셔야 댓글을 작성하실 수 있습니다."></textarea>
                    <button type="button" class="btn btn-dark commentSave">등록</button>
                </div>
            </div>
        </div>
      ...생략
```
`<h4> 댓글` 옆에 게시판(board.html) 페이지같이 `([[${boardForm.commentCnt}]])`를 추가합니다.  
`댓글(댓글 개수)` 형식으로 보여지게 됩니다.  

이제 제대로 댓글 개수가 출력이 되는지 확인해 보겠습니다.  

## 결과
`게시판(board.html)`에서 댓글 개수 출력 확인  
![comment-count-1](https://user-images.githubusercontent.com/60730405/161542028-e5af3785-1c34-47ff-8f1c-df0c3a298da9.JPG)

`게시글 조회(read.html)`에서 댓글 개수 출력 확인  
![comment-count-2](https://user-images.githubusercontent.com/60730405/161542025-dcee40c0-62d2-48f1-bf8a-45e789f89aac.JPG)

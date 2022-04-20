![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에는 추천 기능이 정상적으로 동작하도록 구현해 보도록 하겠습니다.  

일단 추천 버튼을 눌렀을 때 그 정보를 저장하기 위해 `Repository`를 구현을 먼저 하겠습니다.  
그 전에 특정 게시글에 특정 사용자가 추천 버튼을 눌렀는지 확인하는 메서드를 `QueryDsl`을 이용해서 `Repository`를 custom하여 구현해보겠습니다.  

## LikesCustomRepository 구현(Custom Interface)
```java
package com.moon.aza.repository.likes;

import com.moon.aza.entity.Likes;

import java.util.Optional;

public interface LikesCustomRepository {
    Optional<Likes> exist(Long memberId, Long boardId);

}
```
특정 게시글에 특정 사용자가 추천 버튼을 눌렀는지 확인하는 메서드인 `exist` 추상 메서드로 구현합니다.  
여기에 `Member`의 PK값과 `Board`의 PK값을 입력받습니다.  

## LikesCustomRepositoryImpl 구현(Custom Class)
```java
package com.moon.aza.repository.likes;

import com.moon.aza.entity.Likes;
import com.moon.aza.entity.QLikes;
import com.querydsl.jpa.impl.JPAQueryFactory;
import org.springframework.data.jpa.repository.support.QuerydslRepositorySupport;

import javax.persistence.EntityManager;
import java.util.Optional;

public class LikesCustomRepositoryImpl extends QuerydslRepositorySupport
        implements LikesCustomRepository{

    JPAQueryFactory jpaQueryFactory;
    QLikes qLikes = QLikes.likes;

    public LikesCustomRepositoryImpl(EntityManager em) {
       super(Likes.class);
       this.jpaQueryFactory = new JPAQueryFactory(em);
   }

    @Override
    public Optional<Likes> exist(Long memberId, Long boardId) {
        Likes likes = jpaQueryFactory.selectFrom(qLikes)
                .where(qLikes.member.id.eq(memberId),
                qLikes.board.id.eq(boardId))
                .fetchFirst();

        return Optional.ofNullable(likes);
    }

}
```
`LikesCustomRepository` 인터페이스의 `exist()` 메서드를 실제 구현하기 위해 상속받아 처리합니다.  
`Likes`에서 `Member`의 PK값인 id와 `Board`의 PK값인 id가 일치하는 정보가 있는지 탐색하여 하나만 조회하도록 합니다.  
왜 하나만 조회하냐면 어차피 추천 버튼을 특정 게시글에 특정 사용자는 한 번밖에 누를 수 밖에 없기 때문입니다.  

## LikesRepository 구현
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Likes;
import com.moon.aza.repository.likes.LikesCustomRepository;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;



@Repository
public interface LikesRepository extends JpaRepository<Likes, Long>, LikesCustomRepository {
}

```
지금까지 구현한 `LikesCustomRepository`를 상속받습니다.  
이제 이를 이용하여 `Service` 계층에서 처리할 것입니다.  

## LikesService 구현
```java
package com.moon.aza.service;

import com.moon.aza.dto.LikesDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Likes;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.BoardRepository;
import com.moon.aza.repository.LikesRepository;
import com.moon.aza.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Transactional
@RequiredArgsConstructor
@Service
public class LikesService {
    private final LikesRepository likesRepository;


    // 추천버튼 눌렀을 때
    public Boolean pushLike(LikesDTO likesDTO){
        Optional<Likes> result = likesRepository.exist(likesDTO.getMemberId(), likesDTO.getBoardId());

        // 존재하지 않으면
        if(!result.isPresent()){
            likesRepository.save(dtoToEntity(likesDTO));
            return true;
        } else {
          likesRepository.deleteById(result.get().getLikeId());
        }
        return false;
    }
    // 사용자가 추천을 했는지 확인
    public Boolean findLike(Long memberId, Long boardId){
        Optional<Likes> result = likesRepository.exist(memberId, boardId);
        if(!result.isPresent()) return false;
        else return true;
    }

    // DTO -> Entity
    private Likes dtoToEntity(LikesDTO likesDTO){
        Likes likes = Likes.builder()
                .member(Member.builder().id(likesDTO.getMemberId()).build())
                .board(Board.builder().id(likesDTO.getBoardId()).build())
                .build();
        return likes;
    }

}

```
`LikesDTO`를 `Entity`로 변환하기 위해 `dtoToEntity()` 메서드를 구현합니다.  

게시글을 조회할 때 현재 사용가 추천을 했는지 확인하여 화면에 표시하기 위해 `findLike()` 메서드를 구현합니다.  
이 메서드에는 아까 `LikesCustomRepository`에서 구현한 `exist()` 메서드를 이용하여 **특정 게시글에 특정 사용자가 추천했는지 확인**하여 존재하면 true를 존재하지 않으면 false를 반환하도록 합니다.  

특정 게시글에서 사용자가 추천 버튼을 눌렀을 때 호출하는 메서드인 `pushLike()`를 구현합니다.  
이 메서드는 마찬가지로 `exist()` 메서드를 이용하여 **특정 게시글에 특정 사용자가 추천했는지 확인**하여 누른적이 없다면 실제 Database에 정보를 저장하도록 하고 아니면 삭제하도록 구현합니다.  

## BoardController 구현
이제 게시글에서 추천 버튼을 눌렀을 때 동작하도록 구현합니다.  
```java
package com.moon.aza.controller;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.dto.LikesDTO;
import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.service.BoardService;
import com.moon.aza.service.LikesService;
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


    @GetMapping({"/read","/modify"})
    public void boardRead(@CurrentMember Member member, Long id, PageRequestDTO pageRequestDTO, Model model){
        log.info("id : "+id);
        BoardForm boardForm = boardService.read(id);
        model.addAttribute("boardForm", boardForm);
        if(member != null)
            model.addAttribute(member);

        // 추천을 누른 사용자 확인
        Boolean likes = likesService.findLike(member.getId(), id);
        model.addAttribute("likes", likes);
    }
    
    ...생략
    
    @PostMapping("/{boardId}/likes")
    public @ResponseBody ResponseEntity<Boolean> likes(@RequestBody LikesDTO likesDTO, Model model){
        log.info("LikesDTO : "+likesDTO);
        Boolean likes = likesService.pushLike(likesDTO);
        return new ResponseEntity<>(likes, HttpStatus.OK);
    }
}

```
추천 버튼을 누르게 되면 `/board/{게시글ID}/likes`로 `POST` 방식으로 요청합니다.  
그러면 사용자와 게시글번호의 정보를 받아 이를 토대로 여태 구현한 `LikeService`에서 `pushLike`를 호출합니다.  
성공적으로 동작이 되면 `ResponseEntity<>()`로 그 결과값(True나 False)을 데이터에 담아 응답합니다.  

그리고 게시글을 조회 시에 추천을 누른 사용자를 조회합니다.  
자신이 이 게시글에 추천을 눌렀는지 확인하여 `Boolean`값을 `likes`라는 이름으로 데이터를 보냅니다.  
왜 이 동작을 추가하였냐면 자신이 특정 게시글에 추천을 눌렀는지 안눌렀는지 모를 수 있기 때문에 이 결과값을 보내서  
추천 버튼을 눌렀으면 버튼 배경색이 나타나고 아니면 배경색이 없도록 표시할 것입니다.  

여기까지 구현했으면 동작하는데 무리없겠지만 각 게시글마다 추천 개수를 표시하기 위해 `BoardForm`과 `QueryDsl`로 작성했던 `SearchBoardRepository`와 `BoardRepository`를 수정합니다.  

## BoardForm 수정
```java
package com.moon.aza.dto;


import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;


@Builder
@NoArgsConstructor
@AllArgsConstructor
@Data
public class BoardForm {
    private Long id;

    private String title;

    private String writer;

    private String contents;

    private LocalDateTime regDate, modDate;

    private int commentCnt;

    private int likesCnt; // 추가
}

```
추천 개수를 나타내는 필드인 `likesCnt`를 추가합니다.  

## SearchBoardRepositoryImpl 수정
```java
package com.moon.aza.repository.search;

import com.moon.aza.entity.Board;
import com.moon.aza.entity.QBoard;
import com.moon.aza.entity.QComment;
import com.moon.aza.entity.QLikes;
import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.Tuple;
import com.querydsl.core.types.Order;
import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.core.types.dsl.PathBuilder;
import com.querydsl.jpa.JPQLQuery;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.repository.support.QuerydslRepositorySupport;

import java.util.List;
import java.util.stream.Collectors;

public class SearchBoardRepositoryImpl extends QuerydslRepositorySupport
        implements SearchBoardRepository {

    public SearchBoardRepositoryImpl() {super(Board.class);}

    @Override
    public Page<Object[]> searchBoard(BooleanBuilder booleanBuilder, Pageable pageable) {
        QBoard qBoard = QBoard.board;
        QComment qComment = QComment.comment;
        QLikes qLikes = QLikes.likes; // 추가

        JPQLQuery<Board> jpqlQuery = from(qBoard);
        jpqlQuery.leftJoin(qComment).on(qComment.board.eq(qBoard));
        jpqlQuery.leftJoin(qLikes).on(qLikes.board.eq(qBoard)); // 추가
                                                                                    // 추가
        JPQLQuery<Tuple> tuple = jpqlQuery.select(qBoard, qComment.countDistinct(), qLikes.countDistinct());
        tuple.where(booleanBuilder);
        tuple.groupBy(qBoard);

        Sort sort = pageable.getSort();
        sort.stream().forEach(order -> {
            Order direction = order.isAscending()? Order.ASC : Order.DESC;
            String property = order.getProperty();

            PathBuilder orderByExpression = new PathBuilder(Board.class, "board");
            tuple.orderBy(new OrderSpecifier(direction, orderByExpression.get(property)));
        });

        long count = tuple.fetchCount();

        tuple.offset(pageable.getOffset());
        tuple.limit(pageable.getPageSize());

        List<Tuple> result = tuple.fetch();

        Page<Object[]> page = new PageImpl<>(result.stream().map(t -> t.toArray()).collect(Collectors.toList()),
                pageable, count);
        return page;
    }
}

```
**Q도메인**인 `QLikes`를 선언하여 left join을 추가하고 추천 개수를 `qLikes.countDistinct()`로 조회하도록 합니다.  

## BoardRepository 수정
```java
package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import com.moon.aza.repository.search.SearchBoardRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface BoardRepository extends JpaRepository<Board, Long>, SearchBoardRepository {
  ...생략

    @Query("select b, count(c), count(l) from Board b " +
            "left outer join Comment c on c.board = b " +
            "left outer join Likes l on l.board = b " +
            "where b.id = :boardId group by b")
    List<Object[]> getBoardWithAll(@Param("boardId") Long boardId);

}

```
위 `@Query`에서 **left outer join**으로 `Likes`를 join하고 개수를 `count`하도록 수정합니다.  
참고로 `getBoardWithAll()` 메서드는 게시글 조회시 나타나는 정보입니다.  

## BoardService 수정
```java
package com.moon.aza.service;

import com.moon.aza.dto.BoardForm;
import com.moon.aza.dto.PageRequestDTO;
import com.moon.aza.dto.PageResultDTO;
import com.moon.aza.entity.Board;
import com.moon.aza.entity.Member;
import com.moon.aza.entity.QBoard;
import com.moon.aza.repository.BoardRepository;
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
        Long likesCnt = (Long) result.get(0)[2];  //추가

        return entityToDto(board, commentCnt, likesCnt);
    }
 

    // 페이징 처리
    public PageResultDTO<BoardForm, Object[]> getList(PageRequestDTO requestDTO){
        Pageable pageable = requestDTO.getPageable(Sort.by("id").descending());
        BooleanBuilder booleanBuilder = getSearch(requestDTO);

        //Page<Board> result = boardRepository.findAll(pageable);
        //Page<Object[]> result = boardRepository.getListPage(pageable);
        Page<Object[]> result = boardRepository.searchBoard(booleanBuilder, pageable);

        Function<Object[], BoardForm> fn = (arr -> entityToDto(
                (Board) arr[0],
                (Long) arr[1],
                (Long) arr[2] //추가
        ));

        return new PageResultDTO<>(result, fn);
    }
    // entity -> dto
    public BoardForm entityToDto(Board board, Long commentCnt, Long likesCnt){
        BoardForm dto = BoardForm.builder()
                .id(board.getId())
                .title(board.getTitle())
                .contents(board.getContents())
                .writer(board.getWriter())
                .regDate(board.getRegDate())
                .modDate(board.getRegDate())
                .build();
        dto.setCommentCnt(commentCnt.intValue());
        dto.setLikesCnt(likesCnt.intValue()); // 추가
        return dto;
    }
}

```
추천 개수를 출력하기 위해 페이징 처리나 게시글 조회, `Entity`를 `DTO`로 변환하는 메서드에 추천 개수를 나타내는  
`likesCnt`를 추가합니다.  

이제 **뷰(View)** 에도 추천 버튼을 눌렀을 때 Ajax 요청하도록 구현합니다.  

## 게시글 조회(read.html) 페이지 구현
```html
  ...생략
      <div class="board__read">
                <div class="read__button">
                    <button type="button" class="btn btn-secondary modifyButton" th:onclick="|location.href='@{/board/modify(id=${boardForm.id}, page=${pageRequestDTO.page}, 
                    type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}'|">수정</button>
                    <button type="submit" class="btn btn-danger removeButton">삭제</button>
                    <button type="button" class="btn btn-info" th:onclick="|location.href='@{/board(page=${pageRequestDTO.page}, 
                    type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}'|">목록</button>
                    <button type="button" id="likes" class="btn" th:classappend="${likes ? 'btn-danger' : 'btn-outline-danger'}"><img class="img" th:src="@{/imgs/likes-icon.png}" 
                        style="width:1.3rem;height: 1.3rem; padding-right: 3px; padding-bottom: 3px;">추천<span class="cnt"> [[${boardForm.likesCnt}]]</span></button>     
                </div>
            </form>
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
        
        <script type="text/javascript" th:inline="javascript">
            $(document).ready(function(){
                var currentMember = [[${member?.nickname}]];
                var writer = [[${boardForm.writer}]];
                var memberId = [[${member?.id}]];
                var boardId =[[${boardForm.id}]];
                var text = $('textarea[name="text"]');
                var cnt = Number($('.cnt').text());
            
          ...생략
          
                $('#likes').click(function(){
                    var data = {memberId : memberId, boardId : boardId};
                    console.log(data);
                    if(currentMember==null){
                        alert("로그인 하셔야합니다.");
                    } else {
                        $.ajax({
                            url : '/board/'+boardId+'/likes',
                            type : 'POST',
                            contentType : 'application/json; charset=utf-8',
                            data : JSON.stringify(data),
                        success : function(result){
                            console.log("success : " +result);
                            if(!result) {
                                $('#likes').prop("class","btn btn-outline-danger");
                                cnt -= 1;
                                $('.cnt').html('&nbsp;'+cnt);
                            } else {
                                $('#likes').prop("class","btn btn-danger");
                                cnt += 1
                                $('.cnt').html('&nbsp;'+cnt);
                            }
                         
                        },
                        error : function(request, status, error){
                            console.log("code:"+request.status+"\n"+"message:"+request.responseText+"\n"+"error:"+error);
                        }

                        });
                    }
                });
           ...
            });
        </script>
    </th:block>
</html>
```
일단 추천 버튼에 `Thymeleaf` 문법으로 `th:classappend="${likes ? 'btn-danger' : 'btn-outline-danger'}"`로 boolean 형태의 데이터인 `likes`가 **true**면 이미 현재 사용자가 눌렀다는 의미이니 `Bootstrap`의 버튼 디자인을 빨간색으로 채워진 버튼으로 바꾸도록 class 속성을 추가합니다.  
**false**면 현재 사용자는 안눌렀다는 의미이니 버튼 디자인을 겉에만 빨간 테두리로 바꾸도록 class 속성을 추가합니다.  

그리고 `Javascript`의 `JQuery`로 비동기 통신인 `Ajax`를 이용하여 추천버튼을 눌렀을 때, `/board/{게시글번호}/likes`로 요청하도록 합니다.  
성공적이면 현재 사용자가 이미 추천을 누른 상태에서 또 눌렀거나 그게 아닌 경우에 따라 추천 버튼의 디자인을 다르게 바꿉니다.  
그리고 바로 추천 개수를 즉시 반영하도록 바꿉니다.  

지금까지 추천 기능을 구현했습니다.  
이제 추천 기능이 제대로 동작하는지 확인해 보겠습니다.  

## 결과
현재 사용자가 추천 버튼을 안눌렀을 때  
![likes-2](https://user-images.githubusercontent.com/60730405/164247464-17c69e2f-bee2-4255-ad7a-3faf081fb483.JPG)

현재 사용자가 추천 버튼을 눌렀을 때
![likes-3](https://user-images.githubusercontent.com/60730405/164247462-c11a0fa1-40a0-4f3b-baa0-29e11132a74b.JPG)

게시판(board.html) 페이지의 추천 개수 확인
![likes-4](https://user-images.githubusercontent.com/60730405/164247453-a76524b3-60a6-406c-85ad-57315fa35bf4.JPG)

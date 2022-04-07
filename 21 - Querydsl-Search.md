![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이전에는 `querdsl`을 위한 세팅을 마쳤습니다.  
이번에는 세팅한 `querydsl`을 가지고 간단한 게시글 검색 기능을 구현해 보겠습니다.  

일단 `Spring Data JPA`는 두 가지 방식으로 `querydsl`을 지원합니다.  
* **QueryDslPredicateExecutor** : JPA Repository와 Querydsl의 Predicate를 결합해 사용, join, fetch를 사용할 수 없다는점이 있습니다.  
* **QueryDslRepositorySupport** : `Service` 계층에 `querydsl`을 사용하지 않고 사용 가능하며, 이 방식은 custom repository를 이용해 구현합니다.  

저같은 경우는 `QueryDslRepositorySupport`을 사용하여 `repository`를 custom하여 구현했습니다.  
그러면 이 `QueryDslRepositorySupport`로 구현해 보도록 하겠습니다.  

## SearchBoardRepository 구현(Custom Interface)
저는 이 `SearchBoardRepository` 인터페이스를 `repository` 폴더 안에 `search` 폴더를 생성하여 그 안에 구현했습니다.  
```java
package com.moon.aza.repository.search;

import com.querydsl.core.BooleanBuilder;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface SearchBoardRepository {
    Page<Object[]> searchBoard(BooleanBuilder booleanBuilder, Pageable pageable);
}

```
게시글 검색 기능을 위한 `searchBoard()`라는 추상 메서드를 생성합니다.  

## SearchBoardRepositoryImpl 구현(Custom Class)
`SearchBoardRepository` 인터페이스의 `searchcBoard()` 메서드를 실제 구현하기 위해 상속받아 처리하며,  
위에서 설명한 `QueryDslRepositorySupport` 클래스를 상속받습니다. 그리고 `searchcBoard()`를 구현합니다.  

```java
package com.moon.aza.repository.search;

import com.moon.aza.entity.Board;
import com.moon.aza.entity.QBoard;
import com.moon.aza.entity.QComment;
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

        JPQLQuery<Board> jpqlQuery = from(qBoard);
        jpqlQuery.leftJoin(qComment).on(qComment.board.eq(qBoard));

        JPQLQuery<Tuple> tuple = jpqlQuery.select(qBoard, qComment.countDistinct());
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

일단 `JPQL(Java Persistence Query Language)`는 `JPA(Java Persistence API)`의 일부로 정의된 플랫폼에 독립적인   
객체지향 쿼리 언어입니다. JPQL은 관계형 데이터베이스의 엔티티에 대한 쿼리를 만드는데 사용됩니다.  

왜 `JPQL`을 사용했냐면 `JPA`는 Entity 객체를 중심으로 개발하므로 SQL을 사용하지 않습니다. 하지만 검색쿼리를 사용할 때는 SQL을 사용해야 합니다.  
그리고 SQL의 영향을 받아 SQL과 비슷하나, DB 테이블에 직접 접근하는 것이 아닌 `JPA` Entity에 동작한다. 
그래서 `JPQL`의 쿼리에는 테이블이 아닌 Entity에서 사용되는 컬럼의 이름을 사용해야 합니다.  

* **SQL** : 데이터베이스 테이블을 대상으로 쿼리 
* **JPQL** : Entity 객체를 대상으로 쿼리  

위의 코드를 대략적으로 설명하겠습니다.  
```java
        JPQLQuery<Board> jpqlQuery = from(qBoard);
        jpqlQuery.leftJoin(qComment).on(qComment.board.eq(qBoard));

        JPQLQuery<Tuple> tuple = jpqlQuery.select(qBoard, qComment.countDistinct());
        tuple.where(booleanBuilder);
        tuple.groupBy(qBoard);
```
사실 이 코드는 아래와 같이 `BoardRepository`에서 작성한 `getListPage()`메서드와 유사합니다.  
```java
    @Query("select b, count(c) from Board b " +
            "left outer join Comment c on c.board = b " +
            "group by b")
    Page<Object[]> getListPage(Pageable pageable);
```
여기서 `JPQLQuery<Tuple>`는 SELECT로 가져오는 데이터는 하나의 Entity가 아닌, 여러 Entity의 데이터들이 섞인 Object라고 할 수 있는데, 
이는 "Tuple"이라는 객체를 통해 추출할 수 있습니다.  

또한 `tuple.where(booleanBuilder)`에서 `booleanBuilder`는 검색 조건들을 넣어주는 컨테이너 역할을 합니다.  
원하는 검색 조건들은 나중에 `booleanExpression`을 생성하여 `Q도메인` 클래스의 필드들을 변수로 활용하여 얻어온 `keyword` 값이 포함되어 있다는 조건을 생성할 것입니다.  

```java
        Sort sort = pageable.getSort(); (1)
        sort.stream().forEach(order -> { (2)
            Order direction = order.isAscending()? Order.ASC : Order.DESC;(3)
            String property = order.getProperty();(4)

            PathBuilder orderByExpression = new PathBuilder(Board.class, "board");(5)
            tuple.orderBy(new OrderSpecifier(direction, orderByExpression.get(property)));
        });

        long count = tuple.fetchCount();
```
`JPQL`에서는 Sort 객체를 지원하지 않기 때문에 orderBy()의 경우 OrderSpecifier<T> extends Comparable>을 파라미터로 처리해야 합니다.
`tuple.orderBy()`으로 표현할 수 있지만 재사용을 위해 다르게 표현합니다.  
  
(1) Pageable로부터 Sort를 가져옵니다.  

(2) Sort는 단일 컬럼에만 가능한 것이 아니므로 순회하며 처리합니다.  

(3) 방향이 내림차순인지 오름차순인지 확인합니다.  

(4) property는 컬럼을 가져오는 것입니다. (예: order by id desc 이면 id를 가져오는 것)  

(5) PathBuilder를 이용해 Sort객체의 속성 등을 처리합니다.  
 
여기에 count를 얻는 방법은 `fetchCount()`를 사용하면 됩니다.

여기서 제가 원하는 것은 Service 계층으로 넘겨줘야할 데이터 타입인 Page<Object[]> 입니다. 이 객체는 페이징처리 + 검색처리까지 완료된 결과물을 담게 되는 것입니다.  
  
```java
        tuple.offset(pageable.getOffset());
        tuple.limit(pageable.getPageSize());

        List<Tuple> result = tuple.fetch();

        Page<Object[]> page = new PageImpl<>(result.stream().map(t -> t.toArray()).collect(Collectors.toList()),
                pageable, count);
        return page;
```
마지막으로 `offset`과 `limit`만 설정해주고 Page<Object[]>로 변환해주면 검색 처리 + 페이징 기능을 한 번에 처리할 수 있는 메서드를 완성하게 됩니다.  
`Page<T>`는 인터페이스 이므로 이를 구현한 `PageImpl<T>`객체를 반환하도록 합니다.  
PageImpl 생성자에는 Pageable과 long값을 이용하는 생성자가 존재합니다.  


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
}

```
지금까지 구현한 `SearchBoardRepository` 인터페이스를 상속받습니다.  
이제 이를 이용하여 `BoardService` 계층에서 처리할 것입니다.  
그 전에 `PageRequestDTO`를 수정합니다.  

## PageRequestDTO 수정
```java
package com.moon.aza.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

@Builder
@AllArgsConstructor
@Data
public class PageRequestDTO {
    private int page;
    private int size;
    // 추가
    private String type;
    private String keyword;

  ...생략
}

```
`검색 타입(type)`과 `키워드(keyword)`를 추가합니다.  

## BoardService 구현
이제 `BoardService`에서 `querydsl`를 이용해서 검색 처리 메서드인 `getSearch()`를 구현합니다.  
 
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

    // Querydsl 검색 처리
    public BooleanBuilder getSearch(PageRequestDTO requestDTO){
        String type = requestDTO.getType();
        String keyword = requestDTO.getKeyword();

        BooleanBuilder booleanBuilder = new BooleanBuilder();
        QBoard qBoard = QBoard.board;
        BooleanExpression expression = qBoard.id.gt(0L); // 0보다 큰 id값 조건 생성
        booleanBuilder.and(expression);

        // 검색 조건이 없는 경우
        if(type==null || type.trim().length()==0)
            return booleanBuilder;

        // 검색 조건
        BooleanBuilder conditionBuilder = new BooleanBuilder();

        if(type.contains("t")) // 제목
            conditionBuilder.or(qBoard.title.contains(keyword));
        if(type.contains("c")) // 내용
            conditionBuilder.or(qBoard.contents.contains(keyword));
        if(type.contains("w")) // 작성자
            conditionBuilder.or(qBoard.writer.contains(keyword));

        // 모든 조건 통합
        booleanBuilder.and(conditionBuilder);

        return booleanBuilder;
    }
  ...생략

}

```
`getSearch()` 메서드는 `PageRequestDTO`를 parameter로 받아서 검색 조건(type)이 있는 경우에는 `BooleanBuilder` 변수를 생성하여 각 검색 조건을 `or`로 연결해서 처리합니다.  
반면에 검색 조건이 없다면 게시글의 번호를 나타내는 `id`를 0보다 큰 값으로만 생성하도록 합니다.  

```java
    // 페이징 처리
    public PageResultDTO<BoardForm, Object[]> getList(PageRequestDTO requestDTO){
        Pageable pageable = requestDTO.getPageable(Sort.by("id").descending());
        BooleanBuilder booleanBuilder = getSearch(requestDTO);

        //Page<Board> result = boardRepository.findAll(pageable);
        //Page<Object[]> result = boardRepository.getListPage(pageable);
        Page<Object[]> result = boardRepository.searchBoard(booleanBuilder, pageable);

        Function<Object[], BoardForm> fn = (arr -> entityToDto(
                (Board) arr[0],
                (Long) arr[1]
        ));

        return new PageResultDTO<>(result, fn);
    }
```
게시글 목록을 조회할 때 사용하는 `getList()` 메서드에서 기존의 코드를 아까 구현한 `searchBoard` 메서드와 `getSearch` 메서드로 수정합니다.  
이제 뷰(View)를 수정하도록 하겠습니다.  
  
## 게시글 목록(board.html) 페이지 수정  
```html
  <div class="content__search">
          <form class="search__form" th:action="@{/board}" method="GET">
            <input type="hidden" name="page" value="1">
            <div class="search__select">
                <select name="type" class="search__select--type">
                    <option value="tc" th:selected="${pageRequestDTO.type=='tc'}">제목+내용</option>
                    <option value="t"  th:selected="${pageRequestDTO.type=='t'}">제목</option>
                    <option value="c"  th:selected="${pageRequestDTO.type=='c'}">내용</option>
                    <option value="w"  th:selected="${pageRequestDTO.type=='w'}">작성자</option>
                </select>
            </div>
            <input class="search__input--text" name="keyword" th:value="${pageRequestDTO.keyword}" 
            placeholder="게시글 검색">
            <button type="submit" class="search__input--button">검색</button>  
          </form>
        </div>
  ...생략
  <!--페이지-->
        <div class="content__end">
          <ul class="pagination h-100 justify-content-center align-items-center">
              <li class="page-item" th:if="${result?.prev}">
                <a class="page-link" th:href="@{/board(page= ${result.start-1}, type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}" tabindex="-1">Prev</a>
              </li>
              <li th:class=" 'page-item '+ ${result?.page == page? 'active' : ''} " th:each="page : ${result?.pageList}">
                <a class="page-link" th:href="@{/board(page= ${page}, type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}">
                  [[${page}]]
                </a>
              </li>
              <li class="page-item" th:if="${result?.next}">
                <a class="page-link" th:href="@{/board(page= ${result.end+1}, type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}" tabindex="-1">Next</a>
              </li>
          </ul>
        </div>
```
검색할 수 있도록 검색창을 추가하고 `PageRequestDTO`를 이용해서 검색 타입(type)에 맞게 선택될 수 있도록 구성하고 키워드(keyword)는 `<input>` 태그로 처리합니다.  
그리고 `<form>` 태그 안에 `<input>` 태그의 `hidden` 속성으로 검색 시 바로 1 페이지로 이동하도록 page 값이 1로 처리되어 있습니다.  
또한 기존의 페이지 처리에서 page외에 `type`과 `keyword`를 추가합니다.  

마지막으로 게시글 수정(modify.html) 페이지와 게시글 조회(read.html) 페이지에도 `type`과 `keyword`를 추가합니다.  

## 게시글 수정(modify.html) 페이지 수정
```html
  <div class="board__modify">
            <div class="modify__top">
                <h1>게시글 수정</h1>
            </div>
            <form th:action="@{'/board/modify/' +${boardForm.id}}" method="Post">
                <input type="hidden" name="_method" value="put"/>
                <input type="hidden" id="id" name="id" th:value="${boardForm.id}"/>
                <!--페이지 번호 추가-->
                <input type="hidden" name="page" th:value="${pageRequestDTO.page}">
                <input type="hidden" name="type" th:value="${pageRequestDTO.type}">
                <input type="hidden" name="keyword" th:value="${pageRequestDTO.keyword}">
                 <!--페이지 번호 추가-->
              
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
```
## 게시글 조회(read.html) 페이지 수정  
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
                    <!--type과 keyword 추가-->
                    <button type="button" class="btn btn-secondary modifyButton" th:onclick="|location.href='@{/board/modify(id=${boardForm.id}, page=${pageRequestDTO.page}, 
                    type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}'|">수정</button>
                    <button type="submit" class="btn btn-danger removeButton">삭제</button>
                    <button type="button" class="btn btn-info" th:onclick="|location.href='@{/board(page=${pageRequestDTO.page}, 
                    type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}'|">목록</button>
                    <!--type과 keyword 추가-->
                </div>
            </form>
```
이제 제대로 검색 기능이 되는지 확인해 보겠습니다.  

## 결과
**제목+내용**을 `검색 타입(type)`으로 키워드(keyword)는 **테스트**라고 입력한 뒤 검색 결과  
![querydsl-3](https://user-images.githubusercontent.com/60730405/162219108-b4de7248-0d30-4f72-a054-70c2f8682f45.JPG)




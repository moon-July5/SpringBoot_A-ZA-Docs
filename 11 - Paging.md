![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***

이번에는 게시글 조회 기능을 구현하기 전에 **페이징(Paging) 처리**를 구현해 보겠습니다.  
페이징 처리라는 것은 게시글을 계속 작성하다 보면 목록에는 게시글의 갯수가 점점 늘어나게 됩니다.  
게시글 목록에는 이 수많은 데이터들을 담기에는 무리가 있고 로딩 속도도 느려질 수 있다는 점도 있기 때문에 페이징 처리를 구현합니다.  
즉, 전체 데이터의 일부를 보여준다고 말할 수 있습니다.  

## PageRequestDTO(페이지 요청 처리) 구현
목록 화면에서 페이지 처리를 하는 경우가 많이 있기 때문에 `페이지 번호(page)`, `페이지 내 목록의 개수(size)`들이 많이 사용됩니다.  
이러한 parameter들을 DTO로 선언하고 나중에 재사용하는 용도로 사용합니다.  
이 `PageRequestDTO`는 **목록 화면에서 전달되는 목록 관련된 데이터에 대한 DTO입니다.**  
화면에서 전달되는 page 파라미터와 size 파라미터를 수집하는 역할을 합니다.  


```java
package com.moon.aza.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

/* 화면에서 전달되는 page 와 size 파라미터들을 수집하는 역할*/
@Builder
@AllArgsConstructor
@Data
public class PageRequestDTO {
    private int page;
    private int size;

    public PageRequestDTO(){
        this.page = 1;
        this.size = 10;
    }
    public Pageable getPageable(Sort sort){
        return PageRequest.of(page-1, size, sort);
    }
}
```
페이지 처리를 위한 가장 중요한 존재는 **Pageable** 인터페이스가 아닐까라는 생각이 듭니다.  
**Pageable** 인터페이스는 페이지 처리에 필요한 정보를 전달하는 용도의 타입입니다.  
인터페이스이기 때문에 실제 객체를 생성할 때는 **PageRequest** 라는 클래스를 사용하는데, 특이하게 protected로 선언되어 new를 이용할 수 없습니다. 그래서 static한 `of()`를 이용해서 처리합니다.  
`of()`는 몇 가지의 형태가 있는데, 그 중에서 `of(int page, int size, Sort sort)`형태를 이용했습니다.  

이 `PageRequestDTO` 클래스의 또 다른 목적은 윗줄에 설명한 JPA 쪽에서 사용하는 **Pageable**타입의 객체를 생성하는 것입니다.  
JPA를 이용하는 경우 페이지 번호가 0부터 시작한다는 점을 감안해서 `page-1`을 하는 형태로 작성했습니다.  

## PageResultDTO(페이지 결과 처리) 구현 
JPA를 이용하는 `Repository`에서는 페이지 처리 결과를 `Page<Entity>` 타입으로 반환하게 됩니다.  
따라서 `Service` 계층에서 이를 처리하기 위해서 별도의 클래스를 만들어서 처리해야 합니다.  
이 클래스에는 다음과 같은 내용이 있습니다.  

* `Page<Entity>`의 엔티티 객체들을 **DTO** 객체로 변환해서 자료구조로 담습니다.  
* 목록 화면 출력에 필요한 페이지 정보들을 구성합니다.  

```java
package com.moon.aza.dto;

import lombok.Data;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import java.util.List;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

@Data
public class PageResultDTO<DTO, EN> {
    private List<DTO> dtoList;

    // 총 페이지 번호
    private int totalPage;

    // 현재 페이지 번호
    private int page;

    // 목록 사이즈
    private int size;

    // 시작 페이지 번호, 끝 페이지 번호
    private int start, end;

    // 이전, 다음
    private boolean prev, next;

    // 페이지 번호 목록
    private List<Integer> pageList;

    // Function<EN, DTO> -> Entity 객체들을 DTO 객체로 변환
    public PageResultDTO(Page<EN> result, Function<EN, DTO> fn){
        dtoList = result.stream().map(fn).collect(Collectors.toList());
        totalPage = result.getTotalPages();
        makePageList(result.getPageable());

    }
    private void makePageList(Pageable pageable){
        this.page = pageable.getPageNumber() + 1; // 0부터 시작하므로 1 추가
        this.size = pageable.getPageSize();

        // 소수점을 올림 처리하여 끝 번호를 계산
        int tempEnd = (int) (Math.ceil(page/10.0)) * 10;

        start = tempEnd - 9;

        prev = start > 1;

        end = totalPage > tempEnd ? tempEnd : totalPage;

        next = totalPage > tempEnd;

        pageList = IntStream.rangeClosed(start, end).boxed().collect(Collectors.toList());
    }
}
```
`PageResultDTO` 클래스는 다양한 곳에서 사용할 수 있도록 Generic 타입을 이용해서 `DTO`와 `EN`이라는 타입을 지정합니다.  
그리고 `Page<Entity>` 타입을 이용해서 생성할 수 있도록 생성자로 작성합니다.  
`makePageList()` 메서드는 화면에 필요한 구성(10개씩 페이지 번호들을 출력, 10페이지 이후에는 이전으로 가능 링크 생성 등)`을 처리합니다.  

## BoardService(페이징 처리) 구현
`BoardService`에서 구현해야 할 것은 Entity 객체를 DTO 객체로 변환하는 `entityToDto()` 메서드를 정의합니다.  
그리고 `PageRequestDTO`를 parameter로, `PageResultDTO`를 return 타입으로 사용하는 `getList()` 메서드를 설계합니다.  

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
import java.util.function.Function;

@RequiredArgsConstructor
@Transactional
@Service
public class BoardService {
    private final BoardRepository boardRepository;

  ...생략
  
    // 페이징 처리
    public PageResultDTO<BoardForm, Board> getList(PageRequestDTO requestDTO){
        Pageable pageable = requestDTO.getPageable(Sort.by("id").descending());

        Page<Board> result = boardRepository.findAll(pageable);

        Function<Board, BoardForm> fn = (entity -> entityToDto(entity));

        return new PageResultDTO<>(result, fn);
    }
    // entity -> dto
    public BoardForm entityToDto(Board board){
        BoardForm dto = BoardForm.builder()
                .id(board.getId())
                .title(board.getTitle())
                .contents(board.getContents())
                .writer(board.getWriter())
                .regDate(board.getRegDate())
                .modDate(board.getRegDate())
                .build();
        return dto;
    }
}
```
`entityToDto()`를 이용해서 `Function`을 생성하고 이를 `PageResultDTO`로 구성했습니다.  
`PageResultDTO`에는 JPA의 처리 결과인 `Page<Entity>`와 `Function`을 전달해서 Entity 객체들을 DTO의 리스트로 변환하고  
화면에 페이지 처리와 필요한 값들을 생성합니다.  

***
여태까지 페이징 처리를 위한 구현이였습니다.  
다음에는 이번에 구현한 페이징 처리를 가지고 게시판 목록 화면에서 게시글을 출력하고 그 게시글을 조회하는 기능을 구현해보도록 하겠습니다.  


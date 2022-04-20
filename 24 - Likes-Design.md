![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 기능은 **추천(좋아요)** 기능입니다.  
사용자들이 게시글을 보고 좋다고 생각하면 다른 사용자들에게 보여지도록 추천하는 기능입니다.
이 추천 기능은 각 게시글마다 한 사용자가 한 번의 추천만 허용하며, 추천한 상태에서 다시 추천을 하게되면 추천을 취소하도록 합니다.   
그럼 추천 기능을 간단하게 구현해 보도록 하겠습니다.  
그 전에 일단 **추천(Likes)** 도메인을 설계하겠습니다.  

## Likes Entity 구현
`Likes` Entity를 구현하기에 앞서, 추천 기능에 필요한 항목들을 정리해 보겠습니다.  

* **LikeId** : 추천 번호(PK)를 나타냅니다.  
* **member** : 개체(Entity)간의 관계(Relation)를 표현합니다. 어느 사용자가 추천했는지 나타냅니다.  
* **board** : 개체(Entity)간의 관계(Relation)를 표현합니다. 어느 게시글에 추천했는지 나타냅니다.  

이를 토대로 `Likes` Entity를 구현해 보겠습니다.  

```java
package com.moon.aza.entity;

import lombok.*;

import javax.persistence.*;

@ToString
@Builder
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
@Entity
public class Likes extends BaseEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long likeId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "board_id")
    private Board board;

}

```
`@ManyToOne`를 이용하여 `Member`와 `Board` Entity 관계를 설정한 것이외에는 별 다른것이 없습니다.  
오히려 심플하게 구현했습니다.  

## Board 수정(연관관계 설정)  
```java
package com.moon.aza.entity;


import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import lombok.*;

import javax.persistence.*;
import java.util.ArrayList;
import java.util.List;


@Builder
@ToString
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Board extends BaseEntity {
  ...생략

    @OneToMany(mappedBy = "board", fetch = FetchType.LAZY, cascade = CascadeType.REMOVE)
    private List<Likes> likes;

  ...생략

}

```
`Board` Entity 쪽에도 @OneToMany를 이용하여 서로 양방향 관계를 맺어줍니다.  
여기서 속성으로 `cascade = CascadeType.REMOVE`을 추가하여 게시글이 삭제되면 추천도 같이 삭제되도록 합니다.  

## DTO구현(LikesDTO)
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
public class LikesDTO {
    private Long likeId;
    private Long boardId;
    private Long memberId;
    private LocalDateTime regDate, modDate;
}

```
이렇게 `Likes` Entity와 `Likes` DTO를 구현 완료했으며, 이제 게시글 조회하게되면 보이게 되는 추천버튼을 구현해 보도록 하겠습니다.  

## 게시글(read.html) 페이지 수정
```html
    ...생략
              <div class="read__button">
                    <button type="button" class="btn btn-secondary modifyButton" th:onclick="|location.href='@{/board/modify(id=${boardForm.id}, page=${pageRequestDTO.page}, 
                    type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}'|">수정</button>
                    <button type="submit" class="btn btn-danger removeButton">삭제</button>
                    <button type="button" class="btn btn-info" th:onclick="|location.href='@{/board(page=${pageRequestDTO.page}, 
                    type=${pageRequestDTO.type}, keyword=${pageRequestDTO.keyword})}'|">목록</button>
                    <!--추천 버튼 추가-->
                    <button type="button" id="likes" class="btn btn-outline-danger"><img class="img" th:src="@{/imgs/likes-icon.png}" 
                        style="width:1.3rem;height: 1.3rem; padding-right: 3px; padding-bottom: 3px;">추천</button>
                    <!--추천 버튼 추가-->
                </div>
    ...생략
```
버튼 옆에 추천할 수 있는 추천 버튼을 구현합니다.  

## 결과
**게시글 조회(read.html)** 시 나타나는 추천 버튼  
![likes-1](https://user-images.githubusercontent.com/60730405/164227856-78220912-a96b-4e64-ba47-3369a926f95a.png)  

***
지금까지 **추천(Likes) 도메인 설계**와 게시글 조회(read.html) 페이지에서 추천 버튼을 구현했습니다.  
다음에는 이 추천 기능이 정상적으로 동작하도록 구현하겠습니다.  


![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
전에는 게시글 생성, 조회, 수정, 삭제 그리고 이미지 업로드 기능을 구현하였습니다.  

이번에 구현할 기능은 **댓글 기능**을 구현할 것입니다.  
사실 이 댓글 기능을 구현하면서 댓글에 답글을 달 수 있도록 **대댓글 기능**까지 구현하려고 했습니다.  
하지만 구현하려다 매우 복잡하여 시간만 많이 소비하고 진행이 더셔서 일반적으로 댓글만 달 수 있도록 구현할 것입니다.  
먼저 **댓글(Comment)** 도메인 설계를 해보겠습니다.  

## Comment Entity 구현
`Comment` Entity를 구현하기에 앞서, 먼저 댓글 작성에 필요한 항목들을 정리해 보겠습니다.  

* **commentNum** : 댓글 번호(PK)를 나타냅니다. 이를 이용해 댓글 수정, 삭제 기능 등을 수행합니다.  
* **text** : 댓글 내용을 나타냅니다.  
* **member** : 개체(Entity)간의 관계(Relation)를 표현합니다. 어느 사용자가 작성했는지 나타냅니다.  
* **board** :  개체(Entity)간의 관계(Relation)를 표현합니다. 어느 게시글에 작성했는지 나타냅니다.  

그리고 여기에 댓글 수정을 위한 메서드인 `changText()`를 구현합니다.  
```java
public void changeText(String text){ this.text = text;}
```

이를 토대로 `Comment` Entity를 구현해 보겠습니다.  
```java
package com.moon.aza.entity;

import lombok.*;

import javax.persistence.*;


@ToString
@Builder
@Getter
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Comment extends BaseEntity {
    @Id
    @GeneratedValue
    private Long commentNum;

    @Column(nullable = false)
    private String text;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "board_id")
    private Board board;

    public void changeText(String text){
        this.text = text;
    }
}
```
저번 `Board` Entity를 구현했을 때와 비슷합니다.  
여기에 `@ManyToOne`를 이용하여 `Member`와 `Board` Entity를 관계 설정했습니다.  

`Board` Entity 쪽에도 `Comment`와 연관관계를 설정합니다.  

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
  
    @OneToMany(mappedBy = "board" ,fetch = FetchType.LAZY, cascade = CascadeType.REMOVE)
    private List<Comment> comments;

  ...생략

}
```
`Board` Entity 쪽에도 `@OneToMany`를 이용하여 서로 양방향 관계를 맺어줍니다.  
여기서 속성으로 `cascade = CascadeType.REMOVE`을 추가하여 게시글이 삭제되면 댓글도 같이 삭제되도록 합니다.  

이제 DTO를 구현해보겠습니다.  

## DTO 구현(CommentDTO)
댓글을 작성할 때 필요한 필드들, `commentNum(댓글 번호)`, `boardId(게시글 번호)`, `memberId(사용자 번호)`, `댓글 내용(text)`,  
`nickname(사용자 닉네임)`, `redDate/modDate(댓글 등록/수정시간)`를 필드로 구현했습니다.  

```
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
public class CommentDTO {
    private Long commentNum;
    private Long boardId;
    private Long memberId;
    private String text;
    private String nickname;
    private LocalDateTime regDate, modDate;
}
```
`Comment` Entity와 `Comment` DTO를 구현 완료했으며, 이제 게시글 조회하게되면 보이게 되는 댓글창을 구현해보겠습니다.  

## 게시글(read.html) 페이지 수정
```html
  ...생략
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
            <!--댓글 입력창 추가-->
            <div class="commentList">
                <h4>댓글</h4>
                <ul class="list-group"></ul>
            </div>
            <div class="comment">
                <div class="comment__input">
                    <textarea class="form-control text"  name="text" placeholder="로그인을 하셔야 댓글을 작성하실 수 있습니다."></textarea>
                    <button type="button" class="btn btn-dark commentSave">등록</button>
                </div>
            </div>
            <!--댓글 입력창 추가-->
        </div>
  ...생략
```
게시글 내용 밑에 댓글을 입력할 수 있도록 추가합니다.  
`<ul class="list-group"></ul>` 부분은 나중에 댓글을 작성하여 저장하게되면 댓글 목록이 이 `<ul>` 태그 하위에  `<li`> 태그로 나타나게 될 것입니다.  

그리고 `Javascript`를 이용하여 로그인하지 않은 사용자들은 댓글 입력창을 클릭 시 로그인하도록 유도하는 알림창을 출력하도록 합니다.  
```javasript
 <script type="text/javascript" th:inline="javascript">
            $(document).ready(function(){
                var currentMember = [[${member?.nickname}]];
                var writer = [[${boardForm.writer}]];
                var memberId = [[${member?.id}]];
                var boardId =[[${boardForm.id}]];
                
                ... 생략

                // 로그인 시 댓글 작성 
                $('.text').click(function(e){
                    if(currentMember==null){
                        if(confirm("로그인 하시겠습니까?")){
                            location.href="/login";
                        }
                    }
                });
</script>
```
확인을 누르게되면 `로그인(login.html) 페이지`로 이동하게 됩니다.  

이제 게시글을 조회하여 댓글 입력창을 확인해 보겠습니다.  

## 결과
댓글 입력창 확인
![comment-1](https://user-images.githubusercontent.com/60730405/161412581-79758de1-87f9-423c-af11-09e086a62d5b.JPG)  

로그인하지 않은 사용자가 댓글 입력창 클릭 시 로그인 페이지로 유도하는 알림창 확인
![comment-2](https://user-images.githubusercontent.com/60730405/161412585-9d5a2195-e22e-46ff-8193-e7375d4e9964.JPG)

***
다음에는 작성한 댓글을 Database에 저장하는 것에 대해서 구현해보도록 하겠습니다.  

  

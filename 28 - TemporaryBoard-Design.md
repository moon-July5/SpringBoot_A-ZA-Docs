![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 기능은 **임시등록** 기능입니다.  
게시글을 작성하다 다른 볼일이 생겨 게시판에 게시글을 등록할 수 없을 때, 임시로 저장하여 작성했던 게시글을 불러와서  
이어서 작성하여 게시판에 게시글을 등록할 수 있는 기능입니다. 블로그, 카페, 커뮤니티 등 게시글을 작성할 수 있는 사이트에는  
필수적인 기능이라고 생각합니다. 서비스하고 있는 다른 사이트보다 엄청 엉성하지만 그래도 구현을 시도해봤습니다.  

일단, `게시글 작성(/board/register) 페이지`에서 아래의 이미지와 같이 `임시등록 버튼`을 만들어, 
`임시등록` 버튼을 누르면 알림창으로 임시등록할 것인지 묻고 확인을 누르면 저장되게 되며, `숫자` 버튼을 누르면 모달창이 나와 임시등록한 목록을 확인할 수 있습니다.  
목록에서 제목을 클릭하게 되면, 알림창으로 불러올 것인지 묻고 확인을 누르면 임시저장했던 내용을 불러옵니다.  
![1](https://user-images.githubusercontent.com/60730405/168411302-0d4d026e-aa3c-4456-9fc2-78de78daf631.png)


그러면 게시글을 임시등록할 `TemporaryBoard Entity`를 구현합니다.  

## TemporaryBoard Entity 구현
`TemporaryBoard Entity`를 구현하기에 앞서, 이 기능에 필요한 항목들을 정리해 보겠습니다.  
* **id** : 임시등록 번호(PK)를 나타냅니다.  
* **title** : 임시등록 게시물의 제목을 나타냅니다.  
* **contents** : 임시등록 게시물의 내용을 나타냅니다.  
* **writer** : 임시등록 게시물의 작성자를 나타냅니다.  
* **member** : 개체(Entity)간의 관계(Relation)를 표현합니다. 어느 사용자가 작성했는지 나타냅니다.  

이제 이를 토대로 `Entity`를 구현해 보겠습니다.  
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
public class TemporaryBoard extends BaseEntity{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "temporary_id")
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false)
    private String contents;

    private String writer;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

}
```
사실 `Board Entity`와 거의 흡사하며, `Board Entity`와 다른점은 `comments`와 `likes`의 관계가 없다는 점입니다.  
그냥 등록만 할 것이기 때문에 필요없기 때문입니다.  

## DTO 수정(BoardForm)
DTO는 기존에 게시글을 등록할때 구현한 `BoardForm`을 재사용할 것입니다.  
여기서 생성자를 새로 구현하여 조금 수정하도록 하겠습니다.  
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

    private int likesCnt;
    
    /* 생성자 추가 */
    public BoardForm(String title, String contents){
        this.title = title;
        this.contents = contents;
    }

}

```
이러한 생성자를 추가한 이유는 나중에 임시등록한 게시글을 불러올 때, 이 생성자에 담아서 불러올 것이기 때문에 추가했습니다.  

## 게시글 등록(register.html) 수정
`임시등록` 버튼과 임시등록 버튼 옆에 숫자를 누르게 되면 나오는 모달창을 구현해 보겠습니다.  
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
                    <!-- 임시등록 버튼 -->
                    <div class="btn-group">
                        <button type="button" class="btn btn-light btn-temp">임시등록</button>
                        <button type="button" data-bs-toggle="modal" data-bs-target="#temporary" class="btn btn-light btn-count" style="color: red;"></button>
                    </div>
                    <!-- 임시등록 버튼 -->
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
        </script>  
    </th:block>
</html>
```

***
지금까지 `temporaryBoard`와 `임시등록` 버튼 및 모달창을 구현해 했습니다. 
다음에는 게시물을 임시등록하여 Database에 등록하는 기능을 구현해 보겠습니다.

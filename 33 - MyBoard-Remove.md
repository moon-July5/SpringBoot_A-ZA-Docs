![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 기능은 `내가 쓴 게시글`에서 `CheckBox`를 선택하여 선택된 게시글만 삭제하도록 기능을 구현해 보겠습니다.  

일단 Database에서 게시글과 연관된 댓글이나 추천 또한 같이 연쇄적으로 삭제가 되어야 하기 때문에 각 `Repository`에서  
**Delete**하는 `Query` 문을 작성하도록 합니다.  


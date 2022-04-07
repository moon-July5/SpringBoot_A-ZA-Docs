![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이전까지 댓글 기능을 구현했습니다.  

이번에는 간단한 게시글 검색 기능을 구현할 것인데, `Querydsl`을 이용할 것입니다.  
JPA의 쿼리 메서드 기능과 @Query를 통해서 많은 기능을 구현할 수 있지만 고정적인 형태라는 단점을 가지고 있기 때문에  
복잡한 조합을 이용하는 경우의 수가 많은 상황에서는 동적으로 쿼리를 생성해서 처리할 수 있는 기능이 필요한데, 그것이 `Querydsl`입니다.  
이 `Querydsl`을 이용하면 복잡한 검색조건이나 조인, 서브 쿼리 등의 기능도 구현이 가능합니다.  

하지만 작성된 Entity 클래스를 그대로 이용하는 것이 아닌 `Q도메인`이라는 것을 이용해야만 합니다.  
이를 작성하기 위해서는 `Querydsl 라이브러리`를 이용해서 Entity 클래스를 `Q도메인` 클래스로 변환하는 방식을 이용하기 때문에  
추가적인 설정이 필요합니다.  

이번에는 이 `Querydsl`을 설정하여 `Q도메인` 클래스로 변환해보도록 하겠습니다.  

## buil.gradle 설정
```
buildscript {
	ext {
		queryDslVersion = "5.0.0"
	}
}

plugins {
	id 'org.springframework.boot' version '2.6.3'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
	id 'war'
	id 'com.ewerk.gradle.plugins.querydsl' version '1.0.10' // 추가
}
```
`plugins` 항목에 querydsl 관련 부분을 추가합니다.  
저같은 경우는 `buildscript` 항목을 추가하여 `queryDslVersion = "5.0.0"`으로 버전을 정했습니다.  

```
dependencies {
  ...생략

	implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
	implementation "com.querydsl:querydsl-apt:${queryDslVersion}"

  ...셍략
}
```
`dependencies` 항목에 `querydsl` 라이브러리를 추가합니다.  
위의 `buildscript` 항목에 추가한 `querydsl` 버전을 적어두도록 합니다.  

여기서 `gradle`에서 사용할 추가적인 task를 추가합니다.  
아래와 같은 내용으로 `build.gradle`의 마지막에 추가하도록 합니다.  
```
def querydslDir = "$buildDir/generated/querydsl"

querydsl {
	jpa = true
	querydslSourcesDir = querydslDir
}
sourceSets {
	main.java.srcDir querydslDir
}
compileQuerydsl{
	options.annotationProcessorPath = configurations.querydsl
}
configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
	querydsl.extendsFrom compileClasspath
}

```
위의 내용을 추가하고 `build.gradle` 파일을 갱신합니다.  
성공적으로 갱신이 되면 `compileQuerydsl`이라는 실행 가능한 task가 추가된 것을 확인할 수 있습니다.  
![querydsl-1](https://user-images.githubusercontent.com/60730405/162196935-df37e491-25ee-4194-ba6c-813cf716446e.JPG)

위의 이미지 같은 상태에서 `compileQuerydsl`을 실행해 봅니다.  
실행된 후에는 프로젝트 내 `build` 폴더 안에 다음과 같은 구조가 생성됩니다.  
![querydsl-2](https://user-images.githubusercontent.com/60730405/162197453-367d7471-ae7f-4f10-8ac8-c97b735c9b90.JPG)

생성된 파일들은 모두 'Q'로 시작되고 이후는 Entity 클래스의 이름과 동일하게 만들어집니다.  

## QBoard 클래스 확인
```java
@Generated("com.querydsl.codegen.DefaultEntitySerializer")
public class QBoard extends EntityPathBase<Board> {

    private static final long serialVersionUID = -1171272575L;

    private static final PathInits INITS = PathInits.DIRECT2;

    public static final QBoard board = new QBoard("board");

    public final QBaseEntity _super = new QBaseEntity(this);

    public final ListPath<Comment, QComment> comments = this.<Comment, QComment>createList("comments", Comment.class, QComment.class, PathInits.DIRECT2);

    public final StringPath contents = createString("contents");

    public final NumberPath<Long> id = createNumber("id", Long.class);

    public final QMember member;

    //inherited
    public final DateTimePath<java.time.LocalDateTime> modDate = _super.modDate;

    //inherited
    public final DateTimePath<java.time.LocalDateTime> regDate = _super.regDate;

    public final StringPath title = createString("title");

    public final StringPath writer = createString("writer");
```
생성된 `QBoard` 클래스를 보면 내부적으로 선언된 필드들이 변수 처리된 것을 확인할 수 있습니다.  

***
지금까지 `Querydsl` 설정을 완료했습니다.  
다음에는 간단한 게시글 검색 기능을 구현해 보겠습니다.  



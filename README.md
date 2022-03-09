# A-ZA-Docs
개인 프로젝트 코드 설명

## 개발환경

OS - Window 10  

IDE - IntelliJ IDEA 2021.3.2 (Community Edition)  

JAVA 11  

Spring Boot 2. 6. 4  

Gradle 7.4  

## 의존성
프로젝트의 `build.gradle` 파일
```
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web' (1)
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa' (2)
	runtimeOnly 'mysql:mysql-connector-java' (3) 

	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf' (4)
	implementation 'nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect' (5)

	compileOnly 'org.projectlombok:lombok' (6) 
	annotationProcessor 'org.projectlombok:lombok' (6) 

	developmentOnly 'org.springframework.boot:spring-boot-devtools' (7)

	implementation 'org.springframework.boot:spring-boot-starter-validation' (8)
	implementation 'org.springframework.boot:spring-boot-starter-mail' (9)
	implementation 'org.springframework.boot:spring-boot-starter-security' (10)
	implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity5' (11)

	testImplementation 'org.springframework.boot:spring-boot-starter-test' (12)
	testImplementation 'org.springframework.security:spring-security-test'
}
```
gradle은 **오픈소스 빌드 자동화 툴**입니다.  
`build.gradle`은 프로젝트의 라이브러리 의존성, 플러그인, 라이브러리 저장소 등을 설정할 수 있는 빌드 스크립트 파일입니다.

(1) - Web 관련 개발을 위한 패키지입니다.  
(2) - JPA를 위한 패키지입니다.  
(3) - mysql과 연동하기 위해 추가해야하는 패키지입니다.  
(4) - Front 개발을 위한 HTML 템플릿 엔진입니다.  
(5) - Thymeleaf 레이아웃 적용을 위한 패키지입니다.  
(6) - 개발을 편리하게 도와주는 lombok 패키지입니다.  
(7) - 개발할 때 실시간으로 reload를 해주는 기능을 제공해주는 패키지입니다.    
(8) - 유효성 검사를 위한 Validation를 사용하기 위해 추가하는 패키지입니다.  
(9) - 메일 전송을 위한 패키지입니다.  
(10) - 인증, 인가 등을 편리하게 개발할 수 있게 해주는 패키지입니다.  
(11) - Thymeleaf에서 Security 정보를 사용할 수 있게 해주는 패키지입니다.  
(12) - 테스트 기능을 제공해주는 패키지입니다.  





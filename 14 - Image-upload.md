![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
지금까지 게시글의 생성, 조회, 수정, 삭제 기능까지 구현했습니다.  
이번에는 게시글에서 이미지를 업로드하는 기능을 구현해 보겠습니다.  
게시글을 작성하는데 위지윅 에디터인 **CKEditor**를 적용했기 때문에 이미지를 업로드할 수 있습니다.  

## 게시글 작성(register.html) 페이지 확인
```html
  ...생략
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

            });
        </script>  
```
**CKEditor**를 적용하면서 코드 밑 부분에 위와 같이 `javascript`로 **CKEditor** 설정을 했을 것입니다.  
여기서 `filebrowserUploadUrl: '/image/upload'`는 이미지를 선택하고 `서버로 전송`을 하게 되면 설정한 경로(/image/upload)로 이미지가 전송되게 됩니다.  

설정한 경로(/image/upload)로 전송될 때 수행할 `Controller`를 구현합니다.  
그 전에 `application.properties`에서 upload에 대한 설정을 합니다.  

## application.properties 설정
```
# upload
spring.servlet.multipart.enabled = true
spring.servlet.multipart.location = C:\\upload
spring.servlet.multipart.max-request-size = 50MB
spring.servlet.multipart.max-file-size = 10MB
image.upload.path = C:\\upload
```
* **spring.servlet.multipart.enabled** : 파일 업로드 가능 여부를 선택합니다.  
* **spring.servlet.multipart.location** : 업로드된 파일의 임시 저장 경로입니다.  
* **spring.servlet.multipart.max-request-size** : 한 번에 최대 업로드 가능 용량입니다.  
* **spring.servlet.multipart.max-file-size** : 파일 하나의 최대 크기입니다.  
* **image.upload.path** : 업로드된 파일의 실제 저장 경로입니다. `Controller`에서 이 설정값을 이용합니다.  

## UploadController 구현
```java
package com.moon.aza.controller;

import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.*;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.UUID;

@Log4j2
@RestController
public class UploadController {
    @Value("${image.upload.path}")
    private String uploadPath;

    @PostMapping("/image/upload")
    public void postImage(@RequestParam MultipartFile upload, HttpServletResponse res, HttpServletRequest req){
        log.info("/image/upload");

        OutputStream out = null;
        PrintWriter printWriter = null;

        res.setCharacterEncoding("utf-8");
        res.setContentType("text/html;charset=utf-8");

        try{
            // 파일 이름 파악(전체경로)
            String originalName = upload.getOriginalFilename();
            String fileName = originalName.substring(originalName.lastIndexOf("\\")+1);
            log.info("fileName : "+fileName);

            // UUID
            String uuid = UUID.randomUUID().toString();

            // 파일 이름 중간에 _를 이용하여 구분
            String saveName = uploadPath + File.separator + uuid + "_" + fileName;

            Path savePath = Paths.get(saveName);

            byte[] bytes = upload.getBytes();

            // 이미지 저장
            out = new FileOutputStream(String.valueOf(savePath));
            out.write(bytes);
            out.flush();

            // ckEditor 로 전송
            printWriter = res.getWriter();
            String callback = req.getParameter("CKEditorFuncNum");
            String fileUrl = "/upload/"+ uuid + "_" + fileName;
            log.info("fileUrl : "+fileUrl);

            // 업로드 메시지 출력
            printWriter.println("<script type='text/javascript'>"
                    + "window.parent.CKEDITOR.tools.callFunction("
                    + callback+",'"+ fileUrl+"','이미지를 업로드하였습니다.')"
                    +"</script>");

            printWriter.flush();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                if(out != null) { out.close(); }
                if(printWriter != null) { printWriter.close(); }
            } catch(IOException e) { e.printStackTrace(); }
        }

    }
}

```
**CKEditor**에서 이미지 업로드 시 요청하는 경로(/image/upload)에 `postImage()` 메서드를 호출하도록 합니다.  
대략 `application.properties`에 설정한 파일 저장 경로에 이미지 파일의 이름을 중복되지 않게 `UUID`를 생성하여 변경하여 저장합니다.  
`OutputStream`으로 이미지 파일들을 설정한 저장 공간에 저장한 후 `PrintWriter`로 성공적으로 이미지 업로드 시 화면에 띄어줄 문구를 작성합니다.  
또한 **CKEditor**에 callback과 fileUrl을 함께 보내줍니다.  
여기서 클라이언트에서 이미지를 전송하고 이미지가 화면에 출력될 때 사진을 찾는 경로는 `fileUrl`이 될 것입니다.  

하지만 여기서 `fileUrl`에 `/upload`에 접근을 못한다는 에러가 발생할 것입니다.  
그래서 위와 같이 특정 경로에 접근하면 외부 경로에 있는 resource에 접근할 수 있도록 설정합니다.  
먼저 `application.properties`에 추가 설정을 합니다.  

## application.properties 설정
```
resource.handler = /upload/**
resource.location = file:///C:/upload/
```

## AppConfig 구현
```java
package com.moon.aza.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;


@Configuration
public class AppConfig implements WebMvcConfigurer{
    @Value("${resource.handler}")
    private String resourceHandler;

    @Value("${resource.location}")
    private String resourceLocation;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler(resourceHandler)
                .addResourceLocations(resourceLocation);
    }

  ...생략
}

```
`WebMvcConfigurer`를 상속받고 `addReourceHandlers()` 메서드를 재정의합니다.  
`application.properties`에 설정한 값들을 불러와 `resourceHandler(/upload/**)`로 시작하는 경로 요청일 경우,  
`resourceLocation(C:/upload/)로 요청을 전달하도록 설정합니다.  

여기서 `SecurityConfig`에서 추가적인 설정이 필요합니다.  
**CKEditor4**에서 최신 버전을 적용했는데 이미지 업로드 시 에러가 발생하여 설정이 필요합니다.  

## SecurityConfig 추가 설정
```java
package com.moon.aza.config;

import com.moon.aza.service.MemberService;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.rememberme.JdbcTokenRepositoryImpl;
import org.springframework.security.web.authentication.rememberme.PersistentTokenRepository;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

import javax.sql.DataSource;

@Log4j2
@RequiredArgsConstructor
@EnableWebSecurity // spring security 설정 클래스 선언
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    ...생략

    // http 요청에 대한 웹 기반 보안
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        
        ...생략
        
        // X-Frame-Options Click jacking 공격 막기 설정
        http.headers()
                .frameOptions().sameOrigin();
        // 'ckeditor4' 이미지 업로드 시 403 Forbidden Error로 인해 설정
        http.cors().and()
                .csrf().disable();
    }
}

```
다른 분들이 **CKEditor4**를 적용하는 것을 참고했는데, 거기서 저는 에러가 발생하여 구글링한 끝에 위와 같이 설정했습니다.  
`Spring Security`를 적용하게 되면 기본적으로 X-Frame-Options Click jacking 공격 막기 설정이 되어있기 때문에 설정합니다.  

이제 게시글에서 이미지가 정상적으로 업로드되는지 확인해 보겠습니다.  

## Image Upload 확인
이미지가 성공적으로 업로드 시 나타나는 문구는 아래의 이미지와 같습니다.  
![image-upload-1](https://user-images.githubusercontent.com/60730405/160277783-35182998-268e-4076-ab62-2df86179dc05.JPG)  

이미지를 찾는 경로는 URL에 나타난 경로와 같습니다.  
![image-upload-2](https://user-images.githubusercontent.com/60730405/160277786-30400ba6-43cc-4653-8c16-c2b33b89f96a.png)  

이미지가 화면에 출력된 것을 확인할 수 있습니다.  
![image-upload-3](https://user-images.githubusercontent.com/60730405/160277788-dec7dffb-da99-4bc1-87d2-16049a3f026c.png)  

게시글을 저장하고 조회한 화면을 아래와 같습니다.  
![image-upload-4](https://user-images.githubusercontent.com/60730405/160277789-a1907662-3f54-41cc-a401-654716fc822f.png)  

실제 C:/upload에 이미지가 저장된 것을 확인할 수 있습니다.  
![image-upload-5](https://user-images.githubusercontent.com/60730405/160277791-26befcf8-bb35-45f2-b18b-042a4ae0e587.png)


![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
바로 전에는 Spring Boot에서 AWS S3를 사용하기 위한 설정을 완료했습니다.  
이번에는 CKEditor로 이미지를 S3에 업로드하도록 구현해 보겠습니다.  

도중에 Error가 많이 발생하여 시간을 너무 많이 소비했습니다.  
그래서 이미지를 S3에 업로드하는 것까지만 진행하겠습니다.  

## AwsS3Service 구현 - 1
```java
package com.moon.aza.service;

import com.amazonaws.auth.AWSCredentials;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.CannedAccessControlList;
import com.amazonaws.services.s3.model.PutObjectRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;


import javax.annotation.PostConstruct;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.util.Optional;

@Log4j2
@RequiredArgsConstructor
@Service
public class AwsS3Service {
    private AmazonS3 amazonS3;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    @Value("${cloud.aws.credentials.accessKey}")
    private String accessKey;

    @Value("${cloud.aws.credentials.secretKey}")
    private String secretKey;


    @Value("${cloud.aws.region.static}")
    private String region;

    @PostConstruct
    public void setS3Client() {
        AWSCredentials credentials = new BasicAWSCredentials(this.accessKey, this.secretKey);

        amazonS3 = AmazonS3ClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(credentials))
                .withRegion(this.region)
                .build();
    }

}

```
저번에 `application.properties`에 S3 및 IAM에 대한 설정을 완료했습니다.  
설정 값들을 `@Value`를 통해 불러옵니다.  
`aws-cloud-starter-aws`에서 제공하는 `AmazonS3`를 이용하여 이미지를 업로드하는데, 설정 값들을 `setS3Slient()` 메서드를 통해 등록합니다.  

`@PostContruct`는 WAS가 올라올 때 bean이 생성된 다음 딱 한번만 실행됩니다. 이를 사용하여 딱 한번만 등록하면 되는 Key 값들을 등록하여 사용할 수 있습니다.  

사실 원래는 또 다른 파일을 생성하여 `@Configuration`을 사용하여 따로 S3 설정 값들을 지정해 놓고  
이 `AwsS3Service`에 `AmazonS3`을 사용하도록 했는데 설정 값들이 안받아지는지 계속 `NULL`이 발생하였습니다.  
이러한 에러로 정말 며칠동안 뭐가 문제인지 잘 몰라서 해맸습니다. 그래서 위의 방식대로 했더니 정상적으로 동작이 됐습니다.  
아직 부족하다는 의미인 것 같아 더 공부해야 할 것 같습니다.  

## AwsS3Service 구현 - 2 (X)
```java
 public String upload(MultipartFile multipartFile, String saveName) throws IOException {
        File uploadFile = convert(multipartFile).orElseThrow(() -> new IllegalArgumentException("파일변환실패"));
        log.info("uploadFile : "+uploadFile);

        String uploadImageUrl = putS3(uploadFile, saveName);
        log.info("uploadImageUrl : "+uploadImageUrl);

        removeNewFile(uploadFile);

        return uploadImageUrl;
    }

    private String putS3(File uploadFile, String saveName) {
        amazonS3.putObject(new PutObjectRequest(bucket, saveName, uploadFile)
                        .withCannedAcl(CannedAccessControlList.PublicRead)
        );
        return amazonS3.getUrl(bucket, saveName).toString();
    }

    private void removeNewFile(File targetFile) {
        if(targetFile.delete()) {
            log.info("파일이 삭제되었습니다.");
        }else {
            log.info("파일이 삭제되지 못했습니다.");
        }
    }

    private Optional<File> convert(MultipartFile file) throws  IOException {
        File convertFile = new File(file.getOriginalFilename());
        log.info("convertFile : "+convertFile);
        if (convertFile.createNewFile()) {
            try (FileOutputStream out = new FileOutputStream(convertFile)) {
                out.write(file.getBytes());
            }
            return Optional.of(convertFile);
        }
        return Optional.empty();
    }
```
`upload()` 메서드는 이미지를 업로드하는 역할을 담당합니다.  
`MultipartFile` 형식의 이미지를 `File` 객체로 변환합니다. 왜냐하면 S3에서는 `MultipartFile` 형식이 받아들여지지 않기때문입니다.  
`saveName` 변수는 `UUID_파일명`에 `업로드하는 경로`가 합쳐진 것입니다.  

`convert()` 메서드는 `MultipartFile` 형식을 `File` 형식으로 변환하는 역할을 합니다.  

`pusS3()` 메서드는 실제 S3에 이미지를 업로드하는 역할을 담당합니다.  
이때 `bucket`은 본인의 S3 저장소 이름이고, `saveName`은 파일을 업로드할 경로, `uploadFile`은 업로드할 파일을 나타냅니다.  
`.withCannedAcl(CannedAccessControlList.PublicRead)`은 누구나 파일을 읽을 수 있도록 해줍니다.  
이에 성공적으로 동작하면 리턴 값은 `해당 파일의 주소`를 반환합니다.  

`removeNewFile()` 메서드는 `pustS3()` 메서드를 통해 실제 S3에 이미지를 업로드하게 되는데, 그 전에
`conver()` 메서드가 동작 시 로컬에도 이미지가 업로드가 됩니다. 그렇게 되면 공간 낭비가 발생하기 때문에 이 메서드를 통해  
로컬에 있는 이미지 파일을 삭제시켜주는 역할을 합니다.  

다음은 구현한 이 `AwsS3Service`를 `UploadController`에서 호출하도록 구현합니다.  

**수정**
위와 같이 하게되면 문제점이 하나 있습니다.  

로컬에서 정상적으로 동작하는 것을 확인 후, 당연히 EC2 서버에서도 돌아갈 것이라 생각하고 이미지를 업로드 했지만  

`File에 대한 IOExceiption : permission denied` 에러가 발생합니다.  

왜냐하면 `convertFile.createNewFile()` 코드로 인해 프로젝트 내에 이미지를 업로드 먼저 하고  

AWS S3 버킷 안에 이미지가 업로드되고 공간 낭비를 해소하기 위해 프로젝트 내에 이미지를 삭제하는 절차를 가집니다.  

프로젝트 내에 이미지를 저장하는데 EC2 서버 내에서는 접근이 거부되서 아예 MultipartFile 형식을 File 형식으로 변환할 수 없고  

그렇게 되면 업로드가 되지 않아 발생하는 문제입니다.  

그렇기 때문에 바로 AWS S3으로 전달하여 업로드하도록 해야합니다.  

그러한 것을 해주는 `putObject(new PutObjectRequest())` 메서드를 조금 수정하면 됩니다.  

수정된 코드는 아래와 같습니다.  

## AwsS3Service 구현 - 3 (O)
```java
package com.moon.aza.service;

import com.amazonaws.auth.AWSCredentials;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.CannedAccessControlList;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PutObjectRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.server.ResponseStatusException;


import javax.annotation.PostConstruct;
import java.io.IOException;

@Log4j2
@RequiredArgsConstructor
@Service
public class AwsS3Service {
    private AmazonS3 amazonS3;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    @Value("${cloud.aws.credentials.accessKey}")
    private String accessKey;

    @Value("${cloud.aws.credentials.secretKey}")
    private String secretKey;

    @Value("${cloud.aws.region.static}")
    private String region;

    @PostConstruct
    public void setS3Client() {
        AWSCredentials credentials = new BasicAWSCredentials(this.accessKey, this.secretKey);

        amazonS3 = AmazonS3ClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(credentials))
                .withRegion(this.region)
                .build();
    }

    public String upload(MultipartFile multipartFile, String saveName) throws IOException {

        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType(multipartFile.getContentType());
        metadata.setContentLength(multipartFile.getSize());

        String uploadImageUrl = putS3(multipartFile, saveName, metadata);
        log.info("uploadImageUrl : "+uploadImageUrl);

        return uploadImageUrl;
    }

    private String putS3(MultipartFile uploadFile, String saveName, ObjectMetadata metadata) throws IOException {
        try {
            amazonS3.putObject(new PutObjectRequest(bucket, saveName,
                    uploadFile.getInputStream(), metadata)
                    .withCannedAcl(CannedAccessControlList.PublicRead)
            );
        } catch (IOException e){
            throw new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "파일 업로드 실패");
        }
        return amazonS3.getUrl(bucket, saveName).toString();
    }
}

```
여기서 `convert()` 메서드와 `removeNewFile()` 메서드가 없어도 좋습니다.  

수정할 곳은 S3에 업로드하는 `putS3()` 메서드에 `ObjectMetadata` 객체를 추가합니다.  

그리고 MultipartFile 형식의 파일을 읽어오는 `getInputStream()` 메서드를 사용합니다.  

수정은 여기까지 하면되며, 오히려 코드가 전보다 간결해진 것을 확인할 수 있습니다.  

이 에러로 인해 정말 며칠동안 파악하고 해결하느라 시간을 너무 많이 소비했습니다.  
그래도 얻어간 것이 있어 다행입니다.  


## UploadController 구현
```java
package com.moon.aza.controller;

import com.moon.aza.entity.Member;
import com.moon.aza.service.AwsS3Service;
import com.moon.aza.support.CurrentMember;
import lombok.RequiredArgsConstructor;
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
@RequiredArgsConstructor
@RestController
public class UploadController {
    private final AwsS3Service awsS3Service;

    @PostMapping("/image/upload")
    public void postImage(@RequestParam MultipartFile upload, HttpServletResponse res, HttpServletRequest req,
                          @CurrentMember Member member){
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

            // 저장 경로
            String uploadPath = "upload/" + member.getId().toString()+"/";

            // 파일 이름 중간에 _를 이용하여 구분
            String saveName = uploadPath + uuid + "_" + fileName;

            log.info("saveName : "+saveName);

            // ckEditor 로 전송
            printWriter = res.getWriter();
            String callback = req.getParameter("CKEditorFuncNum");
            String fileUrl = awsS3Service.upload(upload, saveName);
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
이 `UploadController`는 저번에 CKEditor를 통한 이미지 업로드를 구현할 때 이미 구현했습니다.  
거기서 조금만 수정하면 됩니다.  

방금 전에 구현한 `AwsS3Service`를 선언하고 `fileUrl` 변수에 `awsS3Service`의 `upload(MultipartFile 형식, 업로드할 경로+UUID_파일명)`를 호출합니다.  
그러면 S3에 버킷 안에 저장된 주소를 반환할 것입니다.  

저 같은 경우에는 만들어둔 S3 버킷 안에 `upload/사용자 PK번호/` 경로에 이미지가 저장되어 있을 것입니다.  

한번 정상적으로 동작되는지 확인해 보겠습니다.  

## 결과
원하는 이미지를 CKEditor를 통해 업로드 합니다.  
![13](https://user-images.githubusercontent.com/60730405/172039530-a5aa934b-0d44-4298-a5db-b0148ba1a7c9.JPG)
![14](https://user-images.githubusercontent.com/60730405/172039524-17bc25d0-dc81-4611-a62f-c25f018fd9c4.JPG)

정상적으로 등록이 됐는지 게시글에서 확인합니다.  
![16](https://user-images.githubusercontent.com/60730405/172039529-c1cc887b-6857-416c-866b-30b5ca982b6d.JPG)

그리고 AWS S3의 생성한 버킷에도 정상적으로 이미지가 등록됐는지 경로와 함께 확인합니다.  
![15](https://user-images.githubusercontent.com/60730405/172039528-32a700a1-a935-4297-b0bb-aa66bc73681a.png)

이상으로 CKEditor를 이용한 AWS S3에 이미지를 업로드하는 기능을 구현했습니다.  


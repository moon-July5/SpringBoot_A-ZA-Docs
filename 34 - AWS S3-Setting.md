![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
제가 이 GitHub에 따로 정리하지 않았지만 이 **SpringBoot_A-ZA** 프로젝트를 AWS EC2 서버에 올려놨습니다.  
처음에 CKEditor를 통한 이미지 업로드는 로컬에 저장하도록 되어있는데 AWS EC2 서버에 올려놨기 때문에 이미지 업로드와 이미지를 불러올 수 없었습니다.  

그래서 파일 서버의 역할을 할 수 있는 AWS **S3(Simple Storage Service)** 라는 것을 이용하여 이미지를 업로드하도록 할 것입니다.  

이번에는 S3를 Spring Boot에서 사용할 수 있게 설정하는 것까지 하겠습니다.  

일단 **AWS(Amazon Web Service)** 에 회원가입이 되었다는 가정 하에 진행하겠습니다.  

## AWS S3 버킷 생성

1. 먼저 AWS에서 S3의 버킷을 생성해야 합니다.  
![1](https://user-images.githubusercontent.com/60730405/172035751-82b974ca-123c-4135-90cf-ecda0e8624e7.png)

2. 생성할 버킷에 대한 설정을 합니다. 아래와 같이 진행해 주시면 됩니다.  
![2](https://user-images.githubusercontent.com/60730405/172035794-818b11cf-ae6d-4207-ba92-5c73d355b656.JPG)
![3](https://user-images.githubusercontent.com/60730405/172035852-0e53fc4a-1fb0-4e19-8856-6834a0071dd2.png)
이 부분에서 **ACL 비활성화됨(권장)** 을 체크하지 않고 **ACL 활성화됨** 을 체크하는 이유는 CKEditor를 통해 이미지를 업로드 시  
`The bucket does not allow ACLs (Service: Amazon S3; Status Code: 400; Error Code: AccessControlListNotSupported; ~` 라는 오류가 발생합니다. 이 오류를 해결하기 위해 구글링을 해보니 ACL 활성화됨을 체크하는 것으로 해결할 수 있었습니다.    

![4](https://user-images.githubusercontent.com/60730405/172035856-64770d2a-61fa-4948-9d6c-f476d5e888d5.png)  
기본적으로 **퍼블릭 액세스 차단**이 되어있는데, 이것을 해제해줘야 합니다.  

3. 그러면 아래와 같이 버킷이 생성된 것을 확인할 수 있습니다.  
![5](https://user-images.githubusercontent.com/60730405/172035868-9250df90-04c5-4f69-b720-8ace1f9fbcdd.JPG)

이제 이 만들어진 버킷에 버킷 정책을 설정합니다.  

## AWS S3 버킷 정책 설정  

1. 방금 만든 버킷을 클릭하여 들어가 `[권한 -> 버킷 정책 -> 편집]`을 클릭합니다.  
![6](https://user-images.githubusercontent.com/60730405/172036130-5d867ebf-9e86-46ff-a46d-a2415d400117.png)

2. `버킷 ARN`을 복사 해두고 `정책 생성기`를 클릭합니다.  
![7](https://user-images.githubusercontent.com/60730405/172036169-706853c7-49dc-4720-8d31-3d75ea9781ca.png)  

3. `정책 생성기`에 들어가 아래와 같이 설정한 후에, `Add Statement`를 클릭합니다.  
![8](https://user-images.githubusercontent.com/60730405/172036348-9cedac3f-4364-41e6-9e93-ac25bcfefe02.png)  

3. `Add Statement`를 클릭하게 되면 위에 추가한 정보들이 보이게 되며, 아래 `Generate Policy`를 클릭하여, 나온 정책을 복사합니다.  
![9](https://user-images.githubusercontent.com/60730405/172036349-2547aba3-fc16-48bc-94bd-41c84dc73c98.png)  

4. `버킷 정책 편집`에서 복사한 정책을 붙여넣기 하며, `Resource` 끝에 `/*`을 추가합니다.  
![10](https://user-images.githubusercontent.com/60730405/172036385-e78c9d04-40b3-4b22-976b-b4c52404a04c.JPG)  

## AWS IAM 설정  

AWS S3에 접근하는 것을 허용하기 위해 **IAM(Identity and Access Management)** 를 생성해야 합니다.  
IAM은 AWS에서 제공하는 서비스의 접근 방식과 권한을 관리합니다.  

1. AWS 웹 콘솔에서 IAM을 검색하여 이동합니다.  
![2-1](https://user-images.githubusercontent.com/60730405/172036732-9ff33069-3c34-4339-a411-99dca1560bdf.png)

2. IAM 왼쪽 사이드바에서 `[사용자 -> 사용자 추가]`를 차례로 클릭합니다.  
![2-2](https://user-images.githubusercontent.com/60730405/172036733-b95b7f79-a172-4e02-9e23-7034986966fe.png)

3. 사용자 이름과 `액세스 키-프로그래밍 방식 액세스`를 체크한 후에 다음 설정을 위해 이동합니다.  
![11](https://user-images.githubusercontent.com/60730405/172036829-1f6d9cf6-cc02-44f0-8c53-1fae27002b79.JPG) 

4. `기존 정책 직접 연결`을 클릭 후 검색창에 `s3full`을 입력 후에 `AmazonS3FullAccess`를 체크합니다.  
![12](https://user-images.githubusercontent.com/60730405/172036831-6f1ca03f-397b-4d7a-8f26-7bd0df3eae77.JPG)

5. 최종 생성이 완료되면 액세스 키와 비밀 액세스 키가 생성이 된 것을 확인할 수 있습니다.  
이 키를 잊지 않게 `.csv 다운로드`를 클릭하여 안전한 곳에 저장합니다.  

이제 이 키를 Spring Boot에 설정합니다.  

## Spring Boot [application.properteis] 설정
```
# S3 Bucket
cloud.aws.credentials.accessKey= 액세스 키
cloud.aws.credentials.secretKey= 비밀 액세스 키
cloud.aws.stack.auto=false

# AWS S3 Service bucket
cloud.aws.s3.bucket= 버킷 이름
cloud.aws.region.static= 지역 이름(ex. ap-northeast-2)

# AWS S3 Bucket URL
cloud.aws.s3.bucket.url= 버킷 주소(설정 안해줘도 무방함)

# multipart 사이즈 설정
spring.http.multipart.max-file-size=20MB
spring.http.multipart.max-request-size=20MB

```
위의 설정한 정보들은 절대 GitHub에 Push 하시면 안됩니다!!  

다음은 `build.gradle`에 `Spring-Cloud-AWS` 의존성을 추가합니다.  
```
dependencies {

	implementation 'org.springframework.cloud:spring-cloud-starter-aws:2.2.6.RELEASE'

}
```

이제 Spring Boot AWS S3에 이미지를 업로드하기 위한 설정을 완료했습니다.  


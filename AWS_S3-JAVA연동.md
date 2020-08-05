## AWS S3 - Java연동

<br><br>

### [실습한 내용]

- S3에 저장된 객체리스트를 Java SDK를 이용하여 불러온다.(트리 모양)

- REST API형식으로 요청이 들어오면 S3에 접근하여 객체리스트를 가져오는 방식.

- **Clean 프로젝트** 위에 실습을 진행

<br>
<br>
  

### [실습 환경]

- AWS 개인 계정

  =>  AWS Educate 계정은 IAM사용자를 만들었을때 인증키를 생성할 권한이 없음. 따라서 개인 계정으로 진행.

- S3

  - 버킷 / 폴더 / 객체로 이루어진 데이터 저장 공간
  - 특정 버킷의 객체리스트를 받아오는 것이 목표

- IAM
  - JAVA SDK(로컬)에서 S3로 접근하기 위해서는 IAM의 인증된 사용자의 인증키를 이용해야 한다.
  - 인증키는 외부로 노출이 안되도록 주의해야한다.

- Java SDK
  - [Java용 AWS SDK  설명서](https://docs.aws.amazon.com/ko_kr/sdk-for-java/?id=docs_gateway)
  - [API Reference1](https://sdk.amazonaws.com/java/api/latest/)
  - [API Reference2(for ListObjects) ](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/AmazonS3.html#getObject-com.amazonaws.services.s3.model.GetObjectRequest-)
  - [사용 예시](https://docs.aws.amazon.com/ko_kr/sdk-for-java/v2/developer-guide/examples-s3-buckets.html)


<br><br>


### [1] S3에 Bucket 생성

-> javascript연동 할때와 동일.


<br><br>






### [2] IAM 사용자 계정 생성(for 인증키)

##### 1. Educate 계정

(1) 프로그래밍 방식 액세스로 선택

<img src=".\typora-user-images\image-20200804171934010.png" alt="image-20200804171934010" style="zoom:80%;" />

<br>

(2) 기존 정책 직접 연결에서 s3를 검색후 S3FullAccess로 선택

<img src=".\typora-user-images\image-20200804172130282.png" alt="image-20200804172130282" style="zoom:80%;" />

<br>

(3) 선택사항 -> 추가하지 않음

<img src=".\typora-user-images\image-20200804172208282.png" alt="image-20200804172208282" style="zoom:80%;" />


<br>
(4) 최종확인

<img src=".\typora-user-images\image-20200804172247623.png" alt="image-20200804172247623" style="zoom:80%;" />

<br>

(5) 이후 생성하면 사용자는 만들어지나 필요한 인증키를 생성하지 못한다.권한이 없다고 나옴.

<img src=".\typora-user-images\image-20200804172349154.png" alt="image-20200804172349154" style="zoom:80%;" />


=> 따라서 개인계정으로 다시 진행.

<br>

(6) 인증키 생성 완료

<img src=".\typora-user-images\image-20200804173522544.png" alt="image-20200804173522544" style="zoom:80%;" />



<br><br>





### [3] Spring-Boot 에 S3 import

<br>

(1) Gradle 의존성 추가

```java
implementation('software.amazon.awssdk:bom:2.13.66')
implementation group: 'software.amazon.awssdk', name: 's3', version: '2.13.66'
testCompile group: 'software.amazon.awssdk', name: 's3', version: '2.13.66'
```

=> [버전 참고](https://mvnrepository.com/artifact/software.amazon.awssdk/s3)

<br>

(2) Reroad 하여 잘 추가가되는지 확인한다.

<br><br><br>

### [4] Controller 생성

(1) 네비게이션
<img src=".\typora-user-images\image-20200804174650880.png" alt="image-20200804174650880"  />

<br>

(2) 소스 코드

- accessKey와 secretKey는 IAM에서 인증키를 생성
- bucketName : S3에서 생성한 버킷이름
- folderName : 빈칸부터 특정 폴더까지 해당 폴더부터 하위 객체를 불러올수있다. 

```java
@GetMapping("/list")
    public void ListS3(){
        String accessKey="접근키";
        String secretKey="보안키";
        String bucketName="test-anji";
        String folderName="";

        AWSCredentials crd = new BasicAWSCredentials(accessKey, secretKey);
        AmazonS3 s3Client = AmazonS3ClientBuilder
                .standard()
                .withCredentials(new AWSStaticCredentialsProvider(crd))
                .withRegion(Regions.AP_NORTHEAST_2)
                .build();

        ObjectListing objects = s3Client.listObjects(bucketName, folderName);
        do { //1000개 단위로 읽음
             for (S3ObjectSummary objectSummary : objects.getObjectSummaries()) {
                 System.out.println(objectSummary+"\n");
             }
             objects = s3Client.listNextBatchOfObjects(objects); //<--이녀석은 1000개 단위로만 가져옴..
        } while (objects.isTruncated());

    }
```


<br>


(3) 실습

- **folderName="(빈칸)"**
  => 해당 버킷의 모든 객체가 보여진다.
  <img src=".\typora-user-images\image-20200804175123444.png" alt="image-20200804175123444" style="zoom:80%;" />

<br>

- **folderName="폴더이름"**
  => 지정된 폴더부터 하위 객체들이 보여진다.
  => ex) folderName="first-folder"
  <img src=".\typora-user-images\image-20200804175303543.png" alt="image-20200804175303543" style="zoom:80%;" />


<br>


- **folderName="폴더이름/파일이름"**
  => 폴더이름/파일이름 으로 지정시 해당객체 하나만 불러와진다.
  => ex) folderName="first-folder/33.PNG"
  <img src=".\typora-user-images\image-20200804175500500.png" alt="image-20200804175500500" style="zoom:80%;" />

<br>

- **folderName="파일 이름"**

  => 반환 되는 객체 결과 없음
<br>

<br><br>


### [5] REST API 형식으로 재정리
<br>

(1) 소스 코드 

```java
 @GetMapping("/list")
    public ResponseEntity<List<S3ObjectSummary>> ListS3(){
        String accessKey="접근키";
        String secretKey="보안키";
        String bucketName="test-anji";
        String folderName="";

        AWSCredentials crd = new BasicAWSCredentials(accessKey, secretKey);
        AmazonS3 s3Client = AmazonS3ClientBuilder
                .standard()
                .withCredentials(new AWSStaticCredentialsProvider(crd))
                .withRegion(Regions.AP_NORTHEAST_2)
                .build();

        ObjectListing objects = s3Client.listObjects(bucketName, folderName);
 
       return  new ResponseEntity<>(objects.getObjectSummaries(), HttpStatus.CREATED);
    }
```

<br>

(2) Postman으로 호출 결과 확인  => Rest API 통신 완료.

<img src=".\typora-user-images\image-20200805104955460.png" alt="image-20200805104955460" style="zoom:80%;" />





<br><br><br>

​																																															... 2020.08.04 작성 완료



------------------
<br><br>


### [5] 참고 블로그

1. [황군's  story](https://using.tistory.com/87)
2. [진짜 개발자](https://galid1.tistory.com/590)
3. [기억보단 기록을](https://jojoldu.tistory.com/300)

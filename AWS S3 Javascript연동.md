## AWS S3 Javascript연동
<br>

### [실습한 내용]

- s3에 저장된 객체리스트를 화면단에 가져와 보여준다.( 트리 모양 )
- 현재는 프론트 단에서 서버를 거치지않고 내용을 불러오지만 서버를 거쳐서 불러오는것으로 바꿀예정(java용 sdk사용 or REST API)
- javascript용 sdk를 사용함.



<br><br>

### [실습 환경]

- AWS Educate 계정 
  - resion : 미국 동부(버지니아 북부) -> 고정
- S3
  - 버킷 / 폴더 / 객체로 이루어진 데이터 저장 공간
  - 특정 버킷의 객체리스트를 받아오는 것이 목표
- Cognito
  - S3를 외부에서 접근할 때 S3에 접근을 위한 자격 증명 풀 이용.
  - 자격 증명 풀 : AWS 자격증명을 제공하여 기타 AWS 서비스에 대한 사용자 액세스 권한을 부여.

- JavaScript SDK
  - [javascript sdk](https://docs.aws.amazon.com/sdk-for-javascript/index.html)
  - [API Reference](https://docs.aws.amazon.com/ko_kr/AWSJavaScriptSDK/latest/index.html)


<br>


### [1] S3에 Bucket 생성



1. 버킷을 생성한다.

   <img src=".\typora-user-images\image-20200730174655494.png" alt="image-20200730174655494" style="zoom: 80%;" />
<br>
2. 버킷의 권한을 설정하여 외부에서 접근을 허가하게 바꾼다.

<img src=".\typora-user-images\image-20200730174802538.png" alt="image-20200730174802538" style="zoom:80%;" />
<br>
3. CORS 구성 수정

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
       <CORSRule>
           <AllowedOrigin>*</AllowedOrigin>
           <AllowedMethod>POST</AllowedMethod>
           <AllowedMethod>GET</AllowedMethod>
           <AllowedMethod>PUT</AllowedMethod>
           <AllowedMethod>DELETE</AllowedMethod>
           <AllowedMethod>HEAD</AllowedMethod>
           <AllowedHeader>*</AllowedHeader>
           <ExposeHeader>ETag</ExposeHeader>
       </CORSRule>
   </CORSConfiguration>
   ```

   <br>

4. 객체 등록

   - "/"

   <img src=".\typora-user-images\image-20200730175318664.png" alt="image-20200730175318664" style="zoom:80%;" />

   - "/first-folder"<img src=".\typora-user-images\image-20200730175333960.png" alt="image-20200730175333960" style="zoom:80%;" />
     
   - "/second-folder"<img src=".\typora-user-images\image-20200730175350086.png" alt="image-20200730175350086" style="zoom:80%;" />



<br>



### [2] Amazon Cognito 자격증명 풀 생성

1.  자격 증명 풀 생성

  
   ```javascript
   // Amazon Cognito 인증 공급자를 초기화합니다
   AWS.config.region = 'us-east-1'; // 리전
   AWS.config.credentials = new AWS.CognitoIdentityCredentials({
       IdentityPoolId: '개인아이디',
   });
   ```

 <br>  





### [3] JavaScript로 호출

1. index.html

```html
<!DOCTYPE html>
<html>
  <head>
     <!-- **DO THIS**: -->
    <!--   Replace SDK_VERSION_NUMBER with the current SDK version number -->
    <script src="https://sdk.amazonaws.com/js/aws-sdk-2.723.0.min.js"></script>
    <script src="./app.js"></script>
    <script>
       function getHtml(template) {
          return template.join('\n');
       }
       listAlbums();
      // viewAlbum();
      listObjects();
    </script>
  </head>
  <body>
    <h1>My Photo Albums App</h1>
    <div id="app"></div>
  </body>
</html>
```


<br>
2. app.js

```javascript
// 초기 변수 설정 + 라이브러리 호출

var albumBucketName = "first-test-anji";
var bucketRegion = "us-east-1";
var IdentityPoolId = "개인 아이디";


AWS.config.update({
  region: bucketRegion,
  credentials: new AWS.CognitoIdentityCredentials({
    IdentityPoolId: IdentityPoolId
  })
});

var s3 = new AWS.S3({
  apiVersion: "2006-03-01",
  params: { Bucket: albumBucketName }
});


// 객체리스트 호출 메소드
let paramOflist={
    Bucket:albumBucketName,
    MaxKeys:1000
};
function listObjects(){
    s3.listObjectsV2(paramOflist,function(err,data){
        if(err)console.log(err,err.stack);
        else console.log(data);
    });
}



// S3에 등록된 사진파일 View메소드
function listAlbums() {
    s3.listObjects({ Delimiter: "/" }, function(err, data) {
      if (err) {
        return alert("There was an error listing your albums: " + err.message);
      } else {
        var albums = data.CommonPrefixes.map(function(commonPrefix) {
          var prefix = commonPrefix.Prefix;
          var albumName = decodeURIComponent(prefix.replace("/", ""));
          return getHtml([
            "<li>",
            "<span onclick=\"deleteAlbum('" + albumName + "')\">X</span>",
            "<span onclick=\"viewAlbum('" + albumName + "')\">",
            albumName,
            "</span>",
            "</li>"
          ]);
        });
        var message = albums.length
          ? getHtml([
              "<p>Click on an album name to view it.</p>",
              "<p>Click on the X to delete the album.</p>"
            ])
          : "<p>You do not have any albums. Please Create album.";
        var htmlTemplate = [
          "<h2>Albums</h2>",
          message,
          "<ul>",
          getHtml(albums),
          "</ul>",
          "<button onclick=\"createAlbum(prompt('Enter Album Name:'))\">",
          "Create New Album",
          "</button>"
        ];
        document.getElementById("app").innerHTML = getHtml(htmlTemplate);
      }
    });
  }
  function viewAlbum(albumName) {
    var albumPhotosKey = encodeURIComponent(albumName) ;
    s3.listObjects({ Prefix: albumPhotosKey }, function(err, data) {
      if (err) {
        return alert("There was an error viewing your album: " + err.message);
      }
      // 'this' references the AWS.Response instance that represents the response
      var href = this.request.httpRequest.endpoint.href;
      var bucketUrl = href + albumBucketName + "/";
  
      var photos = data.Contents.map(function(photo) {
        var photoKey = photo.Key;
        var photoUrl = bucketUrl + encodeURIComponent(photoKey);
        return getHtml([
          "<span>",
          "<div>",
          '<img style="width:128px;height:128px;" src="' + photoUrl + '"/>',
          "</div>",
          "<div>",
          "<span onclick=\"deletePhoto('" +
            albumName +
            "','" +
            photoKey +
            "')\">",
          "X",
          "</span>",
          "<span>",
          photoKey.replace(albumPhotosKey, ""),
          "</span>",
          "</div>",
          "</span>"
        ]);
      });
      var message = photos.length
        ? "<p>Click on the X to delete the photo</p>"
        : "<p>You do not have any photos in this album. Please add photos.</p>";
      var htmlTemplate = [
        "<h2>",
        "Album: " + albumName,
        "</h2>",
        message,
        "<div>",
        getHtml(photos),
        "</div>",
        '<input id="photoupload" type="file" accept="image/*">',
        '<button id="addphoto" onclick="addPhoto(\'' + albumName + "')\">",
        "Add Photo",
        "</button>",
        '<button onclick="listAlbums()">',
        "Back To Albums",
        "</button>"
      ];
      document.getElementById("app").innerHTML = getHtml(htmlTemplate);
    });
  }



```

<br>

3. 실행 화면
   <img src=".\typora-user-images\image-20200730181033872.png" alt="image-20200730181033872" style="zoom: 67%;" />


<br>
4. console
   <img src=".\typora-user-images\image-20200730181151026.png" alt="image-20200730181151026" style="zoom:80%;" />

   =>  first-test-anji의 버킷의 객체들의 리스트가 정상적으로 출력되는 것을 확인. 
   =>  조금 가공하여 화면단에 출력해야함(트리 형식).

   => JAVA SDK를 사용하여 서버단에서도 가능할지 알아보는 중.
















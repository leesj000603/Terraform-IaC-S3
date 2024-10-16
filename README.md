# Terraform-IaC-training
terraform을 사용한 코드형 인프라 구축 실습


## Terraform을 이용한 S3생성 + html 파일 업로드 & 접속 실습

### terraform 설치
```
$ sudo su -

# wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg


# echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# apt-get update && apt-get install terraform -y

# terraform -version
```

### bocket.tf - 버킷 생성 (하단의 권한 설정이 되지 않으면, S3에 업로드 및 수정, 조회가 제한된다.)
```
# S3 버킷 생성
resource "aws_s3_bucket" "bucket1" {
  bucket = "생성하고자 하는 S3 버킷 이름" 
}

# S3 버킷의 public access block 설정
resource "aws_s3_bucket_public_access_block" "bucket1_public_access_block" {
  bucket = aws_s3_bucket.bucket1.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# S3 버킷의 웹사이트 호스팅 설정
resource "aws_s3_bucket_website_configuration" "xweb_bucket_website" {
  bucket = aws_s3_bucket.bucket1.id  # 생성된 S3 버킷 이름 사용

  index_document {
    suffix = "main.html"
  }
}

# S3 버킷의 public read, object 업로드 정책 설정
resource "aws_s3_bucket_policy" "public_read_access" {
  bucket = aws_s3_bucket.bucket1.id  # 생성된 S3 버킷 이름 사용

  policy = <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": [ "s3:GetObject" ],
        "Resource": [
          "arn:aws:s3:::[버킷명]",
          "arn:aws:s3:::[버킷명]/*"
        ]
      },
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": [ "s3:PutObject" ],
        "Resource": [
          "arn:aws:s3:::[버킷명]"
          "arn:aws:s3:::[버킷명]/*"
        ]
      }
    ]
  }EOF
}
```

### output.tf - aws콘솔창 확인 없이 명령어 만으로 접속 url확인
```
output "website_endpoint" {
  value = aws_s3_bucket.bucket1.website_endpoint
  description = "The endpoint for the S3 bucket website."
}
```

- 이러한 방식으로 접속 url이 생성된다.
  <br>
![image](https://github.com/user-attachments/assets/c2135dcc-46ce-4318-9501-ff2cfafb1730)

### putmain.tf
```
# 이미 존재하는 S3 버킷에 main.html 파일을 업로드
resource "aws_s3_object" "main" {
  bucket        = "버킷명"  # 존재하는 버킷 이름 사용
  key           = "main.html"               # 업로드할 파일 이름
  source        = "main.html"   # 로컬 파일 경로
  content_type  = "text/html"                 # 콘텐츠 유형
  etag          = "${md5(file("main.html"))}"  # ETag 사용하여 변경 감지
}
```


### putabout.tf
```
# S3 버킷에 about.html 파일 업로드 (수정 시 덮어씀)
resource "aws_s3_object" "about" {
  bucket        = "버킷명"  # 존재하는 버킷 이름 사용
  key           = "about.html"               # 업로드할 파일 이름
  source        = "about.html"   # 로컬 파일 경로
  content_type  = "text/html"                 # 콘텐츠 유형
  etag          = "${md5(file("about.html"))}"  # ETag 사용하여 변경 감지
}
```

### putcontact.tf
```
# S3 버킷에 contact.html 파일 업로드 (수정 시 덮어씀)
resource "aws_s3_object" "contact" {
  bucket        = "버킷명"  # 존재하는 버킷 이름 사용
  key           = "contact.html"               # 업로드할 파일 이름
  source        = "contact.html"   # 로컬 파일 경로
  content_type  = "text/html"                 # 콘텐츠 유형
  etag          = "${md5(file("contact.html"))}"  # ETag 사용하여 변경 감지
}
```


### terraform 설정 적용
```
$ terraform init
$ terraform apply -auto-approve # yes입력 없이 자동 허용
```


### 성공적으로 S3의 정적페이지가 hosting된다.
![image](https://github.com/user-attachments/assets/3bf1e73d-c22f-49df-9af0-d5385ee00779)
![image](https://github.com/user-attachments/assets/523a05c2-0277-4b0b-a831-c8b18b34efb8)
![image](https://github.com/user-attachments/assets/43e3f05e-4715-44c6-a631-c7b2a5699af0)



### 나머지 html내용 일부 수정, css를 적용해본 후 다시 적용.
```
terraform init
terraform apply
```
### apply 완료
![image](https://github.com/user-attachments/assets/60342c7b-0932-4d01-8243-1c72327cca85)
### main
![image](https://github.com/user-attachments/assets/3bf1e73d-c22f-49df-9af0-d5385ee00779)

### about
![image](https://github.com/user-attachments/assets/ebd96992-acad-40ef-8007-b681974fa8dc)

### contact
![image](https://github.com/user-attachments/assets/84a45436-5936-4627-b83a-a61c216f94af)


### 성공적으로 S3의 bucket의 내부 object들이 수정된 모습이다.



## Trouble Shooting
### PutBucketPolicy 문제 발생
```
putting S3 Bucket (ce22-bucket1) Policy: operation error S3: PutBucketPolicy
```

- bocket.tf에 bucket의 obkect를 수정, 게시할 수 있는 권한 추가로 해결

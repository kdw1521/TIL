# S3 + Cloudfront로 프론트 정적 파일을 서빙 한다.

프론트 프로젝트가 React로 되어있어, S3에 올려 Cloudfront로 서빙한다.

# 준비

- AWS S3
- AWS Cloudfront
- 빌드된 React 프로젝트 (ex. dist 디렉토리)

# 구현

## 1. S3 생성

1-1. 버킷이름을 지정한다.  
1-2. 객체 소유권 - ACL 비활성화로 지정. (default)  
1-3. Cloudfront로 서빙하므로 `모든 퍼블릭 액세스 차단` 으로 설정.  
1-4. 나머지는 default 값으로 진행. (필요한게 있으면 설정해주면 된다.)

## 2. Cloudfront 생성

#### 1. 원본

1-1. Origin domain - 위에서 만든 S3로 지정해준다.  
1-2. 이름 - 프로젝트명과 연관있는 이름을 지정한다.

#### 2. 기본 캐시 동작

2-1. 뷰어 프로토콜 정책 - Redirect HTTP to HTTPS 로 설정  
2-2. 캐시 키 및 원본 요청 - CachingOptimized 로 설정

#### 3. 설정

3-1. 가격 분류 - 북미, 유렵, 아시아, 중동 및 아프리카에서 사용 으로 선택  
3-2. 대체 도메인 이름(CNAME) - 사용할 서브도메인을 입력한다. (ex. test.wando.com)  
3-3. Custom SSL certificate - 내가 발급한 인증서를 선택한다. (인증서는 버지니아 북부에 있어야함.)  
3-4. Default root object(기본값 루트 객체) - index.html 로 설정.

#### 4. 나머지 설정 안한것들

default 로 해두었다. 필요하다면 설정해주면 된다.

#### 5. 생성 후 오류페이지를 설정한다.

400, 403, 404 일 경우 index.html 로 설정해주었고, HTTP 응답코드는 200으로 해두었다.

## 3. S3 버킷정책에 Cloudfront 추가

#### 1. S3 의 권한 탭에 진입.

#### 2. 버킷 정책에 추가한다.

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::나의_S3_명/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "만든_Cloudfront의_ARN"
        }
      }
    }
  ]
}
```

json 자체를 넣어도 되고 정책생성기에서 GetObject로 추가해줘도 된다.

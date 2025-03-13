# Github-hosted Runner (with. Cloudfront, S3, pnpm, supabase)

React 프로젝트를 Runner를 사용해 AWS Cloudfront 와 S3에 배포를 진행한다.  
S3 배포 후 연결된 Cloudfont 무효화를 진행한다.

# 준비

- React 프로젝트
- AWS Cloudfront, S3, IAM(S3 권한을 가진) key
- .github/workflows/deploy.yml
- Slack webhook (optional)

# 구현

### 1. secrets 등록과 .github/workflows/deploy.yml 작성

1-1. secrets 를 등록한다.
해당 프로젝트 repository 에서 Settings > Secrets and variables > Actions 진입.  
Secrets 탭에서 `New Repository secret` 으로 등록한다.  
deploy.yml 에서 `${{ secrets.등록한_key_name_입력 }}` 이렇게 사용한다.

1-2. .github/workflows/deploy.yml

```yaml
# deploy.yml

name: React + Cloudfront + S3 Deploy

on:
  push:
    branches:
      - main

jobs:
  build:
    name: React build & deploy
    runs-on: ubuntu-latest

    steps:
      - name: checkout Github Action
        uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        name: Install pnpm
        with:
          version: 9
          run_install: false

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"

      - name: Get pnpm cache directory
        id: pnpm-cache-dir
        run: |
          echo "dir=$(pnpm store path)" >> $GITHUB_ENV

      - uses: actions/cache@v3
        id: pnpm-cache
        with:
          path: ${{ env.dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-

      - name: install pnpm dependencies
        run: pnpm install

      - name: react build
        env:
          VITE_SUPABASE_URL: ${{ secrets.VITE_SUPABASE_URL }}
          VITE_SUPABASE_ANON_KEY: ${{ secrets.VITE_SUPABASE_ANON_KEY }}
          VITE_PROFILE: PROD
        run: pnpm build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_S3_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_S3_SECRET_ACCESS_KEY_ID }}
          aws-region: ap-northeast-2

      - name: Upload to S3
        env:
          BUCKET_NAME: ${{ secrets.AWS_S3_BUCKET_NAME}}
        run: |
          aws s3 sync \
            ./dist s3://$BUCKET_NAME

      - name: CloudFront Invalidation
        env:
          CLOUD_FRONT_ID: ${{ secrets.AWS_CLOUDFRONT_ID}}
        run: |
          aws cloudfront create-invalidation \
            --distribution-id $CLOUD_FRONT_ID --paths '/*'

      - name: Notify Slack on Success
        if: success()
        run: |
          curl -X POST -H 'Content-type: application/json' --data '{
            "text": "[My-React_Project] 🚀 배포가 성공적으로 완료되었습니다!",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "🚀 *배포가 성공적으로 완료되었습니다!*\n"
                }
              },
              {
                "type": "section",
                "fields": [
                  {
                    "type": "mrkdwn",
                    "text": "*Branch:*\n${{ github.ref }}"
                  },
                  {
                    "type": "mrkdwn",
                    "text": "*Repository:*\n${{ github.repository }}"
                  }
                ]
              }
            ]
          }' ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Failure
        if: failure()
        run: |
          curl -X POST -H 'Content-type: application/json' --data '{
            "text": "[My-React_Project] ⚠ 배포에 실패했습니다.",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "⚠ *배포에 실패했습니다. 로그를 확인해주세요.*\n"
                }
              },
              {
                "type": "section",
                "fields": [
                  {
                    "type": "mrkdwn",
                    "text": "*Branch:*\n${{ github.ref }}"
                  },
                  {
                    "type": "mrkdwn",
                    "text": "*Repository:*\n${{ github.repository }}"
                  },
                  {
                    "type": "mrkdwn",
                    "text": "*Job Status:*\n${{ job.status }}"
                  }
                ]
              }
            ]
          }' ${{ secrets.SLACK_WEBHOOK_URL }}
```

패키지 매니저로 pnpm을 사용했고, lock 파일 변경으로 캐시사용여부를 판단해 dependency들을 install 할지 여부를 판단하게 해두었다.  
빌드 후 dist 디렉토리 내용을 S3에 올리고 Cloudfront 무효화를 진행한다.

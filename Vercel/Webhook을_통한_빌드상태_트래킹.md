# Webhook 을 통해 빌드상태를 받아 본다.

Vercel 과 github Repository를 연결하고 main, dev 등 merge가 되면 진행해 배포가 진행된다.  
이때 빌드 상태를 webhook을 통해 전달받을 수 있다.

# 준비

- Vercel (git repo 와 연결)

# 구현

1. Teams > 내가 만든 팀을 선택한다.
2. Settings > Webhooks 진입한다.
3. Events > 등록할 이벤트를 지정한다. (배포 관련 뿐 아니라 프로젝트, 공격 관련 훅도 받을 수 있다.)
4. Projects > 전체 프로젝트에 적용할지 특정 프로젝트에 적용할지 선택한다.
5. Endpoint > 이벤트를 수신할 endpoint를 지정한다. (특정 프로젝트의 endpoint 나 lambda 등 무엇을 사용해도 무방하다. 단, endpoint는 `POST` 메서드로 구현되어있어야 한다.)
6. 생성을 완료하면 `Secret값`이 나오는데 `x-vercel-signature`의 key로 헤더에 포함되어 온다. (이 값은 다시 볼수 없으니 잘 관리한다.)
7. endpoint 에서 Secret 값으로 검증을 한 뒤 필요한 로직을 작성한다.

```javascript
// endpoint 에서 Secret 값으로 검증 예시

/*
  body: vercel 에서 보내준 body 값 (주체: vercel)
  signature: vercel 에서 보내준 header의 x-vercel-signature 값 (주체: vercel)
  VERCEL_WEBHOOK_SECRET: vercel 에서 webhook 등록 후 받은 Secrets 값 (주체: 개발자)
*/
if (!verifySignature(body, signature, VERCEL_WEBHOOK_SECRET)) {
  return {
    statusCode: 401,
    body: JSON.stringify({ message: "Invalid signature" }),
  };
}

// Vercel signature 검증
const verifySignature = (body, signature, secret) => {
  const hmac = crypto.createHmac("sha1", secret);
  hmac.update(body, "utf8");
  const expectedSignature = hmac.digest("hex");
  return signature === expectedSignature;
};
```

# 관련 링크

- [Vercel Webhooks](https://vercel.com/docs/webhooks)

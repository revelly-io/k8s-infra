# Ghost Blog

Ghost 블로그를 Kustomize로 배포합니다 (SQLite 사용).

## 구조

```
kustomize/ghost/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml       # Deployment + PVC (ghost:6-alpine)
│   └── service.yaml          # ClusterIP Service
└── overlays/
    └── sandbox/
        ├── kustomization.yaml      # secretGenerator → Secret ghost-smtp (smtp.env)
        ├── patch-deployment.yaml   # URL, 리소스, envFrom(ghost-smtp)
        ├── patch-pvc.yaml          # storageClass: local-path
        ├── ingress.yaml            # blog.revelly.io → ghost:2368
        ├── smtp.env.example        # SMTP 템플릿 (저장소에 포함)
        └── smtp.env                # 실제 값 (gitignore, 로컬에서만 생성)
```

## SMTP (비밀번호 재설정 메일 등)

1. `overlays/sandbox/smtp.env.example`을 `smtp.env`로 복사한 뒤 `mail__from`, `mail__options__auth__pass` 등을 채운다. (`smtp.env`는 `.gitignore`에 있어 커밋되지 않음.)
2. `kustomization.yaml`의 `secretGenerator`가 `smtp.env`를 읽어 `Secret` `ghost-smtp`를 만든다. `patch-deployment`에서 `envFrom`으로 Ghost 컨테이너에 주입한다.
3. **Resend** 권장: [resend.com](https://resend.com) 무료 티어, SMTP `smtp.resend.com:465`(SSL), 사용자명 `resend`, 비밀번호는 API 키. 테스트 발신은 예: `Blog <onboarding@resend.dev>`. 자기 도메인을 쓰면 Resend에서 도메인 검증 후 `mail__from`을 맞춘다. Gmail은 2단계 인증·앱 비밀번호가 필요하다.
4. 배포 후 Secret을 변경했으면 Pod가 새 값을 쓰도록 `kubectl rollout restart deployment/ghost -n sandbox`를 실행한다.

## 배포

```bash
kubectl apply -k kustomize/ghost/overlays/sandbox --context sandbox
```

원격 노드에서만 `kubectl`이 동작하는 경우:

```bash
kubectl kustomize kustomize/ghost/overlays/sandbox | ssh sandbox001 "kubectl apply -f -"
```

(`smtp.env`가 있는 로컬/CI에서 `kustomize`를 실행해야 `ghost-smtp` Secret이 포함된다.)

## 접속

- 블로그: https://blog.revelly.io
- Admin: https://blog.revelly.io/ghost

## 참고

- DB: SQLite (`/var/lib/ghost/content/data/ghost.db`)
- 콘텐츠(글, 이미지, DB)는 PVC `ghost-content` (local-path, 5Gi)에 저장
- Ghost URL을 `http://`로 설정해야 Cloudflare Tunnel 환경에서 리다이렉트 루프 방지

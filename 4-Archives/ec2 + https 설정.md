
# 🚀 방법 1 (추천): Nginx + Let’s Encrypt (Certbot)

## ✅ 전체 구조

```
Internet → Nginx (HTTPS) → Admin Web (localhost:3000 등)
```

---

# 1️⃣ 도메인 필요 (필수)

예:

```
admin.yourdomain.com
```

👉 이게 EC2 퍼블릭 IP로 연결되어 있어야 함

---

# 2️⃣ Nginx 설치

```bash
sudo dnf install nginx -y   # Amazon Linux 2023
# 또는
sudo yum install nginx -y   # Amazon Linux 2
```

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

# 3️⃣ Certbot 설치 (Let's Encrypt)

```bash
sudo dnf install certbot python3-certbot-nginx -y
```

---

# 4️⃣ Nginx 설정 (HTTP 먼저)

```bash
sudo vi /etc/nginx/conf.d/admin.conf
```

```nginx
server {
    listen 80;
    server_name admin.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;  # admin 서버
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

# 5️⃣ HTTPS 적용 (자동)

```bash
sudo certbot --nginx -d admin.yourdomain.com
```

👉 이거 하면:

- 인증서 발급
    
- nginx 자동 수정
    
- HTTPS 적용까지 끝
    

---

# 6️⃣ 자동 갱신 (중요)

```bash
sudo crontab -e
```

```bash
0 0 * * * certbot renew --quiet
```

---

# 🔥 결과

이제:

```
http://admin.yourdomain.com ❌
https://admin.yourdomain.com ✅
```

---

# ⚠️ 자주 터지는 문제

## 1. 포트 막힘

EC2 보안그룹:

- 80 (HTTP)
    
- 443 (HTTPS)
    

👉 둘 다 열어야 함

---

## 2. 도메인 연결 안됨

👉 Route53 / DNS에서 A 레코드 확인

---

## 3. Docker 쓸 때

```nginx
proxy_pass http://127.0.0.1:컨테이너포트;
```

또는

```bash
docker run -p 3000:3000
```

---

# 💡 방법 2 (대안)

## ALB (로드밸런서)

```
Internet → ALB(HTTPS) → EC2(Nginx or App)
```

👉 장점:

- SSL 관리 없음
    
- AWS가 자동 처리
    

👉 단점:

- 돈 듬 (월 ~2~3만원 이상)
    

---

# 🚀 너 상황 기준 추천

지금 보면:

- EC2 직접 운영
    
- Docker + Nginx
    
- 소규모 / 초기 서비스
    

👉 **무조건 Certbot 방식 추천**

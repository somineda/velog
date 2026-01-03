<p>느린  우체통 서비스를 배포했다
근데 에러를 너무 많이 만나서
트러블 슈팅 ... </p>
<h2 id="배포-스펙">배포 스펙</h2>
<table>
<thead>
<tr>
<th>항목</th>
<th>내용</th>
</tr>
</thead>
<tbody><tr>
<td>서버</td>
<td>AWS EC2 t2.micro (프리티어)</td>
</tr>
<tr>
<td>OS</td>
<td>Ubuntu 24.04 LTS</td>
</tr>
<tr>
<td>웹 서버</td>
<td>Nginx (리버스 프록시)</td>
</tr>
<tr>
<td>앱 서버</td>
<td>Uvicorn (FastAPI)</td>
</tr>
<tr>
<td>프로세스 관리</td>
<td>systemd</td>
</tr>
<tr>
<td>데이터베이스</td>
<td>PostgreSQL</td>
</tr>
<tr>
<td>비용</td>
<td><strong>무료</strong> (프리티어 12개월)</td>
</tr>
</tbody></table>
<h2 id="ec2-인스턴스-생성">EC2 인스턴스 생성</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/dde29924-fbdc-4b81-aadf-f8ff6b30770e/image.png" /></p>
<h3 id="보안-그룹-설정">보안 그룹 설정</h3>
<table>
<thead>
<tr>
<th>유형</th>
<th>프로토콜</th>
<th>포트</th>
<th>소스</th>
<th>용도</th>
</tr>
</thead>
<tbody><tr>
<td>SSH</td>
<td>TCP</td>
<td>22</td>
<td>내 IP</td>
<td>SSH 접속</td>
</tr>
<tr>
<td>HTTP</td>
<td>TCP</td>
<td>80</td>
<td>0.0.0.0/0</td>
<td>웹 접속</td>
</tr>
<tr>
<td>HTTPS</td>
<td>TCP</td>
<td>443</td>
<td>0.0.0.0/0</td>
<td>SSL (선택)</td>
</tr>
<tr>
<td>Custom TCP</td>
<td>TCP</td>
<td>8000</td>
<td>0.0.0.0/0</td>
<td>FastAPI (개발용)</td>
</tr>
</tbody></table>
<h2 id="ssh-접속">SSH 접속</h2>
<pre><code class="language-bash">chmod 400 my-key.pem

# SSH 접속
ssh -i my-key.pem ubuntu@your-ec2-ip</code></pre>
<h2 id="프로젝트-배포-준비">프로젝트 배포 준비</h2>
<pre><code class="language-bash">cd /home/ubuntu
git clone https://github.com/username/slowmail.git
cd slowmail</code></pre>
<h4 id="시스템-패키지-설치">시스템 패키지 설치</h4>
<pre><code class="language-bash"># 시스템 업데이트
sudo apt update &amp;&amp; sudo apt upgrade -y

# Python 3.12 및 필수 패키지
sudo apt install -y python3.12 python3.12-venv python3-pip

# PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Nginx
sudo apt install -y nginx</code></pre>
<h2 id="postgresql-설정">PostgreSQL 설정</h2>
<h4 id="db-생성">DB 생성</h4>
<pre><code class="language-bash">sudo -u postgres psql</code></pre>
<p>순탄하게 진행 되는 줄 알았으나..
여기서 문제가 발생하게 되는데
두둥</p>
<h3 id="⚠️-트러블슈팅">⚠️ 트러블슈팅</h3>
<pre><code>tortoise.exceptions.OperationalError: permission denied for schema public</code></pre><p>이런 에러를 마주쳤다
찾아보니 PostgreSQL 15부터 보안 정책 변경되어서 뜨는 걸로 추정
이전 버전에서는 모든 사용자가 public 스키마에 테이블을 만들 수 있었는데
PostgreSQL 15부터 기본적으로 public 스키마에 대한 CREATE 권한이 제거됨</p>
<p>따라서 연결에 사용하는 DB 유저가 테이블 생성/수정 권한이 없음!!</p>
<pre><code class="language-sql">-- slowmail 데이터베이스에 연결
\c slowmail

-- public schema 권한 부여
GRANT ALL ON SCHEMA public TO slowmail;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO slowmail;

-- DB 소유자 변경
ALTER DATABASE slowmail OWNER TO slowmail;
</code></pre>
<p>비밀번호 입력 후 접속되면 성공</p>
<pre><code class="language-bash">psql -h localhost -U slowmail -d slowmail</code></pre>
<h2 id="환경-변수-설정">환경 변수 설정</h2>
<pre><code class="language-bash">nano .env</code></pre>
<pre><code class="language-env">DATABASE_URL=postgres://slowmail:slowmail123@localhost:5432/slowmail
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password_here</code></pre>
<p>env 파일에 권한 설정도 완료했다!</p>
<h2 id="systemd-서비스-설정">systemd 서비스 설정</h2>
<h3 id="서비스-파일-생성">서비스 파일 생성</h3>
<pre><code class="language-bash">sudo nano /etc/systemd/system/slowmail.service</code></pre>
<pre><code class="language-ini">[Unit]
Description=Slow Mailbox FastAPI Application
After=network.target postgresql.service

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/slowmail
Environment=&quot;PATH=/home/ubuntu/slowmail/venv/bin&quot;
Environment=&quot;PYTHONUNBUFFERED=1&quot;
EnvironmentFile=/home/ubuntu/slowmail/.env
ExecStart=/home/ubuntu/slowmail/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10
TimeoutStartSec=300
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target</code></pre>
<p><strong>각 항목 설명:</strong></p>
<table>
<thead>
<tr>
<th>항목</th>
<th>설명</th>
</tr>
</thead>
<tbody><tr>
<td><code>After</code></td>
<td>PostgreSQL이 먼저 시작되도록</td>
</tr>
<tr>
<td><code>Type=simple</code></td>
<td>포그라운드 프로세스</td>
</tr>
<tr>
<td><code>User=ubuntu</code></td>
<td>ubuntu 유저로 실행</td>
</tr>
<tr>
<td><code>WorkingDirectory</code></td>
<td>프로젝트 경로</td>
</tr>
<tr>
<td><code>Environment=&quot;PATH=...&quot;</code></td>
<td>venv Python 사용</td>
</tr>
<tr>
<td><code>EnvironmentFile</code></td>
<td>.env 자동 로드</td>
</tr>
<tr>
<td><code>ExecStart</code></td>
<td>실행 명령어</td>
</tr>
<tr>
<td><code>Restart=always</code></td>
<td>실패 시 자동 재시작</td>
</tr>
<tr>
<td><code>TimeoutStartSec=300</code></td>
<td>시작 타임아웃 5분</td>
</tr>
</tbody></table>
<h3 id="⚠️-트러블슈팅-status203exec">⚠️ 트러블슈팅: status=203/EXEC</h3>
<p><strong>에러</strong></p>
<pre><code>Main process exited, code=exited, status=203/EXEC</code></pre><p>경로가 없거나 실행 파일이 잘못 되었다고 뜬다..</p>
<p><strong>확인 방법</strong></p>
<pre><code class="language-bash"># uvicorn 경로 확인
which uvicorn  

# 직접 실행 테스트
/home/ubuntu/slowmail/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000</code></pre>
<h3 id="서비스-시작">서비스 시작</h3>
<pre><code class="language-bash"># systemd 리로드
sudo systemctl daemon-reload

# 서비스 활성화 (부팅 시 자동 시작)
sudo systemctl enable slowmail

# 서비스 시작
sudo systemctl start slowmail

# 상태 확인
sudo systemctl status slowmail</code></pre>
<p><strong>정상 상태</strong></p>
<pre><code>● slowmail.service - Slow Mailbox FastAPI Application
     Loaded: loaded
     Active: active (running) since ...
   Main PID: 12275 (uvicorn)
      Tasks: 6
     Memory: 44.3M
        CPU: 1.045s
     CGroup: /system.slice/slowmail.service
             └─12275 /home/ubuntu/slowmail/venv/bin/python3 ...

Dec 31 02:45:10 ip-172-31-34-186 uvicorn[12275]: INFO:     Uvicorn running on http://0.0.0.0:8000</code></pre><h2 id="nginx-설정">Nginx 설정</h2>
<h3 id="nginx-설정-파일">Nginx 설정 파일</h3>
<pre><code class="language-bash">sudo nano /etc/nginx/sites-available/slowmail</code></pre>
<pre><code class="language-nginx">server {
    listen 80;
    server_name _;  

    client_max_body_size 10M;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 타임아웃 설정 (APScheduler 때문에)
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}</code></pre>
<p><strong>설정</strong></p>
<ul>
<li><code>listen 80</code>: HTTP 포트</li>
<li><code>server_name _</code>: 모든 도메인 허용 (IP로 접속)</li>
<li><code>proxy_pass</code>: FastAPI(8000) → Nginx(80)</li>
<li><code>client_max_body_size</code>: 업로드 파일 크기 제한</li>
</ul>
<h3 id="⚠️-트러블슈팅-static-파일-403-forbidden">⚠️ 트러블슈팅: Static 파일 403 Forbidden</h3>
<p>CSS/JS가 로드 안되는 문제 발생 ㄷㄷ
FastAPI가 static 파일을 서빙하는데 Nginx에서 별도 처리하려 했던게 문제였다</p>
<p>모든 요청을 FastAPI로 프록시하도록 설정 변경!</p>
<pre><code class="language-nginx">location /static {
    alias /home/ubuntu/slowmail/static;  # 권한 문제 발생 가능
}

# 이렇게 변경
location / {
    proxy_pass http://127.0.0.1:8000;  # FastAPI가 알아서 처리
}</code></pre>
<h3 id="nginx-활성화">Nginx 활성화</h3>
<pre><code class="language-bash"># 심볼릭 링크 생성
sudo ln -s /etc/nginx/sites-available/slowmail /etc/nginx/sites-enabled/

# 기본 설정 제거
sudo rm /etc/nginx/sites-enabled/default

# 설정 테스트
sudo nginx -t

# Nginx 재시작
sudo systemctl restart nginx

# 상태 확인
sudo systemctl status nginx</code></pre>
<p><strong>정상 상태:</strong></p>
<pre><code>● nginx.service - A high performance web server
     Loaded: loaded
     Active: active (running)</code></pre><h2 id="배포-확인">배포 확인</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/50493c66-1a8e-4f2d-9f5f-24b19f4050c6/image.png" /></p>
<h2 id="10-자동화-배포-스크립트">10. 자동화 배포 스크립트</h2>
<p>매번 수동으로 하기 귀찮으니 스크립트로 만들었다</p>
<h3 id="deploysh-ec2에서-실행">deploy.sh (EC2에서 실행)</h3>
<pre><code class="language-bash">#!/bin/bash

GREEN='\033[0;32m'
NC='\033[0m'

echo -e &quot;${GREEN}[1/5] 코드 업데이트...${NC}&quot;
git pull

echo -e &quot;${GREEN}[2/5] 패키지 설치...${NC}&quot;
source venv/bin/activate
pip install -r requirements.txt

echo -e &quot;${GREEN}[3/5] 서비스 재시작...${NC}&quot;
sudo systemctl restart slowmail
sudo systemctl restart nginx

echo -e &quot;${GREEN}[4/5] 상태 확인...${NC}&quot;
sleep 5
sudo systemctl status slowmail --no-pager

echo -e &quot;${GREEN}[5/5] 완료!${NC}&quot;</code></pre>
<p><strong>사용 방법</strong></p>
<pre><code class="language-bash">chmod +x deploy.sh
./deploy.sh</code></pre>
<h3 id="자동-배포-로컬에서-실행">자동 배포 (로컬에서 실행)</h3>
<pre><code class="language-bash">#!/bin/bash
# auto_deploy.sh

SSH_KEY=&quot;$HOME/Downloads/my-key.pem&quot;
EC2_IP=&quot;3.39.194.234&quot;

echo &quot;배포 시작...&quot;

ssh -i &quot;$SSH_KEY&quot; ubuntu@&quot;$EC2_IP&quot; &lt;&lt; 'ENDSSH'
cd /home/ubuntu/slowmail
git pull
source venv/bin/activate
pip install -q -r requirements.txt
sudo systemctl restart slowmail
sudo systemctl restart nginx
sleep 10
sudo systemctl status slowmail --no-pager
ENDSSH

echo &quot;배포 완료! http://$EC2_IP&quot;</code></pre>
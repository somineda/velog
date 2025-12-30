<blockquote>
<p><strong>Django와 PostgreSQL을 도커 컴포즈로 한 번에 실행하기</strong></p>
</blockquote>
<h3 id="📁-프로젝트-구조">📁 프로젝트 구조</h3>
<pre><code>project/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   ├── wsgi.py
│   │   └── asgi.py
│   ├── requirements.txt
│   ├── manage.py
│   └── Dockerfile
├── nginx/
│   ├── nginx.conf
│   └── Dockerfile
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env
├── .env.example
└── README.md</code></pre><h3 id="🧩-환경-구성">🧩 환경 구성</h3>
<h4 id="requirementstxt">requirements.txt</h4>
<pre><code>Django==4.2.7
psycopg2-binary==2.9.9
djangorestframework==3.14.0
django-cors-headers==4.3.0
pillow==10.1.0
gunicorn==21.2.0
python-decouple==3.8
django-debug-toolbar==4.2.0
celery==5.3.4
redis==5.0.1</code></pre><h3 id="🐳-dockerfile">🐳 Dockerfile</h3>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/9fadc7a5-1c15-4503-b296-17280eb9b40e/image.png" /></p>
<pre><code class="language-bash"># ---- Builder Stage ----

FROM python:3.13-slim as builder

# 필수 패키지 설치
RUN apt-get update &amp;&amp; apt-get install -y build-essential libpq-dev

WORKDIR /app

# Poetry 설치 및 의존성 설치
RUN pip install --no-cache-dir poetry &amp;&amp; poetry config virtualenvs.create false
COPY pyproject.toml poetry.lock ./
RUN poetry install --no-interaction --no-root

FROM python:3.13-slim

# 런타임용 최소 패키지
RUN apt-get update &amp;&amp; apt-get install -y libpq5

WORKDIR /app

# Builder에서 site-packages 복사
COPY --from=builder /usr/local/lib/python3.13/site-packages/ /usr/local/lib/python3.13/site-packages/

# 프로젝트 복사
COPY . .

# Gunicorn으로 앱 실행
CMD [&quot;gunicorn&quot;, &quot;--bind&quot;, &quot;0.0.0.0:8000&quot;, &quot;config.wsgi:application&quot;]</code></pre>
<h3 id="⚙️-docker-composeyml">⚙️ docker-compose.yml</h3>
<h4 id="django-웹-서버">Django 웹 서버</h4>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/631a3b21-7470-467b-968c-283f2d2147a3/image.png" /></p>
<h4 id="celery-비동기-워커">Celery 비동기 워커</h4>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/fc9c5a3e-3e03-414a-88e6-1277e7037733/image.png" /></p>
<h4 id="nginx-리버스-프록시">Nginx 리버스 프록시</h4>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/7624bdec-8770-4b41-92ac-c2ed0a118d97/image.png" /></p>
<h4 id="postgresql-pgvector-포함">PostgreSQL (pgvector 포함)</h4>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/9f58dfb2-98e5-43cb-91e8-d7d4e4ae5c12/image.png" /></p>
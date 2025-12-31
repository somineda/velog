<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/89f01357-068a-477d-ab1d-119579b801d1/image.jpg" />
새해를 기념해서 새해 다짐을 한달뒤에 리마인드 시켜주는
느린우체통 같은 프로젝트를 만들었다!</p>
<p>7일 후와 30일 후에 자동으로 이메일을 발송해주는 기능이 있는데
사용자가 작성한 편지를 db에 저장하고 정해진 시간에 자동으로 이메일을 보내기 위해서 
FastAPI와 APScheduler를 조합해서 백그라운드 구현했다!!</p>
<h2 id="celery가-아니라-왜-apscheduler를">Celery가 아니라 왜 APScheduler를??</h2>
<table>
<thead>
<tr>
<th></th>
<th>장점</th>
<th>단점</th>
</tr>
</thead>
<tbody><tr>
<td><strong>Celery</strong></td>
<td>강력한 분산 처리</td>
<td>복잡한 설정, Redis/RabbitMQ 필요</td>
</tr>
<tr>
<td><strong>Cron</strong></td>
<td>간단함</td>
<td>Python 코드와 분리, 유연성 부족</td>
</tr>
<tr>
<td><strong>APScheduler</strong></td>
<td>가벼움, FastAPI 통합 쉬움</td>
<td>단일 프로세스 제한</td>
</tr>
</tbody></table>
<p>이러한 장단점들이 있는데</p>
<p>지금 만든 것 처럼 간단한 프로젝트에 좋고
Redis 같은 다른 인프라가 필요없고
FastAPI 웹 프레임워크를 쓰기로 결정했기 때문에 </p>
<p><strong>APScheduler</strong>를 사용했다!!</p>
<h2 id="시스템-아키텍쳐">시스템 아키텍쳐</h2>
<p>전체 플로우는 대충 이렇게 된다</p>
<pre><code>1. 사용자가 편지 작성
   ↓
2. DB에 저장 (send_at: 현재 + 7일, second_send_at: 현재 + 30일)
   ↓
3. APScheduler가 1분마다 실행
   ↓
4. 현재 시간 &gt;= send_at인 편지 조회
   ↓
5. 이메일 발송
   ↓
6. DB 업데이트 (sent: True)</code></pre><p>DB 모델 설계를</p>
<pre><code class="language-python">  # 발송 시간
  send_at = fields.DatetimeField()              # 첫 번째 발송 시간 (7일 후)
  second_send_at = fields.DatetimeField()       # 두 번째 발송 시간 (30일 후)

  # 발송 상태
  sent = fields.BooleanField(default=False)
  sent_at = fields.DatetimeField(null=True)
  second_sent = fields.BooleanField(default=False)
  second_sent_at = fields.DatetimeField(null=True)

#등등등</code></pre>
<ul>
<li><code>send_at</code>, <code>second_send_at</code>: 발송 예정 시간을 저장</li>
<li><code>sent</code>, <code>second_sent</code>: 발송 여부를 boolean으로 관리</li>
<li><code>sent_at</code>, <code>second_sent_at</code>: 실제 발송된 시간을 기록
이렇게 했다</li>
</ul>
<hr />
<h2 id="apscheduler-설정">APScheduler 설정</h2>
<p>백그라운드에서 주기적으로 이메일을 발송하는 스케줄러 생성하면 되는데
scheduler.py 에서 비동기 스케줄러를 생성했다!</p>
<pre><code class="language-python"># 비동기 스케줄러 생성
scheduler = AsyncIOScheduler()</code></pre>
<h4 id="1분마다-실행되는-스케줄-작업-함수">1분마다 실행되는 스케줄 작업 함수</h4>
<pre><code class="language-python">from datetime import datetime, timezone
from models import Letter
from email_service import send_email

#1분마다 실행되는 함수
async def check_and_send_letters_async():
    try:
        now = datetime.now(timezone.utc)

        # 첫 번째 발송 확인 (7일 후)
        letters_first_send = await Letter.filter(
            sent=False,              # 아직 안보낸 편지
            send_at__lte=now         # 발송 시간이 지난 편지
        ).all()

        #이메일 발송
        for letter in letters_first_send:
            success = send_email(
                letter.recipient_email,
                letter.content,
                is_second_send=False
            )

            if success: #발송 완료로 업데이트
                letter.sent = True
                letter.sent_at = datetime.now(timezone.utc)
                await letter.save()
                print(f&quot;[First Send] Letter {letter.id} sent&quot;)

        # 두 번째 발송 확인 (30일 후)
        letters_second_send = await Letter.filter(
            second_sent=False,
            second_send_at__lte=now
        ).all()

        for letter in letters_second_send:
            success = send_email(
                letter.recipient_email,
                letter.content,
                is_second_send=True
            )

            if success:
                letter.second_sent = True
                letter.second_sent_at = datetime.now(timezone.utc)
                await letter.save()
                print(f&quot;[Second Send] Letter {letter.id} sent&quot;)

    except Exception as e:
        print(f&quot;Error in scheduler: {e}&quot;)</code></pre>
<ol>
<li><code>sent=False</code> AND <code>send_at &lt;= now</code> → 발송 대상</li>
<li>이메일 발송 성공 시 <code>sent=True</code>로 업데이트</li>
<li>같은 로직을 두 번째 발송에도 적용</li>
</ol>
<h4 id="스케줄러-시작">스케줄러 시작</h4>
<pre><code class="language-python">def start_scheduler():
    scheduler.add_job(
        check_and_send_letters_async,  
        'interval',                   
        minutes=1,                      # 1분마다 주기적으로 실행
        id='check_letters',
        replace_existing=True
    )
    scheduler.start()
    print(&quot;Scheduler started&quot;)

def shutdown_scheduler():
    if scheduler.running:
        scheduler.shutdown()
        print(&quot;Scheduler stopped&quot;)</code></pre>
<p>추가로</p>
<p>main.py 안에 FastAPI 앱 시작/종료 시 스케줄러도 함께 관리해주는
코드를 추가했다!!</p>
<p>그러나........</p>
<p>일은 항상 순조롭게 진행이 되지 않는데..................</p>
<h2 id="💣-트러블슈팅">💣 트러블슈팅</h2>
<h3 id="문제-1-스케줄러가-두-번-실행됨">문제 1: 스케줄러가 두 번 실행됨..</h3>
<p><strong>증상:</strong></p>
<pre><code>Scheduler started
Scheduler started</code></pre><p>FastAPI의 hot reload 기능이 앱을 두 번 시작된게 문제였음!!!</p>
<p><strong>해결</strong></p>
<pre><code class="language-python">import os

@app.on_event(&quot;startup&quot;)
async def startup_event():
    # 개발 모드에서 메인 프로세스에서만 실행
    if os.getenv(&quot;RUN_MAIN&quot;) != &quot;true&quot; or os.getenv(&quot;WERKZEUG_RUN_MAIN&quot;) != &quot;true&quot;:
        scheduler.start_scheduler()</code></pre>
<p>또는 프로덕션에서만 스케줄러 실행되게 설정</p>
<pre><code class="language-bash">#개발
uvicorn main:app --reload

#프로덕션 (스케줄러 실행)
uvicorn main:app --host 0.0.0.0 --port 8000</code></pre>
<h3 id="문제-2-db-연결이-끊김">문제 2: DB 연결이 끊김</h3>
<p><strong>증상:</strong></p>
<pre><code>asyncpg.exceptions.InterfaceError: connection is closed</code></pre><p>스케줄러가 오래 실행되면 DB 연결이 타임아웃 되었다..... ㅎ</p>
<p><strong>해결</strong> 
TortoiseORM 설정에서 connection pool 조정했음</p>
<pre><code class="language-python">await Tortoise.init(
    db_url=settings.DATABASE_URL,
    modules={'models': ['models']},
    #Connection pool 설정
    _create_db=True,
    connection_class=asyncpg.connection.Connection,
    min_pool_size=1,
    max_pool_size=10,
)</code></pre>
<p>현재 이메일 발송은 비동기처리가 안되어있는데
동시에 많은 이메일이 발송 될 경우
문제가 백퍼 생길 것 같아서 비동기처리로 수정이 필요할 것 같다!</p>
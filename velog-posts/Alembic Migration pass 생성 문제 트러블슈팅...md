<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/7c4f86c8-5c5d-4106-9b9c-a6cf01c26559/image.jpg" /></p>
<blockquote>
<p>왜!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! upgrade() 함수가 비어 있을까.............
— FastAPI + SQLAlchemy 환경 트러블슈팅 기록</p>
</blockquote>
<h2 id="🧠-발생한-문제">🧠 발생한 문제..</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/04950639-769c-4706-92dc-eeaa750920b6/image.png" />
이메일 인증 발송, 검증 기능을 만들면서 <strong><code>VerifiedEmail</code></strong> 모델을 추가했다
그리고 자연스럽게 마이그레이션 진행했는데..</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/6e3bc933-6df9-41a0-9794-3ff7e2d3ea7f/image.png" /></p>
<p>db에 VerifiedEmail 테이블이 없댄다...😱😱😱😱😱😱😱😱😱😱😱</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/693758e7-6fae-4dec-b044-eb4b60c39876/image.png" />
보니까 자동 생성된 migration 파일이 이렇게 비어있었다..........에휴</p>
<h2 id="🧭-디버깅">🧭 디버깅</h2>
<h4 id="1️⃣-모델이-basemetadata에-등록되었는지-확인해보기">1️⃣ 모델이 Base.metadata에 등록되었는지 확인해보기</h4>
<pre><code>=== Base object id ===
4690350416

=== Registered tables ===
  - availabilities
  - events
  - event_final_choices
  - participants
  - social_accounts
  - event_time_slots
  - users
  - verified_emails</code></pre><p>  모델 등록 정상 확인 완.</p>
<h4 id="2️⃣-db와-metadata-차이-비교">2️⃣ DB와 Metadata 차이 비교</h4>
<pre><code>=== DB에 있는 테이블 ===
{'participants', 'social_accounts', 'alembic_version', 'users', 'event_final_choices', 'availabilities', 'event_time_slots', 'events'}

=== Metadata에 있는 테이블 ===
{'participants', 'social_accounts', 'verified_emails', 'event_final_choices', 'users', 'availabilities', 'event_time_slots', 'events'}

=== Metadata에만 있는 테이블 (새로 생성해야 할 것) ===
{'verified_emails'}   </code></pre><p><strong><code>{'verified_emails'}</code></strong> → DB에 없음</p>
<h4 id="3️⃣-alembic이-인식하는-차이-직접-확인">3️⃣ Alembic이 인식하는 차이 직접 확인</h4>
<pre><code>  === Alembic이 감지한 차이점 ===
('add_table', Table('verified_emails', MetaData(), Column('id', Integer(), table=&lt;verified_emails&gt;, primary_key=True, nullable=False), Column('email', String(), table=&lt;verified_emails&gt;, nullable=False), Column('verified_at', DateTime(timezone=True), table=&lt;verified_emails&gt;, nullable=False, server_default=DefaultClause(&lt;sqlalchemy.sql.functions.now at 0x1052684d0; now&gt;, for_update=False)), schema=None))
('add_index', Index('ix_verified_emails_email', Column('email', String(), table=&lt;verified_emails&gt;, nullable=False), unique=True))
('add_index', Index('ix_verified_emails_id', Column('id', Integer(), table=&lt;verified_emails&gt;, primary_key=True, nullable=False)))</code></pre><p> <strong><code>('add_table', Table('verified_emails', ...))</code></strong> 이렇게 나오는거 보면 Alembic도 변경사항 인식하는걸로 보이는데
 왜............. migration 파일은 pass냐고요....................</p>
<p> <img alt="" src="https://velog.velcdn.com/images/sommnie/post/f352c6fd-bce7-40ce-baa9-a66165bf27bf/image.jpg" />
근데 alembic/env.py 파일 열어보니까
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/af6076e5-e50a-4e7e-8415-cc9ac51e3691/image.png" />
ㅋㅋ
함수만 있고 호출을 안시킴... 에휴
Alembic은 env.py를 실행할 때 이 함수들이 호출되어야
DB와 메타데이터를 비교하고 실제 migration 내용을 생성하는데
왜때문에 저게 없는지 모르겠다
지웠나.. </p>
<pre><code>if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()</code></pre><p>  호출코드 추가하니까
  <img alt="" src="https://velog.velcdn.com/images/sommnie/post/2edf6789-626d-407f-84a3-6d4a1187d69b/image.png" />
바로 생겼죠?</p>
<p>이렇게 멍청 포인트 적립 +1
트러블슈팅이라 하기도 민망하다 허허..</p>
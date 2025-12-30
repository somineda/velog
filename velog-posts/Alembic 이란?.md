<p>Django만 쓰다가 처음 FastAPI 웹개발 프로젝트 시작 2시간차...
새로운 단어를 발견했다...
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/b5506fd2-c094-4128-92a4-756818097acd/image.jpg" /></p>
<p>Alembic을 세팅해야한다는데...</p>
<h1 id="두-alembic-둥">두 Alembic 둥</h1>
<h2 id="💡-alembic이란">💡 Alembic이란...?</h2>
<p><strong>Alembic</strong>은 <strong>SQLAlchemy</strong>와 함께 사용하는<br />파이썬에서 사용하는 db 마이그레이션 도구</p>
<p>정리해보면</p>
<blockquote>
<p>🧠 Django에서 <code>makemigrations</code>, <code>migrate</code> 역할을 SQLAlchemy 프로젝트에서 대신해주는 역할</p>
</blockquote>
<hr />
<h2 id="⚙️-alembic의-주요-특징">⚙️ Alembic의 주요 특징</h2>
<table>
<thead>
<tr>
<th align="center">기능</th>
<th align="left">설명</th>
</tr>
</thead>
<tbody><tr>
<td align="center"><strong>버전 관리</strong></td>
<td align="left">DB 변경 이력을 버전별로 추적</td>
</tr>
<tr>
<td align="center"><strong>자동 생성</strong></td>
<td align="left">SQLAlchemy 모델 변경 시 자동으로 마이그레이션 파일 생성</td>
</tr>
<tr>
<td align="center"><strong>롤백 지원</strong></td>
<td align="left">이전 버전으로 되돌릴 수 있음</td>
</tr>
<tr>
<td align="center"><strong>다중 환경 지원</strong></td>
<td align="left">개발, 테스트, 운영 DB를 각각 관리 가능</td>
</tr>
</tbody></table>
<hr />
<h2 id="🧱-이제-alembic-설치를-해보자">🧱 이제 Alembic 설치를 해보자</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/502a9939-1ee0-434e-8e6d-d856a8fa253a/image.png" /></p>
<h2 id="🥐-alembic-초기화">🥐 Alembic 초기화</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/02771a87-aedf-4d76-968d-9d3ca14c655c/image.png" />
이렇게 하면 프로젝트 안에 파일/폴더가 생성된다!
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/707d49ab-ba2d-4a3f-b797-9e0382c2fef3/image.png" /></p>
<ul>
<li>versions: 마이그레이션 할 스크립트 코드</li>
<li>env.py: 마이그레이션 시 실행되는 서버 연결 및 마이그레이션 실행 코드</li>
<li>script.py.mako: 마이그레이션 템플릿 파일</li>
</ul>
<hr />
<h2 id="🎮-모델-만들어서-테스트-해보기">🎮 모델 만들어서 테스트 해보기</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/b6fe646a-1e22-456f-a8e4-45bb7f82d901/image.png" />
모델 폴더에 user.py 파일 생성</p>
<p>Alembic이 user 모델을 인식하려면 
어딘가에서 User 를 import 해서 Base.metadata에 올라오게 해야 됨</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/7bf75d5b-bb02-4422-8c59-f9817c2643e2/image.png" />
이렇게 database.py에 import 추가!</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/664f8798-ab9e-45a7-b9af-4c2aeb84f85a/image.png" />
마이그레이션 새 파일 생성.......
하려고 했으나</p>
<pre><code>sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not translate host name &quot;db&quot; to address: nodename nor servname provided, or not known
</code></pre><p>에러발생...
아촤촤 
.env 안 DB 주소가 Docker용으로 설정한 걸 까먹음..</p>
<h4 id="그래서-alembic도-docker-안에서-실행하기로-함">그래서 Alembic도 Docker 안에서 실행하기로 함!!</h4>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/7975b3d1-fd52-4fc4-a9bd-bc8de8e6736b/image.png" />
컨테이너 안에서 설치 확인!</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/dd46a680-9f1d-42d6-9690-b14b79ca5f6c/image.png" /></p>
<p>이렇게 user 테이블 생성된걸 볼 수 있다!!!
![]
(<a href="https://velog.velcdn.com/images/sommnie/post/bc25583c-1435-474c-a5a2-881a29311d94/image.png">https://velog.velcdn.com/images/sommnie/post/bc25583c-1435-474c-a5a2-881a29311d94/image.png</a>)</p>
<p>db 접속해서 유저 테이블 생성된거 확인 가능!</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/dd705589-fb2d-49a9-aac0-21ad5632671b/image.png" /></p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/f48c0a87-9c7c-408b-8466-8726a0c7834f/image.jpg" />
이제 진짜 모델 작성해보자...</p>
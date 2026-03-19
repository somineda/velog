<blockquote>
<p>매일 5번 수동 캡처 + 에어테이블 업로드를 하던 귀찮은 작업을 완전히 자동화 </p>
</blockquote>
<p>부트캠프에서 조교로 수업 관리를 하면서 
매일 <strong>ZEP 화면을 캡처해서 에어테이블에 출결 자료로 올려야</strong> 하는 업무가 있었다</p>
<p>캡처 시간은 하루 4번:</p>
<ul>
<li>10:09 / 11:00 / 16:00 / 18:40</li>
</ul>
<p>이 중 10:09, 16:00, 18:40에 찍은 캡처는 에어테이블의 해당 출결 행(11시/16시/19시)에 업로드까지 해야 했다</p>
<p><strong>매일 반복되는 화면캡쳐 자동화하면 되지 않을까? 라는 생각이 들었다</strong>
왜냐하면.. 질의 받다가도 중간출결 사진촬영을 하러 떠나야하고
과제 피드백 및 pr을 보다가도 사진촬영하러 떠나야해서 너무 흐름이 끊겼다
<del>사실 너무 귀찮았어요...</del></p>
<h2 id="전체-아키텍처">전체 아키텍처</h2>
<p>최종적으로 완성된 자동화 구조는 이렇다:</p>
<pre><code>launchd (스케줄러)
  → ZepCapture.app (Automator 앱)
    → zep_capture.sh (캡처 스크립트)
      → screencapture (macOS 내장 명령어)
      → upload_to_airtable.py (에어테이블 업로드)</code></pre><p>3개의 파일이 역할을 나눠서 동작한다:</p>
<table>
<thead>
<tr>
<th>파일</th>
<th>역할</th>
</tr>
</thead>
<tbody><tr>
<td><code>zep_capture.sh</code></td>
<td>화면 캡처 실행 + 시간 판별 후 업로드 트리거</td>
</tr>
<tr>
<td><code>upload_to_airtable.py</code></td>
<td>에어테이블 API로 레코드 검색 + 첨부파일 업로드</td>
</tr>
<tr>
<td><code>com.zep.capture.plist</code></td>
<td>launchd 스케줄 설정 (언제 실행할지)</td>
</tr>
<tr>
<td><code>ZepCapture.app</code></td>
<td>Automator 앱 래퍼 (화면 기록 권한용)</td>
</tr>
</tbody></table>
<hr />
<h2 id="screencapture로-전체화면-캡처">screencapture로 전체화면 캡처</h2>
<p>이번 자동화 프로그램을 만들면서 처음 알았던 사실!!
macOS에는 <code>screencapture</code>라는 내장 명령어가 있다!</p>
<pre><code class="language-bash"># 기본 전체화면 캡처
screencapture -x ~/Desktop/capture.png

# 듀얼 모니터 사용중이라 디스플레이 별 캡쳐
screencapture -C -x -D 1 ~/Desktop/display1.png
screencapture -C -x -D 2 ~/Desktop/display2.png</code></pre>
<p>주요 옵션:</p>
<ul>
<li><code>-x</code>: 캡처 사운드 끄기</li>
<li><code>-C</code>: 커서 포함 + <strong>앱 창 포함</strong></li>
<li><code>-D 1</code>: 디스플레이 번호 지정</li>
</ul>
<h3 id="듀얼-모니터-이슈">듀얼 모니터 이슈</h3>
<p>싱글 모니터에서는 <code>screencapture -x</code>만으로 충분하지만 <strong>듀얼 모니터 환경</strong>에서는 메인 화면이 안 잡히는 경우가 있었다
파일명을 2개 넘기면 각 디스플레이별로 캡처된다:</p>
<pre><code class="language-bash">screencapture -C -x -D 1 &quot;display1.png&quot; -D 2 &quot;display2.png&quot;</code></pre>
<hr />
<h2 id="launchd로-스케줄링">launchd로 스케줄링</h2>
<p>macOS에서 cron 대신 <strong>Apple이 공식 권장하는 스케줄러</strong>가 launchd다 <code>~/Library/LaunchAgents/</code>에 plist 파일을 넣으면 특정 시간에 자동 실행된다</p>
<pre><code class="language-xml">&lt;!-- com.zep.capture.plist --&gt;
&lt;key&gt;StartCalendarInterval&lt;/key&gt;
&lt;array&gt;
    &lt;!-- 10:00 --&gt;
    &lt;dict&gt;
        &lt;key&gt;Hour&lt;/key&gt;&lt;integer&gt;10&lt;/integer&gt;
        &lt;key&gt;Minute&lt;/key&gt;&lt;integer&gt;0&lt;/integer&gt;
    &lt;/dict&gt;
    &lt;!-- 10:09 --&gt;
    &lt;dict&gt;
        &lt;key&gt;Hour&lt;/key&gt;&lt;integer&gt;10&lt;/integer&gt;
        &lt;key&gt;Minute&lt;/key&gt;&lt;integer&gt;9&lt;/integer&gt;
    &lt;/dict&gt;
    &lt;!-- ... 11:00, 16:00, 18:40 --&gt;
&lt;/array&gt;</code></pre>
<h3 id="등록해제-명령어">등록/해제 명령어</h3>
<pre><code class="language-bash"># 등록
launchctl load ~/Library/LaunchAgents/com.zep.capture.plist

# 해제
launchctl unload ~/Library/LaunchAgents/com.zep.capture.plist

# 수동 실행 (테스트용)
launchctl start com.zep.capture

# 상태 확인
launchctl list | grep zep</code></pre>
<hr />
<h2 id="macos-보안과의-사투-가장-삽질한-부분">macOS 보안과의 사투 (가장 삽질한 부분)</h2>
<p>여기서 가장 큰 삽질이 있었는데</p>
<h3 id="바탕화면만-캡처되는-문제">바탕화면만 캡처되는 문제</h3>
<p>터미널에서 직접 <code>~/zep_capture.sh</code>를 실행하면 크롬 창까지 정상 캡처되는데 <code>launchctl start</code>로 실행하면 <strong>바탕화면만</strong> 찍혔다</p>
<h3 id="macos의-앱-단위-화면-기록-권한을-관리하는-원인">macOS의 앱 단위 화면 기록 권한을 관리하는 원인</h3>
<p>macOS는 <strong>앱 단위로</strong> 화면 기록 권한을 관리한다..!(이번에 처음 안 사실)</p>
<ul>
<li><code>~/zep_capture.sh</code> 직접 실행 → <strong>Terminal.app</strong>이 실행 주체 → Terminal에 화면 기록 권한 있음 → 정상</li>
<li><code>launchctl start</code> → <strong>launchd</strong>가 실행 주체 → launchd에는 화면 기록 권한 없음 → 바탕화면만 캡쳐됨</li>
</ul>
<h3 id="automator-앱으로-감싸서-해결"><strong>Automator 앱으로 감싸서 해결!!</strong></h3>
<p>쉘 스크립트를 <strong>Automator 앱(.app)</strong>으로 감싸고 이 앱에 화면 기록 권한을 부여하면 된다!!
<img alt="업로드중.." src="blob:https://velog.io/3d26278a-2acc-46e7-96bc-786d7585f65b" /></p>
<pre><code>launchd → open ZepCapture.app → Automator가 zep_capture.sh 호출 → 정상 캡처</code></pre><p><strong>Automator 앱 만드는 법:</strong></p>
<ol>
<li>Automator 열기 → <strong>응용 프로그램</strong> 선택</li>
<li>&quot;셸 스크립트 실행&quot; 액션 추가</li>
<li><code>/bin/bash /Users/사용자명/zep_capture.sh</code> 입력</li>
<li>저장</li>
</ol>
<blockquote>
<p>시스템 설정 → 개인정보 보호 및 보안 → 화면 기록 → ZepCapture 앱 허용</p>
</blockquote>
<p>이 권한 설정 안 하면 전부 검은 화면으로 캡처된다ㅠㅠ</p>
<hr />
<h2 id="에어테이블-api-연동">에어테이블 API 연동</h2>
<p>캡처까지 자동화했으니 이제 에어테이블 업로드도 자동화할 차례다</p>
<h3 id="에어테이블-api">에어테이블 API</h3>
<ol>
<li><strong>Personal Access Token (PAT)</strong>: Builder Hub → Personal Access Tokens → Create token</li>
<li><strong>Base ID</strong>: 에어테이블 URL에서 <code>app...</code> 부분</li>
<li><strong>Table ID</strong>: URL에서 <code>tbl...</code> 부분</li>
<li><strong>필드 ID</strong>: 메타 API로 조회</li>
</ol>
<pre><code class="language-bash"># 필드 ID 조회
curl &quot;https://api.airtable.com/v0/meta/bases/{BASE_ID}/tables&quot; \
  -H &quot;Authorization: Bearer {PAT}&quot;</code></pre>
<h3 id="pyairtable-라이브러리-사용">pyairtable 라이브러리 사용</h3>
<p>처음에는 <code>content.airtable.com</code>의 Upload Attachment API를 직접 호출했는데 <code>400 Bad Request</code>가 계속 발생했다. <strong>pyairtable</strong> 라이브러리를 쓰면 알아서 처리해준다 
굿!</p>
<pre><code class="language-bash">pip3 install pyairtable --break-system-packages</code></pre>
<pre><code class="language-python">from pyairtable import Api

api = Api(&quot;pat토큰&quot;)
table = api.table(&quot;appXXX&quot;, &quot;tblXXX&quot;)

# 레코드 검색 (날짜 + 기수 + 출결시간)
formula = &quot;AND(IS_SAME({날짜}, '2026-03-17', 'day'), {기수}='A캠프 n기', {중간 출결}='11시')&quot;
records = table.all(formula=formula, max_records=1)

# 첨부파일 업로드
record_id = records[0][&quot;id&quot;]
table.upload_attachment(record_id, &quot;fldXXX&quot;, &quot;/path/to/screenshot.png&quot;)</code></pre>
<h3 id="날짜-형식-맞추기">날짜 형식 맞추기</h3>
<p>에어테이블 Date 필드는 화면에 <code>2026.3.16</code>으로 보이지만, API로 조회하면 <strong>ISO 형식</strong>(<code>2026-03-16T00:00:00.000Z</code>)으로 저장돼 있다</p>
<p>처음에 <code>FIND('2026.3.16', {날짜})</code>로 검색해서 레코드를 못 찾았는데 <code>IS_SAME({날짜}, '2026-03-16', 'day')</code>로 바꾸니 해결됐다!</p>
<hr />
<h2 id="시간별-업로드-분기">시간별 업로드 분기</h2>
<p>모든 캡처를 에어테이블에 올리는 게 아니라 <strong>특정 시간 캡처만</strong> 해당 출결 행에 올려야 했다</p>
<pre><code class="language-bash">CURRENT_HOUR=$(date '+%H')
CURRENT_MIN=$(date '+%M')

CHULGYUL=&quot;&quot;

# 10:09 캡처 → 11시 행
if [ &quot;$CURRENT_HOUR&quot; = &quot;10&quot; ] &amp;&amp; [ &quot;$CURRENT_MIN&quot; -ge &quot;05&quot; ] &amp;&amp; [ &quot;$CURRENT_MIN&quot; -le &quot;15&quot; ]; then
    CHULGYUL=&quot;11시&quot;
fi

# 16:00 캡처 → 16시 행
if [ &quot;$CURRENT_HOUR&quot; = &quot;16&quot; ] &amp;&amp; [ &quot;$CURRENT_MIN&quot; -ge &quot;00&quot; ] &amp;&amp; [ &quot;$CURRENT_MIN&quot; -le &quot;10&quot; ]; then
    CHULGYUL=&quot;16시&quot;
fi

# 18:40 캡처 → 19시 행
if [ &quot;$CURRENT_HOUR&quot; = &quot;18&quot; ] &amp;&amp; [ &quot;$CURRENT_MIN&quot; -ge &quot;35&quot; ] &amp;&amp; [ &quot;$CURRENT_MIN&quot; -le &quot;45&quot; ]; then
    CHULGYUL=&quot;19시&quot;
fi

if [ -n &quot;$CHULGYUL&quot; ]; then
    /opt/homebrew/bin/python3 ~/upload_to_airtable.py &quot;$CHULGYUL&quot; display1.png display2.png
fi</code></pre>
<p>시간에 약간의 여유 범위를 둬서 (±5분) launchd 실행이 살짝 지연돼도 정상 동작하도록 했다</p>
<hr />
<h2 id="다중-기수-지원">다중 기수 지원</h2>
<p>프로젝트 기간이어서 내가 담당하는 과정 뿐만아니라 다른 과정 캠프도 같은 캡처를 올려야 했다!!
기수를 리스트로 관리하면 간단하다:</p>
<pre><code class="language-python">GISU_LIST = [&quot;B캠프 n기&quot;, &quot;A캠프 m기&quot;]

for gisu in GISU_LIST:
    formula = f&quot;AND(IS_SAME({{날짜}}, '{date_str}', 'day'), {{기수}}='{gisu}', {{중간 출결}}='{chulgyul}')&quot;
    records = table.all(formula=formula, max_records=1)

    if records:
        table.upload_attachment(records[0][&quot;id&quot;], FIELD_ID, file_path)</code></pre>
<p>기수가 추가되면 리스트에 한 줄만 추가하면 된다!</p>
<hr />
<h2 id="최종-결과">최종 결과</h2>
<h3 id="before">Before</h3>
<ol>
<li>시간 맞춰서 수동 캡처 (하루 5번)</li>
<li>캡처 파일 찾기</li>
<li>에어테이블 열기</li>
<li>해당 행 찾기</li>
<li>파일 드래그 앤 드롭</li>
<li>각각 반복</li>
</ol>
<p>→ <strong>하루 약 10~15분 소요</strong></p>
<h3 id="after">After</h3>
<ol>
<li>아무것도 안 함</li>
<li>(맥북이 깨어있기만 하면 됨)</li>
</ol>
<p>→ <strong>소요 시간: 0분</strong></p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/b5cf6c7e-b91a-4615-877f-58552ebdd552/image.png" /></p>
<h1 id="잘-올라오는거-확인했다">잘 올라오는거 확인했다!!</h1>
<p><em>보안때문에 일단 가리고 보기</em></p>
<hr />
<h2 id="팀원-공유">팀원 공유</h2>
<p>이 자동화를 팀원에게도 공유하기 위해 <strong>설치 스크립트 1개</strong>로 패키징했다. 팀원은 터미널에서 한 줄만 실행하면 된다:</p>
<pre><code class="language-bash">bash install_zep_capture.sh</code></pre>
<p>스크립트가 <code>whoami</code>로 사용자명을 자동 감지하고 캡처 스크립트 + Automator 앱 + launchd 스케줄을 전부 자동 생성한다!!</p>
<hr />
<h2 id="후기라면-후기랄까">후기라면 후기랄까</h2>
<p><del>귀찮다</del>는 생각에서 시작한 자동화 작업!
macOS의 보안 모델(앱 단위 화면 기록 권한)이 있을거라고 당연히 생각은 했었지만 모르고 살다가 이렇게 마주치니까 신기했다
해결하는 과정에서 Automator라는 우회 경로를 찾았고 (맥북 10년차 이 어플이 있는것도 이번에 알았음 ㄷㄷ)
에어테이블 API의 날짜 형식 차이나 Python 경로 문제 같은 소소한 삽질도 하나씩 해결해나갔다!!</p>
<p>결과적으로 하루 15~20분의 반복 작업을 완전히 없앴다
그리고 무엇보다도 중출 시간마다 팀원분들이 너무 편하다고 피드백 주셔서 뿌듯했다🤭</p>
<p><strong>반복적인 일은 기계에게 맡기자!!!!!!</strong></p>
<h3 id="사용한-기술-스택">사용한 기술 스택</h3>
<ul>
<li>macOS <code>screencapture</code> (화면 캡처)</li>
<li>launchd + plist (스케줄링)</li>
<li>Automator.app (화면 기록 권한 우회)</li>
<li>Python + pyairtable (에어테이블 API)</li>
<li>Airtable REST API + filterByFormula</li>
</ul>
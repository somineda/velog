<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/25d202ca-4941-4c47-b096-90eb79a96f77/image.jpg" /></p>
<h2 id="도커란">도커란????????????</h2>
<p>도커는 컨테이너 기술을 기반으로 한 일종의 가상화 플랫폼입니다.
<strong>가상화</strong>란 물리적 자원인 하드웨어를 효율적으로 활용하기 위해서 하드웨어 공간 위에 가상의 머신을 만드는 기술
<strong>컨테이너</strong>란 컨테이너가 실행되고 있는 호스트 os의 기능을 그대로 사용하면서 프로세스를 격리해 독립된 환경을 만드는 기술</p>
<p>즉</p>
<h4 id="독립된-환경에-애플리케이션과-그가-동작하기-위한-모든-것을-묶어두기-위해-사용되는-기술">독립된 환경에 애플리케이션과 그가 동작하기 위한 모든 것을 묶어두기 위해 사용되는 기술</h4>
<hr />
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/aba1a69a-ace7-42ea-a869-aad3dcaf5423/image.jpeg" /></p>
<h2 id="그럼-왜-도커를-사용하는걸까">그럼 왜 도커를 사용하는걸까?</h2>
<blockquote>
<p>“내 컴퓨터에서는 잘 되는데?”<br />이 문제를 해결하기 위해 등장한 것이 바로 <strong>도커(Docker)</strong>입니다</p>
</blockquote>
<h3 id="💡-1-환경-차이로-인한-문제-해결">💡 1. 환경 차이로 인한 문제 해결</h3>
<p>도커의 가장 큰 장점은 <strong>‘환경 일관성’</strong>입니다 
보통 개발할 때 이런 일이 자주 생기죠:</p>
<ul>
<li>내 컴퓨터에서는 잘 되는데 서버에서는 에러가 남..</li>
<li>팀원마다 설치된 라이브러리 버전이 달라서 충돌 발생.. </li>
<li>운영체제(OS)에 따라 실행 결과가 달라짐..</li>
</ul>
<p>도커는 이런 문제를 근본적으로 해결합니다.<br />왜냐하면 애플리케이션과 실행 환경(라이브러리, OS 설정 등)을<br /><strong>컨테이너 안에 그대로 묶어서 배포</strong>하기 때문입니다</p>
<h3 id="⚙️-2-빠른-배포와-확장성">⚙️ 2. 빠른 배포와 확장성</h3>
<p>도커 컨테이너는 <strong>가볍고 빠르게 실행</strong>됩니다.<br />OS 전체를 복사하지 않기 때문에 한 줄 명령어로 서버에 즉시 배포 가능합니다. </p>
<p>예를 들어 웹 서비스를 운영할 때 이 한줄이면
바로 배포 완료됩니다.</p>
<pre><code class="language-bash">docker run -d -p 80:80 myapp </code></pre>
<h2 id="📦-컨테이너란">📦 컨테이너란?</h2>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/b55e3f50-2312-4a39-bcb2-61e6d1abe058/image.png" /></p>
<h3 id="💡-1-컨테이너의-정의">💡 1. 컨테이너의 정의</h3>
<p><strong>컨테이너(Container)</strong>란  
애플리케이션이 실행되는 데 필요한 <strong>모든 환경(코드, 라이브러리, 설정 등)</strong>을 하나로 묶은<br /><strong>독립된 실행 단위</strong></p>
<blockquote>
<p>“개발자가 만든 프로그램을 어떤 환경에서도 동일하게 실행할 수 있도록 하는 기술”  </p>
</blockquote>
<h3 id="⚙️-2-컨테이너의-핵심-특징">⚙️ 2. 컨테이너의 핵심 특징</h3>
<table>
<thead>
<tr>
<th align="center">특징</th>
<th align="left">설명</th>
</tr>
</thead>
<tbody><tr>
<td align="center"><strong>격리성(Isolation)</strong></td>
<td align="left">각 컨테이너는 독립된 공간에서 실행되어 서로 영향을 주지 않음</td>
</tr>
<tr>
<td align="center"><strong>경량성(Lightweight)</strong></td>
<td align="left">전체 운영체제를 복제하지 않고, 호스트 OS의 커널을 공유하여 빠르고 가벼움</td>
</tr>
<tr>
<td align="center"><strong>이식성(Portability)</strong></td>
<td align="left">어디서 실행해도 동일한 결과를 보장 (개발 → 테스트 → 운영 동일 환경)</td>
</tr>
<tr>
<td align="center"><strong>일관성(Consistency)</strong></td>
<td align="left">개발자가 만든 환경을 그대로 서버에 배포 가능</td>
</tr>
<tr>
<td align="center"><strong>신속성(Speed)</strong></td>
<td align="left">컨테이너 생성 및 실행 속도가 매우 빠름</td>
</tr>
</tbody></table>
<h3 id="🚀-4-도커와-컨테이너의-관계">🚀 4. 도커와 컨테이너의 관계</h3>
<ul>
<li><strong>Docker</strong>는 <strong>컨테이너를 쉽게 만들고 관리할 수 있게 해주는 플랫폼</strong>입니다.  </li>
<li>직접 컨테이너를 띄우는 대신, Docker 명령어로 간단히 실행할 수 있습니다.</li>
</ul>
<h3 id="✅-5-컨테이너-vs-가상머신vm">✅ 5. 컨테이너 vs 가상머신(VM)</h3>
<table>
<thead>
<tr>
<th align="center">구분</th>
<th align="left">컨테이너(Container)</th>
<th align="left">가상머신(Virtual Machine)</th>
</tr>
</thead>
<tbody><tr>
<td align="center"><strong>운영 구조</strong></td>
<td align="left">호스트 OS 위에서 커널 공유</td>
<td align="left">하이퍼바이저 위에 게스트 OS 별도 실행</td>
</tr>
<tr>
<td align="center"><strong>속도</strong></td>
<td align="left">빠름 (초 단위 실행)</td>
<td align="left">느림 (분 단위 부팅)</td>
</tr>
<tr>
<td align="center"><strong>자원 사용량</strong></td>
<td align="left">효율적 (필요한 부분만 사용)</td>
<td align="left">비효율적 (OS별 자원 소모)</td>
</tr>
<tr>
<td align="center"><strong>대표 기술</strong></td>
<td align="left">Docker, Podman</td>
<td align="left">VMware, VirtualBox</td>
</tr>
</tbody></table>
<blockquote>
<p>컨테이너는 OS를 공유하므로 <strong>가볍고 빠르며</strong>
여러 개의 앱을 동시에 돌리기에도 유리함</p>
</blockquote>
<p>그렇다면...</p>
<p>다음 글에서 도커 이미지에 대해 정리해보겠습니다!</p>
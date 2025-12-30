<h2 id="ip-internet-protocol의-개념">IP (Internet Protocol)의 개념</h2>
<ul>
<li>인터넷에 연결되어 있는 모든 장치들(컴퓨터, 서버 장비, 스마트폰 등)을 식별할 수 있도록 각각의 장비에게 부여되는 고유 주소</li>
<li>데이터를 패킷 단위로 나누어 전송하고, 받는 쪽에서는 그 패킷들을 다시 조립하여 원래의 데이터로 변환하는 과정을 거친다.</li>
<li>IP를 통해 전송되는 데이터는 인터넷 상의 다양한 기기들과 통신 가능
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/10fe8106-ed96-4466-ba1f-a3fcff386346/image.png" /><h2 id="ip의-동작-원리">IP의 동작 원리</h2>
IP를 이용한 데이터의 송, 수신은 다음의 절차를 통해서 이루어짐<h3 id="데이터-패킷화-data-packetization">데이터 패킷화 (Data Packetization)</h3>
</li>
<li><strong>목적</strong>: IP 프로토콜은 데이터를 네트워크 상에서 전달할 수 있는 크기의 패킷으로 분할합니다. 패킷화는 데이터를 효율적으로 전송하고, 전송 중 오류가 발생한 경우에도 해당 부분만 재전송할 수 있도록 합니다.</li>
<li><strong>과정</strong>:<ol>
<li><strong>데이터 분할</strong>: 큰 데이터를 작은 단위인 패킷으로 나눕니다. 각 패킷은 고유한 헤더를 포함하며 이 헤더에는 출발지와 목적지의 IP 주소, 시퀀스 번호, 데이터 길이 등의 정보가 담깁니다.</li>
<li><strong>헤더 정보 추가</strong>: 각 패킷에 IP 헤더가 추가되어 패킷이 네트워크를 통해 올바르게 전달될 수 있도록 정보를 제공합니다. 이 헤더는 목적지 주소, 출발지 주소, 패킷의 길이, 패킷 번호, 패킷의 순서 등이 포함됩니다.<ul>
<li><strong>예시</strong>:</li>
</ul>
</li>
</ol>
<ul>
<li>&quot;Hello, World!&quot;라는 데이터를 전송한다고 할 때, 이 데이터는 크기가 작지만 큰 파일이나 메시지를 전송할 경우 여러 개의 패킷으로 나누어집니다. 패킷화된 데이터는 각각의 패킷에 &quot;Hello&quot;, &quot;World!&quot;와 같은 데이터 조각을 담고 각 패킷은 고유한 번호를 가집니다.<h3 id="라우팅-routing">라우팅 (Routing)</h3>
</li>
</ul>
</li>
<li><strong>목적</strong>: IP 프로토콜은 데이터가 출발지에서 목적지까지 최적의 경로를 통해 전달될 수 있도록 하는 역할을 합니다. 이를 위해 네트워크 상의 라우터들이 패킷을 전달합니다.</li>
<li><strong>과정</strong>:<ol>
<li><strong>라우터의 역할</strong>: 라우터는 네트워크 간의 중계 장치로 패킷의 목적지 주소를 기반으로 다음 목적지를 결정합니다. 라우터는 목적지에 대한 정보를 바탕으로 패킷이 어디로 가야 하는지를 판단하여 해당 패킷을 최적의 경로로 전달합니다.</li>
<li><strong>라우팅 테이블</strong>: 각 라우터는 라우팅 테이블을 사용하여 목적지 주소에 대한 정보를 가지고 있습니다. 라우팅 테이블에는 네트워크 경로 정보가 저장되어 있으며 이 정보를 통해 라우터는 각 패킷을 가장 빠르고 효율적인 경로로 전송합니다.</li>
<li><strong>경로 변경</strong>: 네트워크 트래픽이나 장애로 인해 경로가 변경될 수 있습니다. 이 경우 라우터는 새로운 경로를 자동으로 학습하고 패킷을 해당 경로로 전송합니다.</li>
</ol>
</li>
<li><strong>예시</strong>:<ul>
<li>클라이언트가 서버에 데이터를 요청하면 클라이언트는 라우터를 통해 데이터를 전송합니다. 이 데이터는 여러 라우터를 거쳐서 최종 목적지인 서버로 전달됩니다. 이 과정에서 라우터들은 목적지 주소를 확인하고 적절한 경로를 선택하여 데이터를 전송합니다.<h3 id="전송-후-재조립-reassembly-after-transmission">전송 후 재조립 (Reassembly after Transmission)</h3>
</li>
</ul>
</li>
<li><strong>목적</strong>: 데이터가 여러 개의 패킷으로 나누어 전송되므로 목적지에서 이들을 원래의 데이터로 재조립하여 사용자에게 전달할 수 있도록 해야 합니다.</li>
<li><strong>과정</strong>:<ol>
<li><strong>패킷 수신</strong>: 목적지에 도착한 각 패킷은 IP 프로토콜을 통해 수신됩니다. 이때 각 패킷은 해당하는 시퀀스 번호를 포함하고 있습니다.</li>
<li><strong>패킷 재조립</strong>: 수신된 패킷들은 각 패킷의 시퀀스 번호를 기준으로 순서대로 정렬됩니다. 그런 다음 원래의 데이터 형태로 재조립됩니다. 예를 들어 &quot;Hello&quot;와 &quot;World!&quot;라는 두 개의 패킷이 있을 경우 패킷을 받은 시스템은 이를 하나의 &quot;Hello, World!&quot; 메시지로 합칩니다.</li>
<li><strong>오류 처리</strong>: 만약 일부 패킷이 손실되거나 순서가 어긋난 경우 수신 시스템은 이를 감지하고 손실된 패킷을 다시 요청할 수 있습니다. 이 과정은 상위 계층의 프로토콜(예: TCP)에 의해 처리됩니다.</li>
</ol>
</li>
<li><strong>예시</strong>:<ul>
<li>만약 &quot;Hello, World!&quot;라는 메시지가 2개의 패킷으로 나누어져 전송되었다면 수신자는 각 패킷의 시퀀스 번호를 기준으로 &quot;Hello&quot;와 &quot;World!&quot;를 합쳐서 원래의 메시지로 복원합니다. 만약 패킷 중 하나가 손실되었으면 해당 패킷에 대한 요청을 보내고 수신된 패킷들을 적절히 재조립하여 완전한 메시지를 복원합니다.<h2 id="ip의-종류">IP의 종류</h2>
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/a8d789cd-7ae8-4bbc-9f7b-289fdb3dfb3d/image.png" /><h3 id="ipv4">IPv4</h3>
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/96f46511-1a79-450f-82b0-0f36372b3ff8/image.png" /><h4 id="--ipv4는-ip-version-4의-약자로-전-세계적으로-사용된-첫-번째-인터넷-프로토콜">- IPv4는 IP version 4의 약자로 전 세계적으로 사용된 첫 번째 인터넷 프로토콜</h4>
<h4 id="--아이피는32bit로-이루어진-주소이며-이를-합산해보면-약43억개의-주소를-가지게-됨">- 아이피는32bit로 이루어진 주소이며 이를 합산해보면 약43억개의 주소를 가지게 됨</h4>
</li>
</ul>
</li>
</ul>
<h4 id="특징"><strong>특징</strong></h4>
<ul>
<li>주소체계는 12자리로, 네 부분으로 나뉘어져 있음</li>
<li>각 부분은 0~255까지 3자리의 수로 표현</li>
<li>점으로 구분된 10진수 표기법을 사용</li>
<li>네트워크 접두사와 호스트 번호로 구성</li>
<li>단일 네트워크 내의 모든 호스트는 동일한 네트워크 주소를 공유</li>
</ul>
<h3 id="ipv6">IPv6</h3>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/b69e3d17-2ee7-411e-b24b-2d8b6ade616e/image.png" /></p>
<h4 id="--ip-주소-지정-시스템의-버전으로-기존의-ipv4-주소가-고갈되는-문제를-해결하기-위해-개발">- IP 주소 지정 시스템의 버전으로 기존의 IPv4 주소가 고갈되는 문제를 해결하기 위해 개발</h4>
<h4 id="특징-1"><strong>특징</strong></h4>
<ul>
<li>IPv6 주소는 128비트로 표현되며 16비트 단위로 구분하여 16진수로 변환한 후 콜론(:)으로 구분하여 표기</li>
<li>IPv6 주소공간은 128비트로 표현할 수 있는 2128개로 약 3.4x1038개 의 주소를 갖고 있어 거의 무한대로 사용 가능
⇒ 340,282,366,920,938,463,463,374,607,431,768,211,456개</li>
<li>IPv6는 네트워크 보안을 위해 다양한 확장 헤더를 제공</li>
<li>IPv6는 IPv4와의 하위 호환성을 제공하지 않음</li>
<li>아직 완전히 상용화 되지는 않음</li>
</ul>
<h3 id="🌐-ipv4와-ipv6-비교">🌐 IPv4와 IPv6 비교</h3>
<table>
<thead>
<tr>
<th align="center"><strong>항목</strong></th>
<th align="center"><strong>IPv4</strong></th>
<th align="center"><strong>IPv6</strong></th>
</tr>
</thead>
<tbody><tr>
<td align="center"><strong>주소 길이</strong></td>
<td align="center">32비트</td>
<td align="center">128비트</td>
</tr>
<tr>
<td align="center"><strong>표현 방식</strong></td>
<td align="center">10진수 (예: <code>192.168.0.1</code>)</td>
<td align="center">16진수 (예: <code>2001:db8::1</code>)</td>
</tr>
<tr>
<td align="center"><strong>주소 개수</strong></td>
<td align="center">약 43억 개</td>
<td align="center">사실상 무제한</td>
</tr>
<tr>
<td align="center"><strong>헤더 크기</strong></td>
<td align="center">20바이트</td>
<td align="center">40바이트</td>
</tr>
<tr>
<td align="center"><strong>지원 기술</strong></td>
<td align="center">NAT 사용</td>
<td align="center">NAT 불필요 (직접 통신 가능)</td>
</tr>
<tr>
<td align="center"><strong>주요 목적</strong></td>
<td align="center">기존 인터넷 통신</td>
<td align="center">차세대 인터넷 및 사물인터넷(IoT) 통신 지원</td>
</tr>
</tbody></table>
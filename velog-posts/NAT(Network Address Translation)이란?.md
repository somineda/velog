<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/42f6ee36-fb2e-412d-b3e6-0f278345bf8e/image.png" />
NAT(Network Address Translation)는 네트워크 장비(라우터나 방화벽 등)가 사설 네트워크의 IP 주소를 공인 IP 주소로 변환하거나 그 반대로 변환하는 기술
여러 장치가 하나의 공인 IP 주소를 공유할 수 있으며 보안과 IP 주소의 효율적인 사용이 가능하다</p>
<h3 id="nat을-사용하는-이유">NAT을 사용하는 이유</h3>
<ul>
<li>공인 IPv4주소를 절약하기 위해서<ul>
<li>IPv4 주소의 한정된 개수를 효과적으로 활용</li>
<li>여러 기기가 하나의 공인 IP 주소를 공유하도록 함</li>
</ul>
</li>
<li>보안의 목적<ul>
<li>내부 네트워크 구조가 외부에 노출되지 않음</li>
<li>외부에서 내부 기기에 직접 접근하는 것을 방지</li>
</ul>
</li>
</ul>
<h3 id="nat의-작동-원리">NAT의 작동 원리</h3>
<ul>
<li><strong>송신 트래픽</strong><ul>
<li>내부 네트워크의 장치가 데이터를 외부로 보낼 때, NAT 장비가 사설 IP 주소를 공인 IP 주소로 변환</li>
<li>패킷의 출발지 IP 주소가 변경됨</li>
</ul>
</li>
<li><strong>수신 트래픽</strong><ul>
<li>외부에서 내부 네트워크로 데이터가 들어올 때 NAT 장비가 공인 IP 주소를 사설 IP 주소로 변환</li>
<li>패킷의 목적지 IP 주소가 변경됨</li>
</ul>
</li>
</ul>
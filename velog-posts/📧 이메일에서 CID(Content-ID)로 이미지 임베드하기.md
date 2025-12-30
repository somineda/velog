<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/b1f25061-8239-4eec-8684-f48b0119b0bb/image.jpg" /></p>
<p>새해 기념 새해 다짐 프로젝트를 만들다가 그냥 텍스트만 있는 메일이 아니라
이메일을 들어갔을 때 새해 다짐과 관련된 이미지가 바로 보였으면 좋겠어서
처음엔 단순하게 HTML 메일에 이미지 URL을 넣으면 될 거라고 생각했던 나자신..</p>
<p>전혀 아니었음
자꾸 첨부파일로 보내지고 본문 안에서는 아예 이미지가 안보였다!!
내가 어떻게 고른 새해 다짐 무도짤인데!!!!!!!!!!!!!!!!!!!!!</p>
<p>그러다 찾은 방법 </p>
<h1 id="cid"><strong>CID</strong></h1>
<h2 id="🔍-cid-란">🔍 CID 란?</h2>
<blockquote>
<p>CID(Content-ID) 는 이메일 내부에서 이미지를 식별하기 위한 고유 ID</p>
</blockquote>
<p>즉</p>
<p>이메일 내부에 이미지를 포함시키고
HTML 본문에서 내부 ID로 이미지를 참조하는 방식이다
-&gt; HTML 본문에서 이 ID를 참조하면 이메일에 포함된 이미지를 직접 렌더링 가능</p>
<h2 id="⚙️-동작-원리">⚙️ 동작 원리</h2>
<ol>
<li><p>이메일에 이미지를 첨부</p>
</li>
<li><p>이미지에 CID 부여</p>
</li>
<li><p>HTML 본문에서 cid:ID 로 참조</p>
</li>
<li><p>이메일 클라이언트가 내부 이미지로 렌더링</p>
</li>
</ol>
<p>나같은 경우에는 </p>
<pre><code>    if has_image:
        try:
            with open(image_path, 'rb') as img_file:
                img_data = img_file.read()
                image = MIMEImage(img_data)
                image.add_header('Content-ID', f'&lt;{image_cid}&gt;')
                image.add_header('Content-Disposition', 'inline', filename=os.path.basename(image_path))
                message.attach(image)
                print(f&quot;Embedded image: {os.path.basename(image_path)}&quot;)
        except Exception as e:
            print(f&quot;Failed to embed image: {e}&quot;)</code></pre><p>email.mime 모듈을 사용해서 이미지에 Content-ID를 부여 했다
그리고</p>
<pre><code> &lt;img src=&quot;cid:{{ image_cid }}&quot; alt=&quot;응원 이미지&quot; /&gt;</code></pre><p>HTML 본문에서 이미지 사용</p>
<p>이렇게 하면
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/bb84cb66-08b9-4f12-9bcb-4c3eda5b8a26/image.jpg" /></p>
<p>📨 이렇게 이메일 본문에 이미지가 바로 표시된다!!</p>
<p>프로젝트를 진행하다 보면 프로필 이미지 등록은 거의 필수로 들어가게 된다
프로필 이미지 뿐만 아니라 게시글에 첨부 이미지 등등등 많이 필요함!</p>
<p>이번 LMS 프로젝트 역시도 내가 맡은 파트가 유저쪽이어서 사용자가 프로필 이미지를 등록하는 기능이 필수였다!!!!</p>
<blockquote>
<p>Django + DRF 환경에서 <strong>AWS S3 Presigned URL</strong>을 활용하여 이미지 업로드를 구현하는 방법
지금 프로젝트를 하면서 겪은 처음 구현부터 리팩토링 과정까지 적어보겠다!                                                   </p>
</blockquote>
<h2 id="1-서버-직접-업로드-vs-presigned-url">1. 서버 직접 업로드 vs Presigned URL</h2>
<h3 id="1-1-서버-직접-업로드-방식">1-1. 서버 직접 업로드 방식</h3>
<p>  기존에 많이 사용하는 방식은 클라이언트가 이미지를 서버로 전송하고 서버가 S3에 업로드하는 방식                                          </p>
<pre><code>  Client → [이미지 파일] → Django Server → AWS S3   </code></pre><pre><code class="language-python">  # 서버 직접 업로드 예시                                                                                                                                                                                                                                                                                       
  def update_profile_image(*, user: User, image: UploadedFile) -&gt; User:                                                                                                                                                                                                                                         
      s3 = S3Client()                                                                                                                                                                                                                                                                                           

      # 기존 이미지 삭제                                                                                                                                                                                                                                                                                        
      if user.profile_img_url:                                                                                                                                                                                                                                                                                  
          s3.delete_by_url(user.profile_img_url)                                                                                                                                                                                                                                                                

      # 새 이미지 업로드 (서버 → S3)                                                                                                                                                                                                                                                                            
      key = s3.upload(image, path_prefix=&quot;profile&quot;)                                                                                                                                                                                                                                                             
      user.profile_img_url = s3.build_url(key)                                                                                                                                                                                                                                                                  
      user.save()                                                                                                                                                                                                                                                                                               
      return user                                                                                                                                                           </code></pre>
<p>**  장점:   **                                                                                                                                                                                                                                                                                                      </p>
<ul>
<li>구현이 단순함                                                                                                                                                                                                                                                                                               </li>
<li>서버에서 파일 검증 후 업로드 가능                                                                                                                                                                                                                                                                           </li>
</ul>
<p>**  단점:  **                                                                                                                                                                                                                                                                                                       </p>
<ul>
<li>대용량 파일 처리 시 서버부하 증가                                                                                                                                                                                                                                                                      </li>
<li>서버 대역폭 소모                                                                                                                                                                                                                                                                                            </li>
<li>업로드 시간 증가 (Client → Server → S3)                                                                                                                                                                                                                                                       <h3 id="1-2-presigned-url-방식">1-2. Presigned URL 방식</h3>
<img alt="" src="https://velog.velcdn.com/images/sommnie/post/19cd1b0e-8a84-4332-b08f-06bd2ad22038/image.png" />
Presigned URL은 AWS S3가 제공하는 임시 인증 URL
클라이언트가 이 URL을 사용하면 AWS 자격 증명 없이도 S3에 직접 파일을 업로드 가능                                                                                                                                                      </li>
</ul>
<ol>
<li>Client → Django (Presigned URL 요청)                                                                                                                                                                                                                                                                       </li>
<li>Django → Client (presigned_url, img_url 반환)                                                                                                                                                                                                                                                              </li>
<li>Client → AWS S3 (이미지 직접 업로드)                                                                                                                                                                                                                                                                       </li>
<li>Client → Django (img_url로 DB 업데이트 요청)                                                                                                                                                                                                                                                               </li>
</ol>
<p>**  장점:   **                                                                                                                                                                                                                                                                                                      </p>
<ul>
<li>서버 부하 감소                                                                                                                                                                                                                                                                                              </li>
<li>업로드 속도 향상 (클라이언트 → S3 직접 통신)                                                                                                                                                                                                                                                                </li>
<li>서버 대역폭 절약                                                                                                                                                                                                                                                                                            </li>
</ul>
<p>**  단점:      **                                                                                                                                                                                                                                                                                                   </p>
<ul>
<li><p>API가 2개 필요 (URL 발급 + DB 저장)                                                                                                                                                                                                                                                                         </p>
</li>
<li><p>클라이언트 구현이 조금 더 복잡                                       </p>
<blockquote>
<p>presigned url은 s3 bucket으로 접근 할 수 있게 해주는 엔드포인트!</p>
</blockquote>
<h2 id="2처음-구현---서버-직접-업로드-방식">2.처음 구현 - 서버 직접 업로드 방식</h2>
<h3 id="2-1-당시-요구사항">2-1. 당시 요구사항</h3>
</li>
<li><p>사용자가 프로필 이미지를 업로드하면 S3에 저장                                                                                                                                                                                                                                                               </p>
</li>
<li><p>기존 이미지가 있으면 삭제 후 새 이미지로 교체                                                                                                                                                                                                                                                               </p>
</li>
<li><p>허용 파일 형식: JPEG, PNG  </p>
</li>
</ul>
<p>이렇게  명세서에 적혀 있어서 초기 구조에서 유틸리티 클래스에 s3 관련 기능을 모아두었다</p>
<pre><code class="language-python">class S3Client:                                                                                                                                                                                                                                                                                               
      def __init__(self) -&gt; None:                                                                                                                                                                                                                                                                               
          self.s3 = boto3.client(                                                                                                                                                                                                                                                                               
              &quot;s3&quot;,                                                                                                                                                                                                                                                                                             
              aws_access_key_id=settings.AWS_S3_ACCESS_KEY_ID,                                                                                                                                                                                                                                                  
              aws_secret_access_key=settings.AWS_S3_SECRET_ACCESS_KEY,                                                                                                                                                                                                                                          
              region_name=settings.AWS_S3_REGION,                                                                                                                                                                                                                                                               
          )                                                                                                                                                                                                                                                                                                     
          self.bucket_name: str = settings.AWS_S3_BUCKET_NAME                                                                                                                                                                                                                                                   

      def upload(                                                                                                                                                                                                                                                                                               
          self,                                                                                                                                                                                                                                                                                                 
          file: Any,                                                                                                                                                                                                                                                                                            
          path_prefix: str = &quot;&quot;,                                                                                                                                                                                                                                                                                
          extra_args: dict[str, Any] | None = None,                                                                                                                                                                                                                                                             
      ) -&gt; str:                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           
          original_name = getattr(file, &quot;name&quot;, &quot;unknown_file&quot;)                                                                                                                                                                                                                                                 
          ext = original_name.split(&quot;.&quot;)[-1] if &quot;.&quot; in original_name else &quot;bin&quot;                                                                                                                                                                                                                                 
          file_name = f&quot;{uuid.uuid4()}.{ext}&quot;                                                                                                                                                                                                                                                                   

          clean_prefix = path_prefix.strip(&quot;/&quot;)                                                                                                                                                                                                                                                                 
          key = f&quot;{clean_prefix}/{file_name}&quot; if clean_prefix else file_name                                                                                                                                                                                                                                    

          upload_params: dict[str, Any] = extra_args.copy() if extra_args else {}                                                                                                                                                                                                                               

          if &quot;ContentType&quot; not in upload_params:                                                                                                                                                                                                                                                                
              content_type = getattr(file, &quot;content_type&quot;, None)                                                                                                                                                                                                                                                
              if content_type:                                                                                                                                                                                                                                                                                  
                  upload_params[&quot;ContentType&quot;] = content_type                                                                                                                                                                                                                                                   

          self.s3.upload_fileobj(file, self.bucket_name, key, ExtraArgs=upload_params)                                                                                                                                                                                                                          
          return key                                                                                                                                                                                                                                                                                            

      def delete(self, key: str) -&gt; None:                                                                                                                                                                                                                                                                       
          #S3 객체 삭제                                                                                                                                                                                                                                                                                    
          if not key:                                                                                                                                                                                                                                                                                           
              return                                                                                                                                                                                                                                                                                            
          try:                                                                                                                                                                                                                                                                                                  
              self.s3.delete_object(Bucket=self.bucket_name, Key=key)                                                                                                                                                                                                                                           
          except ClientError as e:                                                                                                                                                                                                                                                                              
              logger.warning(f&quot;S3 삭제 실패 (Key: {key}): {e}&quot;)                                                                                                                                                                                                                                                 

      def delete_by_url(self, url: str) -&gt; None:                                                                                                                                                                                                                                                                
          #URL로 S3 객체 삭제                                                                                                                                                                                                                                                                             
          key = self.extract_key_from_url(url)                                                                                                                                                                                                                                                                  
          if key:                                                                                                                                                                                                                                                                                               
              self.delete(key)                                                                                                                                                                                                                                                                                  

      def build_url(self, key: str) -&gt; str:                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          
          if not key:                                                                                                                                                                                                                                                                                           
              return &quot;&quot;                                                                                                                                                                                                                                                                                         
          domain = f&quot;{self.bucket_name}.s3.{settings.AWS_S3_REGION}.amazonaws.com&quot;                                                                                                                                                                                                                              
          return f&quot;https://{domain}/{key}&quot;                                                                                                                                                                                                                                                                      

      def extract_key_from_url(self, url: str) -&gt; str:                                                                                                                                                                                                                                                                                                                                                                                         
          prefix = f&quot;https://{self.bucket_name}.s3.{settings.AWS_S3_REGION}.amazonaws.com/&quot;                                                                                                                                                                                                                     
          if url.startswith(prefix):                                                                                                                                                                                                                                                                            
              return url.replace(prefix, &quot;&quot;)                                                                                                                                                                                                                                                                    
          return &quot;&quot;   </code></pre>
<p><strong>업로드 흐름</strong></p>
<pre><code>Client → [이미지 파일] → Django Server → AWS S3 → DB 저장  </code></pre><p>단순하고 직관적인 방식이었지만 서버가 모든 파일 업로드를 중계하다 보니 대용량 파일이나 동시 업로드가 많아지면 서버 부하가 커질 수 있다는 단점이 있었다..!</p>
<h2 id="3-presigned-url-도입">3. Presigned URL 도입</h2>
<p> QnA 기능을 개발하면서 이미지 업로드에 Presigned URL 방식을 도입하게 되면서
QnA 앱에 새로운 S3Handler 클래스와 Presigned URL 발급 API가 추가됨                                                                                 </p>
<pre><code class="language-bash">
  apps/qna/                                                                                                                                                                                                                                                                                                     
  ├── utils/                                                                                                                                                                                                                                                                                                    
  │   └── s3_utils.py            # S3Handler (새로 생성)                                                                                                                                                                                                                                                        
  ├── views/                                                                                                                                                                                                                                                                                                    
  │   └── presigned_url_view.py                                                                                                                                                                                                                                                                                 
  ├── serializers/                                                                                                                                                                                                                                                                                              
  │   └── common/                                                                                                                                                                                                                                                                                               
  │       ├── request.py                                                                                                                                                                                                                                                                                        
  │       └── response.py                                                                                                                                                                                                                                                                                       
  └── services/                                                                                                                                                                                                                                                                                                 
      └── common/                                                                                                                                                                                                                                                                                               
          └── command.py      </code></pre>
<h3 id="3-1-문제-발견-코드-중복과-앱-간-의존성">3-1 문제 발견: 코드 중복과 앱 간 의존성</h3>
<p>  QnA의 Presigned URL 구현을 리뷰하면서 문제를 발견하게 되었다                                                                                                                                                        </p>
<pre><code class="language-bash">
  apps/core/utils/s3.py        # S3Client (기존)                                                                                                                                                                                                                                                                
  apps/qna/utils/s3_utils.py   # S3Handler (새로 생성)  </code></pre>
<p><strong>두 클래스의 기능 비교</strong></p>
<table>
<thead>
<tr>
<th>기능</th>
<th>S3Client (core)</th>
<th>S3Handler (qna)</th>
</tr>
</thead>
<tbody><tr>
<td>파일 업로드</td>
<td>✅ <code>upload()</code></td>
<td>❌</td>
</tr>
<tr>
<td>파일 삭제</td>
<td>✅ <code>delete()</code></td>
<td>❌</td>
</tr>
<tr>
<td>URL 생성</td>
<td>✅ <code>build_url()</code></td>
<td>✅ (하드코딩)</td>
</tr>
<tr>
<td>Presigned URL</td>
<td>✅ <code>generate_presigned_url()</code></td>
<td>✅ <code>generate_presigned_put_url()</code></td>
</tr>
<tr>
<td>UUID 자동 생성</td>
<td>❌</td>
<td>✅</td>
</tr>
<tr>
<td>s3v4 서명</td>
<td>❌</td>
<td>✅</td>
</tr>
<tr>
<td>기능이 중복되면서도 각각 다른 부분이 있어서 이대로 두면 유지보수가 어려워질 것 !!!!!!</td>
<td></td>
<td></td>
</tr>
</tbody></table>
<h3 id="3-2-프로필-이미지에서-presigned-url을-쓰려면">3-2. 프로필 이미지에서 Presigned URL을 쓰려면?</h3>
<p>  프로필 이미지도 Presigned URL 방식으로 바꾸고 싶었다
  여기서 두가지 방법이 있는데
  첫번째로는<br />**  QnA의 BasePresignedUrlAPIView 상속 **      받는 방법이다 </p>
<pre><code class="language-python">
# apps/users/views/profile_presigned_url_view.py                                                                                                                                                                                                                                                              
  from apps.qna.views.presigned_url_view import BasePresignedUrlAPIView                                                                                                                         
  class ProfilePresignedUrlAPIView(BasePresignedUrlAPIView):                                                                                                                                                                                                                                                    
      domain = &quot;profile&quot;  </code></pre>
<p>이렇게 하면 빠르게 구현할 가능하지만 users 앱이 qna 앱에 의존하게 된다
앱 간 의존성이 생기면 나중에 분리하거나 수정할 때 문제가 발생  </p>
<p>두번째 방법은 <strong>공용 로직을 core로 이동</strong><br />  S3 관련 로직을 core 앱으로 통합하고 qna와 users 모두 core를 사용하도록 변경<br />  (사실 이게 맞음)</p>
<hr />
<h2 id="4-리팩토링-core로-통합하기">4. 리팩토링: core로 통합하기</h2>
<p>일단 구조부터 변경했다</p>
<pre><code class="language-bash">  apps/                                                                                                                                                                                                                                                                                                         
  ├── core/                                                                                                                                                                                                                                                                                                     
  │   ├── utils/                                                                                                                                                                                                                                                                                                
  │   │   └── s3.py                      # S3Client (Presigned URL 기능 추가)                                                                                                                                                                                                                                   
  │   ├── views/                                                                                                                                                                                                                                                                                                
  │   │   └── presigned_url_view.py      # BasePresignedUrlAPIView                                                                                                                                                                                                                                              
  │   ├── serializers/                                                                                                                                                                                                                                                                                          
  │   │   └── presigned_url.py                                                                                                                                                                                                                                                                                  
  │   └── services/                                                                                                                                                                                                                                                                                             
  │       └── presigned_url_service.py   # FOLDER_MAP 관리                                                                                                                                                                                                                                                      
  ├── qna/                                                                                                                                                                                                                                                                                                      
  │   ├── views/                                                                                                                                                                                                                                                                                                
  │   │   └── presigned_url_view.py      # domain=&quot;question&quot;, &quot;answer&quot;                                                                                                                                                                                                                                          
  │   └── utils/                                                                                                                                                                                                                                                                                                
  │       └── s3_utils.py                # 삭제 예정                                                                                                                                                                                                                                                            
  └── users/                                                                                                                                                                                                                                                                                                    
      └── views/                                                                                                                                                                                                                                                                                                
          └── profile_presigned_url_view.py  # domain=&quot;profile&quot;   </code></pre>
<h3 id="4-1-s3client-확장">4-1. S3Client 확장</h3>
<p> 기존 S3Client에 S3Handler의 기능을 통합했다</p>
<ul>
<li>s3v4 서명 버전 추가                                                                                                                                                                                                                                                                                         <ul>
<li>generate_presigned_put_url 메서드 추가                                                                                                                                                                                                                                                                      </li>
<li>img_url 생성 시 build_url() 메서드 재사용</li>
</ul>
</li>
</ul>
<h3 id="4-2-이제-각-앱에서는-basepresignedurlapiview를-상속받아-domain만-지정하면-됨">4-2. 이제 각 앱에서는 BasePresignedUrlAPIView를 상속받아 domain만 지정하면 됨</h3>
<pre><code class="language-python">  # apps/users/views/profile_presigned_url_view.py                                                                                                                                                                                                                                                              
  from drf_spectacular.utils import extend_schema                                                                                                                                                                                                                                                               

  from apps.core.views.presigned_url_view import BasePresignedUrlAPIView                                                                                                                                                                                                                                        


  class ProfilePresignedUrlAPIView(BasePresignedUrlAPIView):                                                                                                                                                                                                                                                    
      domain = &quot;profile&quot;                                                                                                                                                                                                                                                                                        

      @extend_schema(                                                                                                                                                                                                                                                                                           
          summary=&quot;프로필 이미지 업로드 URL 발급&quot;,                                                                                                                                                                                                                                                              
          tags=[&quot;accounts&quot;],                                                                                                                                                                                                                                                                                    
      )                                                                                                                                                                                                                                                                                                         
      def post(self, request):                                                                                                                                                                                                                                                                                  
          return super().post(request)   </code></pre>
<h3 id="4-3-변경된-업로드-흐름">4-3. 변경된 업로드 흐름</h3>
<ol>
<li><p>Client → Django: POST /me/profile-image/presigned-url/                                                                                                                                                                                                                                                     </p>
<ul>
<li>Request: { &quot;file_name&quot;: &quot;profile.png&quot; }                                                                                                                                                                                                                                                                  </li>
<li>Response: { &quot;presigned_url&quot;: &quot;...&quot;, &quot;img_url&quot;: &quot;...&quot;, &quot;key&quot;: &quot;...&quot; }                                                                                                                                                                                                                                     </li>
</ul>
</li>
<li><p>Client → S3: PUT {presigned_url}                                                                                                                                                                                                                                                                           </p>
<ul>
<li>Body: 이미지 파일                                                                                                                                                                                                                                                                                        </li>
</ul>
</li>
<li><p>Client → Django: PATCH /me/profile-image/                                                                                                                                                                                                                                                                  </p>
<ul>
<li>Request: { &quot;img_url&quot;: &quot;...&quot; }                                                                                                                                                                                                                                                                            </li>
<li>Response: { &quot;detail&quot;: &quot;프로필 사진이 등록되었습니다.&quot; }  </li>
</ul>
</li>
</ol>
<h2 id="5-리팩토링-하다가-궁금한-점">5. 리팩토링 하다가 궁금한 점</h2>
<p>presigned url 방식을 쓰게 되었을 때 <strong>다른 앱의 이미지 업로드 기능을 맡은</strong> 팀원은 로컬 환경에서 AWS 키가 없어도 테스트 가능할 까?</p>
<p>AWS 키가 필요한 작업:</p>
<ul>
<li>presigned url 발급</li>
<li>기존 이미지 삭제</li>
</ul>
<p>AWS 키가 불필요한 작업</p>
<ul>
<li>클라이언트 -&gt; s3 업로드</li>
<li>img_url db에 저장</li>
</ul>
<p>따라서
Presigned url api를 개발/테스트 한다면 필요하지만
그냥 이미지 저장 api만 개발한다면 불필요 할 수도 있다!!!!!</p>
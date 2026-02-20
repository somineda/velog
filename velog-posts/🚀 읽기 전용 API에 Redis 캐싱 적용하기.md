<h2 id="✅-왜-캐싱이-필요한가">✅ 왜 캐싱이 필요한가!!!</h2>
<blockquote>
<p>Django Debug Toolbar로 확인하지 않아도
connection.queries로 추적할 수 있다!!!!!!!</p>
</blockquote>
<pre><code class="language-python">from django.db import connection, reset_queries

reset_queries()
get_available_courses()  
print(len(connection.queries))  # 2

reset_queries()
get_available_courses()
print(len(connection.queries))  # 2 (캐싱 없음 → 동일하게 DB 호출)</code></pre>
<p>👉 매 요청마다 DB를 조회하고 있음</p>
<h2 id="📊-데이터-특성-분석">📊 데이터 특성 분석</h2>
<table>
<thead>
<tr>
<th>엔드포인트</th>
<th>데이터 변경 빈도</th>
<th>조회 빈도</th>
</tr>
</thead>
<tbody><tr>
<td>수강 가능 과목</td>
<td>어드민 기수 추가 시 (하루 몇 번)</td>
<td>유저 페이지 접근 시마다</td>
</tr>
<tr>
<td>회원가입/탈퇴/수강등록 추세</td>
<td>실시간 (1시간 지연 허용)</td>
<td>어드민 대시보드</td>
</tr>
<tr>
<td>탈퇴 사유별 카운트</td>
<td>탈퇴 발생 시</td>
<td>어드민 대시보드</td>
</tr>
</tbody></table>
<p>✅ 변경은 드물고 조회는 빈번 → 캐싱 최적 대상이라고 생각했다!</p>
<blockquote>
<p>특히 Analytics 쿼리는 GROUP BY + COUNT 집계라 데이터 증가 시 비용 증가하고 데이터가 커질수록 무거워진다</p>
</blockquote>
<h2 id="🧩-수정-1-수강-가능-과목-캐싱">🧩 수정 1: 수강 가능 과목 캐싱</h2>
<pre><code class="language-python">course_view.py

def get(self, request):
    courses = get_available_courses()  # 매번 DB 2회
    serializer = AvailableCourseResponseSerializer(courses, many=True)
    return Response(serializer.data)</code></pre>
<p>수정 전 코드를 보면 매 요청마다 DB + Serializer 실행하고 있다
요청이 들어올 때마다 <code>get_available_courses()</code> 호출</p>
<ul>
<li>내부에서 ORM 실행 → DB 쿼리 2번 발생</li>
</ul>
<p><code>AvailableCourseResponseSerializer(...)</code> 생성
<code>serializer.data</code> 접근</p>
<ul>
<li>serializer가 courses를 순회하면서 필드마다 값을 꺼내 dict로 변환</li>
<li>→ 직렬화 연산 수행</li>
</ul>
<p>단순 읽기전용 api 에서 불필요한 리소스가 낭비되고 있다</p>
<pre><code class="language-python">  from django.core.cache import cache as django_cache

  AVAILABLE_COURSES_CACHE_KEY = &quot;available_courses&quot;
  AVAILABLE_COURSES_CACHE_TTL = 300  # 5분

  def get(self, request):
      data = django_cache.get(AVAILABLE_COURSES_CACHE_KEY)

      if data is None:  # 캐시 miss
          courses = get_available_courses()
          data = AvailableCourseResponseSerializer(courses, many=True).data
          django_cache.set(AVAILABLE_COURSES_CACHE_KEY, data, timeout=AVAILABLE_COURSES_CACHE_TTL)

      return Response(data)  # 캐시 hit → DB 쿼리 0</code></pre>
<p>캐시에 <strong>QuerySet이 아닌 이미 직렬화된 결과(data)</strong>를 저장 되도록 수정했다</p>
<p><strong>첫 요청(cache miss):</strong></p>
<ul>
<li><p>캐시 조회 → 없음</p>
</li>
<li><p>ORM 실행 및 DB 조회</p>
</li>
<li><p>serializer로 직렬화</p>
</li>
<li><p>직렬화된 data를 캐시에 저장</p>
</li>
</ul>
<p><strong>이후 요청(cache hit):</strong></p>
<ul>
<li><p>캐시에서 dict/list 데이터를 바로 가져옴</p>
</li>
<li><p>Response 반환</p>
</li>
</ul>
<p>여기서 cache hit 시에는 ORM 실행, DB 쿼리, serializer 실행을 모두 안한다!!!!
API가 거의 메모리 접근 수준의 비용으로 응답을 반환하게 됨!</p>
<h2 id="🧩-수정-2-analytics-4개-엔드포인트">🧩 수정 2: Analytics 4개 엔드포인트</h2>
<p>Analytics는 쿼리 파라미터에 따라 결과가 달라진다
👉 캐시 키에 파라미터를 포함해야 됨</p>
<pre><code class="language-python">  # admin_analytics_views.py
  ANALYTICS_CACHE_TTL = 3600  # 1시간

  # 회원가입 추세
  def get(self, request):
      serializer = SignupTrendsRequestSerializer(data=request.query_params)
      if not serializer.is_valid():
          return Response({&quot;error_detail&quot;: serializer.errors}, status=400)

      interval = serializer.validated_data[&quot;interval&quot;]
      year = serializer.validated_data.get(&quot;year&quot;)

      cache_key = f&quot;signup_trends:{interval}:{year or 'all'}&quot;
      data = django_cache.get(cache_key)

      if data is None:
          result = get_signup_trends(interval, year)
          data = SignupTrendsResponseSerializer(result).data
          django_cache.set(cache_key, data, timeout=ANALYTICS_CACHE_TTL)

      return Response(data, status=200)</code></pre>
<p> <code>signup_trends:monthly:2026</code>과 <code>signup_trends:yearly:all</code>은 다른 캐시 키가 되어 독립적으로 캐싱된다
나머지 3개 엔드포인트(탈퇴 추세, 수강등록 추세, 탈퇴 사유 카운트)도 동일한 패턴으로 적용했다</p>
<h2 id="추가-개선사항-탈퇴-사유-카운트-쿼리-통합">추가 개선사항: 탈퇴 사유 카운트 쿼리 통합</h2>
<p>캐싱과 별개로 같은 테이블을 4번 스캔하는 비효율도 있었다...</p>
<pre><code class="language-python">  total = Withdrawal.objects.count()                                  # 풀 스캔 1
  qs = Withdrawal.objects.values(&quot;reason&quot;).annotate(count=Count(&quot;id&quot;)) # 풀 스캔 2
  oldest = Withdrawal.objects.aggregate(oldest=Min(&quot;created_at&quot;))      # 풀 스캔 3
  latest = Withdrawal.objects.aggregate(latest=Max(&quot;created_at&quot;))      # 풀 스캔 4</code></pre>
<p>수정 전 코드를 보면 4개의 SQL이 각각 withdrawals 테이블을 전체 스캔한다</p>
<pre><code class="language-sql">  [1] SELECT COUNT(*) FROM &quot;withdrawals&quot;
  [2] SELECT reason, COUNT(id) FROM &quot;withdrawals&quot; GROUP BY reason
  [3] SELECT MIN(created_at) FROM &quot;withdrawals&quot;
  [4] SELECT MAX(created_at) FROM &quot;withdrawals&quot;</code></pre>
<p>count, Min, Max는 하나의 aggregate()로 합칠 수 있다!!</p>
<pre><code class="language-python"> aggregates = Withdrawal.objects.aggregate(
      total=Count(&quot;id&quot;),
      oldest=Min(&quot;created_at&quot;),
      latest=Max(&quot;created_at&quot;),
  )
  # → SELECT COUNT(id), MIN(created_at), MAX(created_at) FROM withdrawals  (1 SQL)

  qs = Withdrawal.objects.values(&quot;reason&quot;).annotate(count=Count(&quot;id&quot;))
  # → SELECT reason, COUNT(id) FROM withdrawals GROUP BY reason           (1 SQL)</code></pre>
<p> GROUP BY가 포함된 reason별 카운트는 구조가 달라서 합칠 수 없지만
 나머지 3개를 1개로 합쳤으므로 4 SQL → 2 SQL로 50% 감소
 여기에 1시간 캐싱까지 적용되므로 실질적으로 대시보드 접근 시 DB 히트는 0이게 된다!</p>
<h2 id="📊-실측-결과">📊 실측 결과</h2>
<p>** 수강 가능 과목 조회**</p>
<pre><code>  1차 요청 (캐시 miss): 쿼리 2회
  2차 요청 (캐시 hit):  쿼리 0회  ← DB를 아예 안 침</code></pre><p><strong>회원가입 추세</strong></p>
<pre><code>  1차 요청 (캐시 miss): 쿼리 1회
  2차 요청 (캐시 hit):  쿼리 0회</code></pre><p><strong>탈퇴 사유 카운트</strong></p>
<pre><code>  Before: 쿼리 4회 → After: 쿼리 2회 (캐시 hit 시 0회)</code></pre><p>캐시 무효화에 대해 TTL 기반 캐싱의 한계는 데이터가 변경되어도 TTL이 만료될 때까지 이전 데이터가 보인다는 것이다</p>
<p>이 프로젝트에서는:</p>
<ul>
<li>수강 가능 과목 (TTL 5분): 어드민이 기수를 추가한 뒤 최대 5분간 반영 안 됨 수강신청 페이지 특성상 허용 가능</li>
<li>Analytics (TTL 1시간): 대시보드 통계가 최대 1시간 지연 
실시간 정확성이 필요하지 않은 데이터</li>
</ul>
<p>만약 즉시 반영이 필요한 경우 데이터를 변경하는 시점에서 <code>django_cache.delete(cache_key)</code>로 명시적 무효화를 추가하면 된다!!
예를 들면 기수 상태가 변경될 때 available_courses 키 삭제</p>
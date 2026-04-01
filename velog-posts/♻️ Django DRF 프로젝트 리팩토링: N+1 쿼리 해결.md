<blockquote>
<p>LMS 프로젝트에서 users 앱 전체를 대상으로 코드 품질과 성능을 개선</p>
</blockquote>
<h1 id="n1-쿼리-해결">N+1 쿼리 해결</h1>
<p>Django Debug Toolbar 없이도 django.db.connection.queries로 실측할 수 있다</p>
<h2 id="문제-1-select_related에-인자가-없었다">문제 1: select_related()에 인자가 없었다</h2>
<pre><code class="language-python">  #Before
  queryset = (
      User.objects.filter(role=User.Role.STUDENT)
      .select_related()  #JOIN 0개
      .prefetch_related(&quot;cohort_students__cohort__course&quot;)
  )</code></pre>
<p>select_related()는 인자를 넘기지 않으면 아무 관계도 JOIN하지 않는다 시리얼라이저에서 obj.withdrawal에 접근할 때마다 별도 쿼리가 발생했다!</p>
<h2 id="문제-2-시리얼라이저에서-존재하지-않는-속성을-참조하고-있었다">문제 2: 시리얼라이저에서 존재하지 않는 속성을 참조하고 있었다</h2>
<pre><code class="language-python">  def get_in_progress_course(self, obj):
      cohort_students = getattr(obj, &quot;prefetched_cohort_students&quot;, None)
 # prefetched_cohort_students 속성은 존재하지 않는다..!
      if cohort_students:
          ...

      # 항상 이 fallback으로 빠짐 → 학생마다 추가 쿼리
      cohort_student = obj.cohort_students.select_related(&quot;cohort__course&quot;).first()</code></pre>
<p>  뷰에서 prefetch_related(&quot;cohort_students__cohort__course&quot;)로 프리페치했는데 시리얼라이저는 prefetched_cohort_students라는 다른 이름을 찾고 있었다
getattr은 항상 None을 반환하고 매번 fallback 쿼리가 실행됐다</p>
<h3 id="해결">해결</h3>
<pre><code class="language-python">
  # After — 뷰
  queryset = (
      User.objects.filter(role=User.Role.STUDENT)
      .select_related(&quot;withdrawal&quot;)  # OneToOne 관계 JOIN
      .prefetch_related(&quot;cohort_students__cohort__course&quot;)
  )

  # After — 시리얼라이저
  def get_in_progress_course(self, obj):
      # prefetch_related로 이미 로드된 캐시를 사용
      # .all()은 프리페치 캐시가 있으면 DB를 치지 않는다
      cohort_students = obj.cohort_students.all()
      if not cohort_students:
          return None

      cs = cohort_students[0]
      return {
          &quot;cohort&quot;: {&quot;id&quot;: cs.cohort.id, &quot;number&quot;: cs.cohort.number},
          &quot;course&quot;: {&quot;id&quot;: cs.cohort.course.id, ...},
      }
</code></pre>
<p><strong>중요한 Django ORM의 동작 원리는</strong></p>
<p>  prefetch_related()로 미리 로드된 관계에 .all()을 호출하면 DB에 쿼리를 보내지 않고 Python 메모리의 캐시(_prefetched_objects_cache)에서 가져온다 
  반면 .filter()나 .first()를 호출하면 새로운 쿼리가 발생한다!</p>
<p>**  실측 결과 (Docker DB 기준)**</p>
<p>  Before: 학생 6명 → 쿼리 13회  (1 + 6 withdrawal + 6 cohort)
  After:  학생 6명 → 쿼리  4회  (1 리스트 + 3 prefetch)
  개선율: 69% 감소</p>
<p>  학생 수가 늘어날수록 차이가 극적으로 벌어진다:</p>
<table>
<thead>
<tr>
<th>학생 수</th>
<th>Before</th>
<th>After</th>
</tr>
</thead>
<tbody><tr>
<td>10명</td>
<td>21 쿼리</td>
<td>4 쿼리</td>
</tr>
<tr>
<td>100명</td>
<td>201 쿼리</td>
<td>4 쿼리</td>
</tr>
<tr>
<td>1000명</td>
<td>2001 쿼리</td>
<td>4 쿼리</td>
</tr>
</tbody></table>
<p>  After는 학생 수와 무관하게 항상 고정 4쿼리다</p>
<hr />
<h1 id="계정-상세--탈퇴-상세-api--연쇄-n1">계정 상세 / 탈퇴 상세 API — 연쇄 N+1</h1>
<blockquote>
<p>**  문제:** 두 시리얼라이저 모두 get_assigned_courses(user) 서비스를 호출하는데 이 서비스가 내부적으로 role에 따라 CohortStudent, TrainingAssistant, OperationManager,LearningCoach를 개별 쿼리로 조회했다.</p>
</blockquote>
<pre><code class="language-python">  # assigned_courses_service.py — Before
  def get_assigned_courses_for_student(user):
      # prefetch 없이 항상 새 쿼리
      cohort_students = CohortStudent.objects.filter(user=user).select_related(...)
      return </code></pre>
<p>**  해결:** 3단계로 수정했다!</p>
<h3 id="1단계--뷰에서-필요한-관계를-미리-로드">1단계 — 뷰에서 필요한 관계를 미리 로드</h3>
<pre><code class="language-python">  user = (
      User.objects.select_related(&quot;withdrawal&quot;)
      .prefetch_related(
          &quot;cohort_students__cohort__course&quot;,
          &quot;assisted_cohorts__cohort__course&quot;,   # TrainingAssistant의 related_name
          &quot;managed_courses__course&quot;,             # OperationManager의 related_name
          &quot;coached_courses__course&quot;,             # LearningCoach의 related_name
      )
      .get(id=account_id)
  )
</code></pre>
<h3 id="2단계--시리얼라이저-버그-수정">2단계 — 시리얼라이저 버그 수정</h3>
<pre><code class="language-python">  def get_status(self, obj):
      if hasattr(obj, &quot;withdrawal&quot;) and obj.withdrawal is not None:
          return &quot;WITHDREW&quot;</code></pre>
<h3 id="3단계--서비스에서-prefetch-캐시-재활용">3단계 — 서비스에서 prefetch 캐시 재활용</h3>
<pre><code class="language-python">  # assigned_courses_service.py — After
  def _is_prefetched(obj, attr):
      return attr in getattr(obj, &quot;_prefetched_objects_cache&quot;, {})

  def get_assigned_courses_for_student(user):
      if _is_prefetched(user, &quot;cohort_students&quot;):
          cohort_students = user.cohort_students.all()  # 캐시 사용 쿼리 0회
      else:
          cohort_students = CohortStudent.objects.filter(user=user) 
      return [...]
</code></pre>
<ul>
<li><p>Django ORM은 prefetch_related로 로드한 데이터를 _prefetched_objects_cache라는 딕셔너리에 저장한다</p>
</li>
<li><p>이 캐시가 있으면 .all() 호출 시 DB를 치지 않는다. 캐시가 없는 경우(다른 곳에서 이 서비스를 직접 호출하는 경우)에는 기존처럼 쿼리를 실행해서 하위 호환성을 유지했다</p>
<hr />
<h1 id="마무리">마무리</h1>
<p>이번 리팩토링에서 얻은 교훈을 정리하면...</p>
<ol>
<li><strong>select_related()는 반드시 인자를 명시하자!</strong> 빈 괄호는 아무 일도 하지 않는다</li>
<li><strong>prefetch_related된 데이터는 .all()로 접근하자</strong> .filter()나 .first()를 쓰면 캐시를 무시하고 새 쿼리가 나간다</li>
<li><strong>코드 리뷰 때 SerializerMethodField 내부에서 ORM 호출이 있는지 반드시 확인하자</strong> 리스트 API에서 호출되면 N+1의 직접적 원인이 된다 </li>
<li><strong>중복 코드는 &quot;3번 반복되면 추상화&quot; 원칙을 지키자!</strong> 기존 호출부와의 호환성을 유지하면서 점진적으로 전환하는 것이 안전</li>
<li><strong>connection.queries로 실측하자</strong> </li>
</ol>
</li>
</ul>
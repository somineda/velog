<p>프로젝트 막바지 단계에서 기능 자체는 잘 동작하지만 성능이 점점 문제가 되지 않을까 하는 걱정이 시작되었다
그래서 하나 둘 씩 속도를 줄이고자 리팩토링을 해보았다!</p>
<p><img alt="" src="https://velog.velcdn.com/images/sommnie/post/9fa7a2ae-a970-4ea2-ab18-a3e9c5c60ace/image.jpg" /></p>
<h2 id="🚀-벌크-연산으로-일괄-승인-22배-빠르게-만들기">🚀 벌크 연산으로 일괄 승인 22배 빠르게 만들기</h2>
<h3 id="🔎-문제-발견">🔎 문제 발견</h3>
<p>어드민에서 수강 등록 요청을 일괄 승인하는 기능이 있는데</p>
<pre><code class="language-python"># Before — admin_student_enrollment_service.py

def accept_enrollments(enrollment_ids: list[int]) -&gt; int:
    with transaction.atomic():
        enrollments = StudentEnrollmentRequest.objects.filter(
            id__in=enrollment_ids,
            status=StudentEnrollmentRequest.Status.PENDING,
        ).select_related(&quot;user&quot;, &quot;cohort&quot;)

        count = 0
        for enrollment in enrollments:
            enrollment.status = StudentEnrollmentRequest.Status.APPROVED
            enrollment.accepted_at = timezone.now()
            enrollment.save()                  # SQL 1회

            user = enrollment.user
            user.role = User.Role.STUDENT
            user.save(update_fields=[&quot;role&quot;]) # SQL 1회

            CohortStudent.objects.get_or_create(
                user=user,
                cohort=enrollment.cohort,
            )
            count += 1

    return count</code></pre>
<p>  for 루프 안에서 건마다 save(), save(), get_or_create()를 호출하고 있다...</p>
<pre><code class="language-sql">  -- 반복 1회차
  UPDATE student_enrollment_requests SET status='APPROVED' WHERE id=1;
  UPDATE user SET role='STUDENT' WHERE id=10;
  SELECT * FROM cohort_students WHERE user_id=10 AND cohort_id=5;
  INSERT INTO cohort_students (user_id, cohort_id) VALUES (10, 5);

  -- 반복 2회차
  UPDATE student_enrollment_requests SET status='APPROVED' WHERE id=2;
  UPDATE user SET role='STUDENT' WHERE id=11;
  SELECT * FROM cohort_students WHERE user_id=11 AND cohort_id=5;
  INSERT INTO cohort_students (user_id, cohort_id) VALUES (11, 5);

  -- ... 100회 반복</code></pre>
<p>이걸 100회 반복하게 되면 
100건 × 약 6 SQL = 601개 SQL문이 실행된다🧠🧠🧠</p>
<p>거절은 .update() 한 방인데 승인은 for 루프여서 승인이 더 복잡하긴 하지만 Django ORM의 벌크 연산으로 처리 가능하지 않으까...</p>
<h3 id="수정한-버전">수정한 버전</h3>
<pre><code class="language-python">
  def accept_enrollments(enrollment_ids: list[int]) -&gt; int:
      with transaction.atomic():
          enrollments = list(
              StudentEnrollmentRequest.objects.filter(
                  id__in=enrollment_ids,
                  status=StudentEnrollmentRequest.Status.PENDING,
              ).select_related(&quot;user&quot;, &quot;cohort&quot;)
          )

          if not enrollments:
              return 0

          enrollment_pks = [e.pk for e in enrollments]
          user_ids = [e.user_id for e in enrollments]

          # 등록 요청 일괄 승인 (SQL 1회)
          StudentEnrollmentRequest.objects.filter(pk__in=enrollment_pks).update(
              status=StudentEnrollmentRequest.Status.APPROVED,
              accepted_at=timezone.now(),
          )

          # 유저 role 일괄 변경 (SQL 1회)
          User.objects.filter(id__in=user_ids).update(role=User.Role.STUDENT)

          # CohortStudent 일괄 생성 (SQL 1회, 이미 존재하면 무시)
          CohortStudent.objects.bulk_create(
              [CohortStudent(user=e.user, cohort=e.cohort) for e in enrollments],
              ignore_conflicts=True,
          )

      return len(enrollments)</code></pre>
<p>**  핵심 변경 3가지:**</p>
<ol>
<li>enrollment.save() 루프 → QuerySet.update() 1회: 
WHERE pk IN (...) 조건으로 100건을 한 번에 UPDATE</li>
<li>user.save() 루프 → QuerySet.update() 1회: 
마찬가지로 WHERE id IN (...) 한 방</li>
<li>get_or_create() 루프 → bulk_create(ignore_conflicts=True) 1회: 
INSERT ... ON CONFLICT DO NOTHING으로 한 번에 INSERT</li>
</ol>
<blockquote>
<p>기존 get_or_create()는 매번 SELECT로 존재 여부를 확인한 뒤 INSERT를 했는데 bulk_create에 ignore_conflicts=True 옵션을 주면 DB 레벨에서 중복을 무시하므로 SELECT가 필요 없게 된다!</p>
</blockquote>
<h3 id="📊-벤치마크-결과">📊 벤치마크 결과</h3>
<p>Docker PostgreSQL 환경에서 테스트 데이터를 생성하고 실측했다
테스트 후 savepoint rollback으로 데이터 원복</p>
<table>
<thead>
<tr>
<th>건수</th>
<th>Before</th>
<th>After</th>
<th>속도 개선</th>
</tr>
</thead>
<tbody><tr>
<td>10건</td>
<td>0.0245초 / 61 SQL</td>
<td>0.0044초 / 6 SQL</td>
<td>5.6배</td>
</tr>
<tr>
<td>50건</td>
<td>0.0856초 / 301 SQL</td>
<td>0.0072초 / 6 SQL</td>
<td>11.8배</td>
</tr>
<tr>
<td>100건</td>
<td>0.1910초 / 601 SQL</td>
<td>0.0140초 / 6 SQL</td>
<td>13.6배</td>
</tr>
<tr>
<td>500건</td>
<td>1.2610초 / 3001 SQL</td>
<td>0.0552초 / 6 SQL</td>
<td>22.9배</td>
</tr>
</tbody></table>
<pre><code class="language-python">  # 테스트 코드 
  sid = transaction.savepoint()</code></pre>
<pre><code class="language-python">  # 테스트 유저 + enrollment 100건 생성
  users = User.objects.bulk_create([...])
  enrollments = StudentEnrollmentRequest.objects.bulk_create([...])

  reset_queries()
  start = time.perf_counter()</code></pre>
<pre><code class="language-python">  # Before or After 코드 실행

  elapsed = time.perf_counter() - start
  print(f'{len(connection.queries)} SQL, {elapsed:.4f}초')

  transaction.savepoint_rollback(sid)  # 원복</code></pre>
<p>**  결과:**</p>
<p>500건이면 3001개의 SQL이 실행되면서 1.26초가 걸리는데 수정 이후는 건수와 무관하게 항상 6 SQL 고정(BEGIN, SELECT, UPDATE, UPDATE, INSERT, COMMIT)
건수가 많아질수록 격차가 벌어져 500건에서는 22.9배 차이가 나게 된다!</p>
<p>이 차이가 나는 근본적인 이유는** DB 왕복(round-trip)** 때문인데 
각 SQL문은 애플리케이션 → DB → 애플리케이션 으로 왕복하게 된다 
그 때 이 네트워크 오버헤드가 쿼리 실행 시간보다 클 수 있음!!!!</p>
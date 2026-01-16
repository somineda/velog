<h2 id="🤔-n1-문제가-뭔데">🤔 N+1 문제가 뭔데?</h2>
<p>만약에 학교에 <strong>학생 10명</strong>이 있고 각 학생마다 <strong>담임선생님</strong>이 1명 있다고 가정하면</p>
<p>&quot;학생 10명의 담임선생님 이름을 알고 싶어!&quot;라고 했을 때 데이터베이스에 어떻게 물어봐야 할까</p>
<h3 id="❌-n1-문제-발생하는-방법">❌ N+1 문제 발생하는 방법</h3>
<pre><code>나: &quot;학생 명단 줘&quot; → [철수, 영희, 민수, ...] (1번 쿼리)
나: &quot;철수 담임 누구야?&quot; → 김선생님 (2번째 쿼리)
나: &quot;영희 담임 누구야?&quot; → 박선생님 (3번째 쿼리)
나: &quot;민수 담임 누구야?&quot; → 김선생님 (4번째 쿼리)
... (계속 물어봄) ...</code></pre><p><strong>총 11번 쿼리!</strong> (1 + 10 = 11)</p>
<p>이게 바로 <strong>N+1 문제!!</strong></p>
<ul>
<li>1번: 목록 조회</li>
<li>N번: 각 항목마다 추가 조회</li>
</ul>
<p>학생이 100명이면? <strong>101번 쿼리</strong>.  
1000명이면? <strong>1001번 쿼리</strong>.</p>
<p>API가 느려질 수밖에 없다...</p>
<h3 id="코드로-보면-이런-상황">코드로 보면 이런 상황</h3>
<pre><code class="language-python"># models.py
class Teacher(models.Model):
    name = models.CharField(max_length=50)

class Student(models.Model):
    name = models.CharField(max_length=50)
    teacher = models.ForeignKey(Teacher, on_delete=models.CASCADE)</code></pre>
<pre><code class="language-python"># views.py - N+1 문제 발생!
students = Student.objects.all()  # 쿼리 1번

for student in students:
    print(student.teacher.name)  # 학생마다 쿼리 1번씩 추가 발생!</code></pre>
<p>Django ORM이 <code>student.teacher</code> 에 접근할 때마다 DB에 담임을 물어봄</p>
<hr />
<h2 id="✅-select_related">✅ select_related</h2>
<blockquote>
<p>&quot;학생 명단 주는데 <strong>담임 이름도 옆에 같이 써서</strong> 줘!&quot;</p>
</blockquote>
<pre><code class="language-python">students = Student.objects.select_related('teacher').all()

for student in students:
    print(student.teacher.name)  # 추가 쿼리 없음!</code></pre>
<p><strong>쿼리 1번으로 끝!</strong></p>
<h3 id="sql로-보면">SQL로 보면?</h3>
<pre><code class="language-sql">SELECT student.*, teacher.*
FROM student
INNER JOIN teacher ON student.teacher_id = teacher.id;</code></pre>
<p><code>JOIN</code>을 써서 한 방에 가져옴</p>
<h3 id="언제-쓰나">언제 쓰나</h3>
<ul>
<li><code>ForeignKey</code> (N:1 관계)</li>
<li><code>OneToOneField</code> (1:1 관계)</li>
</ul>
<p><strong>내가 &quot;한 명&quot;을 가리킬 때</strong></p>
<pre><code>학생 ──FK──→ 담임 (1명)
게시글 ──FK──→ 작성자 (1명)
주문 ──FK──→ 고객 (1명)</code></pre><hr />
<h2 id="✅-prefetch_related">✅ prefetch_related</h2>
<p>반대로</p>
<p>학생마다 <strong>숙제가 여러 개</strong> 있다고 하고 학생 10명의 숙제 목록을 조회 한다고 가정하면</p>
<h3 id="❌-n1-문제-발생">❌ N+1 문제 발생</h3>
<pre><code class="language-python">students = Student.objects.all()  # 쿼리 1번

for student in students:
    homeworks = student.homework_set.all()  # 학생마다 쿼리 추가!
    for hw in homeworks:
        print(hw.title)</code></pre>
<h3 id="prefetch_related로-해결">prefetch_related로 해결</h3>
<pre><code class="language-python">students = Student.objects.prefetch_related('homework_set').all()

for student in students:
    for hw in student.homework_set.all():  # 추가 쿼리 없음!
        print(hw.title)</code></pre>
<p><strong>쿼리 2번으로 해결!</strong></p>
<h3 id="sql로-보면-1">SQL로 보면?</h3>
<pre><code class="language-sql">-- 쿼리 1: 학생 전체 조회
SELECT * FROM student;

-- 쿼리 2: 관련 숙제 전체 조회
SELECT * FROM homework WHERE student_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);</code></pre>
<p><code>select_related</code>처럼 JOIN하는 게 아니라 <strong>쿼리 2개를 따로 실행</strong>하고 파이썬에서 합치는 방식</p>
<h3 id="언제-쓰나요">언제 쓰나요?</h3>
<ul>
<li><strong>역관계</strong> (related_name 또는 _set)</li>
<li><code>ManyToManyField</code></li>
</ul>
<p><strong>핵심: 상대방 &quot;여러 명/개&quot;가 나를 가리킬 때</strong></p>
<pre><code>학생 ←─── 숙제들 (여러 개)
게시글 ←─── 댓글들 (여러 개)
게시글 ←──M2M──→ 태그들 (여러 개)</code></pre><hr />
<h2 id="🎯-정리-언제-뭘-쓰지">🎯 정리: 언제 뭘 쓰지?</h2>
<table>
<thead>
<tr>
<th>상황</th>
<th>메서드</th>
<th>예시</th>
</tr>
</thead>
<tbody><tr>
<td>내가 <strong>1명</strong>을 가리킴</td>
<td><code>select_related</code></td>
<td>게시글 → 작성자</td>
</tr>
<tr>
<td>상대 <strong>여러 개</strong>가 나를 가리킴</td>
<td><code>prefetch_related</code></td>
<td>게시글 → 댓글들</td>
</tr>
</tbody></table>
<blockquote>
<p><strong>ForeignKey 따라가면</strong> → <code>select_related</code><br /><strong>역방향이거나 M2M이면</strong> → <code>prefetch_related</code></p>
</blockquote>
<hr />
<h2 id="💡-실전-예제">💡 실전 예제</h2>
<p>블로그 API를 만든다고 생각</p>
<h3 id="모델-구조">모델 구조</h3>
<pre><code class="language-python">class User(models.Model):
    username = models.CharField(max_length=50)

class Category(models.Model):
    name = models.CharField(max_length=50)

class Tag(models.Model):
    name = models.CharField(max_length=30)

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(User, on_delete=models.CASCADE)  # FK
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)  # FK
    tags = models.ManyToManyField(Tag)  # M2M

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()</code></pre>
<h3 id="❌-최적화-전">❌ 최적화 전</h3>
<pre><code class="language-python">def post_list(request):
    posts = Post.objects.all()

    for post in posts:
        print(post.author.username)      # N+1
        print(post.category.name)        # N+1
        print([t.name for t in post.tags.all()])  # N+1
        print([c.content for c in post.comments.all()])  # N+1</code></pre>
<p>게시글 10개 조회하면 쿼리가 <strong>41번</strong> 이상 발생할 수 있어요.</p>
<h3 id="✅-최적화-후">✅ 최적화 후</h3>
<pre><code class="language-python">def post_list(request):
    posts = Post.objects.select_related(
        'author',     # FK: 작성자 (1명)
        'category',   # FK: 카테고리 (1개)
    ).prefetch_related(
        'tags',       # M2M: 태그들 (여러 개)
        'comments',   # 역관계: 댓글들 (여러 개)
    ).all()

    for post in posts:
        print(post.author.username)      # 추가 쿼리 없음
        print(post.category.name)        # 추가 쿼리 없음
        print([t.name for t in post.tags.all()])  # 추가 쿼리 없음
        print([c.content for c in post.comments.all()])  # 추가 쿼리 없음</code></pre>
<p><strong>쿼리 4번으로 끝!</strong></p>
<hr />
<h2 id="🔧-drf-serializer에서-적용하기">🔧 DRF Serializer에서 적용하기</h2>
<p>Django REST Framework 쓴다면, ViewSet에서 <code>get_queryset</code>을 오버라이드하면 된다</p>
<pre><code class="language-python">class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        return Post.objects.select_related(
            'author',
            'category',
        ).prefetch_related(
            'tags',
            'comments',
            'comments__author',  # 댓글의 작성자까지 미리 로드
        )</code></pre>
<h3 id="prefetch-객체로-더-세밀하게">Prefetch 객체로 더 세밀하게</h3>
<p>댓글을 최신순 5개만 가져오고 싶다면?</p>
<pre><code class="language-python">from django.db.models import Prefetch

class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Post.objects.select_related(
            'author',
            'category',
        ).prefetch_related(
            'tags',
            Prefetch(
                'comments',
                queryset=Comment.objects.select_related('author')
                                        .order_by('-created_at')[:5]
            ),
        )</code></pre>
<hr />
<h2 id="🐛-n1-문제-찾는-방법">🐛 N+1 문제 찾는 방법</h2>
<h3 id="1-django-debug-toolbar">1. django-debug-toolbar</h3>
<p>개발 환경에서 쿼리 수를 눈으로 확인할 수 있어요.</p>
<pre><code class="language-bash">pip install django-debug-toolbar</code></pre>
<h3 id="2-로깅으로-쿼리-출력">2. 로깅으로 쿼리 출력</h3>
<pre><code class="language-python"># settings.py
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',
            'handlers': ['console'],
        },
    },
}</code></pre>
<h3 id="3-connectionqueries-확인">3. connection.queries 확인</h3>
<pre><code class="language-python">from django.db import connection

# 쿼리 실행 후
print(f&quot;쿼리 수: {len(connection.queries)}&quot;)
for query in connection.queries:
    print(query['sql'])</code></pre>
<hr />
<h2 id="⚠️-주의사항">⚠️ 주의사항</h2>
<h3 id="1-무조건-다-붙이면-안됨">1. 무조건 다 붙이면 안됨</h3>
<pre><code class="language-python"># 필요 없는 것까지 가져오면 오히려 느려질 수 있음
Post.objects.select_related('author', 'category', 'editor', 'reviewer', ...)</code></pre>
<p><strong>실제로 사용하는 것만</strong> select/prefetch</p>
<h3 id="2-select_related는-join이라-주의">2. select_related는 JOIN이라 주의</h3>
<pre><code class="language-python"># category가 NULL인 게시글은 제외될 수 있음 (INNER JOIN)
Post.objects.select_related('category')

# NULL도 포함하려면 모델에서 null=True 확인</code></pre>
<h3 id="3-prefetch_related-후-filter-주의">3. prefetch_related 후 filter 주의</h3>
<pre><code class="language-python"># 이미 prefetch 했는데 또 filter하면 새 쿼리 발생
posts = Post.objects.prefetch_related('comments')
for post in posts:
    recent = post.comments.filter(created_at__gte=yesterday)  # 새 쿼리!

# Prefetch 객체로 미리 필터링
posts = Post.objects.prefetch_related(
    Prefetch('comments', queryset=Comment.objects.filter(created_at__gte=yesterday))
)</code></pre>
<hr />
<h2 id="📝-요약">📝 요약</h2>
<table>
<thead>
<tr>
<th>문제</th>
<th>해결책</th>
</tr>
</thead>
<tbody><tr>
<td>ForeignKey 접근할 때 N+1</td>
<td><code>select_related</code></td>
</tr>
<tr>
<td>역관계/M2M 접근할 때 N+1</td>
<td><code>prefetch_related</code></td>
</tr>
<tr>
<td>세밀한 제어 필요</td>
<td><code>Prefetch</code> 객체</td>
</tr>
</tbody></table>
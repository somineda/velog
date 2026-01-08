<blockquote>
<p><strong>확정된 일정 당일 오전 7시에 참가자들에게 리마인드 이메일 보내기</strong></p>
</blockquote>
<p> 스케줄 관리 웹사이트를 구현하다가
 미래 일정에 대해 리마인드 기능이 필요해서 쓰는 글!
 <img alt="" src="https://velog.velcdn.com/images/sommnie/post/ac77e371-b5f0-433d-b494-84024d6ea966/image.jpg" /></p>
<p>문제점: 일정 관리 서비스에서 방장이 최종 시간을 확정하면 다음 두 가지 동작이 필요했다</p>
<ol>
<li><strong>즉시</strong>: 모든 참가자에게 확정 이메일 발송  </li>
<li><strong>확정된 날짜 당일 오전 7시</strong>: 리마인드 이메일 발송</li>
</ol>
<p>예를 들어 
📅 <strong>2026년 1월 10일 14:00</strong>으로 확정되었다면:</p>
<ul>
<li><code>2026-01-07 15:30</code> (현재): 확정 이메일 발송 ✅  </li>
<li><code>2026-01-10 07:00</code> (미래): 리마인드 이메일 발송 ⏰  </li>
</ul>
<p>이 문제를 해결하려면 
<strong>Celery Beat으로 주기적 체크</strong>하는 방법이 있는데</p>
<pre><code class="language-python">@periodic_task(run_every=crontab(hour=7, minute=0))
def check_and_send_reminders():
    today = datetime.now().date()
    events = FinalChoice.objects.filter(
        slot__start_datetime__date=today
    )
    for event in events:
        send_reminder_email(event.id)</code></pre>
<p>구조가 단순하고 Celery Beat로 중앙 관리 가능하다는 장점이 있지만
<del>매일 불필요한 DB 쿼리 발생 / 이벤트별 타임존 처리 복잡 / 리소스 낭비</del> 단점이 있어서 이 방법 말고 다른 방법을 찾아보았다!!</p>
<h3 id="celery-apply_async--eta--활용해서-구현">*<em>Celery apply_async + ETA *</em> 활용해서 구현</h3>
<p><strong>장점</strong></p>
<ul>
<li><p>정확한 시간에만 실행</p>
</li>
<li><p>DB 조회 최소화</p>
</li>
<li><p>이벤트별 타임존 스케줄링 가능</p>
</li>
</ul>
<p><strong>단점</strong></p>
<ul>
<li><p>Worker 재시작 시 스케줄 유실 가능..</p>
<hr />
</li>
</ul>
<h3 id="구현-시작">구현 시작!!</h3>
<p>base.py에서 </p>
<blockquote>
<p>CELERY_ENABLE_UTC = True
→ ETA를 UTC 기준으로 변환하여 타임존 이슈를 예방</p>
</blockquote>
<p>리마인드 이메일 Task </p>
<pre><code class="language-python">@shared_task
def send_reminder_email(event_id):
    try:
        event = Event.objects.get(id=event_id, is_deleted=False)
        final_choice = FinalChoice.objects.get(event=event)

        tz = pytz.timezone(event.timezone)
        local_start = final_choice.slot.start_datetime.astimezone(tz)
        local_end = final_choice.slot.end_datetime.astimezone(tz)

        subject = f&quot;[리마인드] {event.title} - 오늘 {local_start.strftime('%H:%M')} 시작&quot;

        message = f&quot;&quot;&quot;
안녕하세요,

'{event.title}' 이벤트가 오늘 진행됩니다!

📅 일정 안내
- 날짜: {local_start.strftime('%Y년 %m월 %d일')}
- 시간: {local_start.strftime('%H:%M')} - {local_end.strftime('%H:%M')}
&quot;&quot;&quot;

        participant_emails = [
            p.email for p in event.participants.all() if p.email
        ]

        if participant_emails:
            send_mail(
                subject,
                message,
                settings.DEFAULT_FROM_EMAIL,
                participant_emails,
            )

    except Exception:
        pass
</code></pre>
<p>리마인드 스케줄링 함수 추가</p>
<pre><code class="language-python">def schedule_reminder_email(event_id):
    event = Event.objects.get(id=event_id, is_deleted=False)
    final_choice = FinalChoice.objects.get(event=event)

    tz = pytz.timezone(event.timezone)
    local_start = final_choice.slot.start_datetime.astimezone(tz)

    reminder_time = local_start.replace(
        hour=7, minute=0, second=0, microsecond=0
    )

    now = datetime.now(tz)

    if reminder_time &gt; now:
        send_reminder_email.apply_async(
            args=[event_id],
            eta=reminder_time
        )
</code></pre>
<p>최종 시간 확정 시 자동 실행</p>
<pre><code class="language-python">@shared_task
def send_final_choice_email(event_id):
    # 확정 이메일 발송
    send_mail(...)

    # 리마인드 이메일 스케줄링
    schedule_reminder_email(event_id)
</code></pre>
<p>✅ 이 때 주의해야 할 점은!!!!!!</p>
<ul>
<li>항상 timezone-aware datetime 사용</li>
<li>이벤트 기준 타임존으로 계산</li>
<li>Celery는 내부적으로 UTC 변환</li>
</ul>
<pre><code class="language-python">  reminder_time = datetime(2026, 1, 10, 7, 0, 0)  # Naive datetime
  send_reminder_email.apply_async(args=[event_id], eta=reminder_time)</code></pre>
<p><strong>이렇게 하면 서버 타임존 기준으로 동작하게 되어서
서울 사용자와 다른 나라에 있는 사용자랑 다른 시간에 이메일을 받게 되는 문제가 발생 할 수도 있다!</strong></p>
<hr />
<h3 id="실제-동작하는지-확인을-해보자">실제 동작하는지 확인을 해보자</h3>
<pre><code class="language-python">def test_reminder_scheduling():
      # 1. 이벤트 생성 (2026-01-10 14:00~16:00)
      event = Event.objects.create(...)

      # 2. 최종 시간 확정
      final_choice = FinalChoice.objects.create(
          event=event,
          slot=time_slot,
          chosen_by=user
      )

      # 3. 확정 이메일 발송 (리마인드 스케줄링 포함)
      send_final_choice_email(event.id)

      # 4. Redis에서 스케줄 확인
      # 2026-01-10 07:00:00+09:00에 예약되었는지 확인</code></pre>
<hr />
<h4 id="⚠️--그러나-여전히-문제점은-있는데">⚠️  그러나 여전히 문제점은 있는데...</h4>
<p>Celery Worker 재시작 시 기존 ETA 스케줄이 사라지는 문제점과
천 개의 이벤트가 동시에 확정되면 Redis에 부하가 발생할 수도 있다는 것이다</p>
<p>이 문제를 해결하기 위해
스케줄을 코드가 아니라 <strong>DB</strong>에 저장해서 사버/워커가 재시작돼도 안 날아가고 운영자가 Admin에서 스케줄을 수정할 수 있도록 로직을 변경했다!</p>
<pre><code># django-celery-beat 설치
pip install django-celery-beat
</code></pre><p>Beat 실행 시 “DB 스케줄러” 사용</p>
<pre><code>celery -A config beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
</code></pre><pre><code class="language-python">from django.utils import timezone
from django_celery_beat.models import ClockedSchedule, PeriodicTask
import json

def schedule_reminder_with_django_celery_beat(event_id, reminder_time_aware):
    clocked, _ = ClockedSchedule.objects.get_or_create(
        clocked_time=reminder_time_aware
    )

    PeriodicTask.objects.create(
        clocked=clocked,
        name=f&quot;reminder-event-{event_id}-{reminder_time_aware.isoformat()}&quot;,
        task=&quot;apps.events.tasks.send_reminder_email&quot;,
        args=json.dumps([event_id]),
        one_off=True,   # 한 번 실행 후 비활성화
        enabled=True,
    )
</code></pre>
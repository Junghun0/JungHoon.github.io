---
layout: post
title: "[Android] 메인 스레드 & Handler 이해하기"
subtitle: ' About Main Thread & Handler '
author: "JungHoon-Park"
header-style: text
header-mask: 0.3
comments: true
catalog:  true
tags:
  - Android
  - Mobile
  - Study
---

# UI 처리를 위한 메인 스레드
애플리케이션은 성능을 위해 멀티 스레드를 많이 활용하지만, `UI를 업데이트하는 데는 단일 스레드 모델(해당 변수나 메서드를 사용하는 시점에는 하나의 스레드만 실행된다.)이 적용`된다.
멀티 스레드로 UI를 업데이트하면 동일한 UI 자원을 사용할 때 교착상태(deadlock), 경합상태(race condition) 등 여러 문제가 발생할 수 있다.
따라서 `UI 업데이트를 메인 스레드에서만 허용`한다.<br/>
앱 프로세스가 시작되면서 메인 스레드가 생성된다. `컴포넌트(액티비티, 서비스, 브로드캐스트 리시버, Application)의 생명주기 메서드와 그 안의 메서드 호출은 기본적으로 메인 스레드에서 실행`된다.<br/>
`메인 스레드는 UI를 변경할 수 있는 유일한 스레드이기 때문에 메인 스레드를 UI 스레드로 부르기도 한다`.
서비스, 브로드캐스트 리시버, Application은 사용자 인터페이스(UI)가 아니기 때문에, UI 스레드에서 실행된다고 하면 개념을 혼동하기 쉽다.
UI를 변경하는 유일한 수단이라는 의미를 강조하기 위해서 UI 스레드를 쓴다고 이해하자.

## 안드로이드 애플리케이션에서 메인 스레드
안드로이드 애플리케이션의 메인 스레드는 뭔가 특별한 것일까? 그렇지 않다. 안드로이드 프레임워크 내부 클래스인 `android.app.ActivityThread`가 바로 애플리케이션의 메인 클래스(실제로는 ZygoteInit 이 시작점이지만 ActivityThread가 시작점이라고 이해하면 된다.)이고, ActivityThread의 main() 메서드가 애플리케이션의 시작 지점이다.
ActivityThread는 클래스명 때문에 Thread를 상속한 것이라고 생각할 수 있지만, 어떤 것도 상속하지 않은 클래스이다. 클래스명의 앞부분이 Activity라서 액티비티와 관련된 것으로 오해할 수도 있다. ActivityThread는 액티비티만 관련되어 있는 것도 아니고 모든 컴포넌트들이 다 관련되어 있다.
여기서 'Activity'라는 이름은 앱의 '활동'으로 이해하는 것이 낫다.<br/><br/>
**ActivityThread.java**
~~~java
public static void main(String[] args) {
    SamplingProfilerIntegration.start();
    CloseGuard.setEnabled(false);
    Environment.initForCurrentUser();
    EventLogger.setReporter(new EventLoggingReporter());
    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper(); //1

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if(sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    AsyncTask.init();

    Looper.loop(); //2

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
~~~
1. 메인 Looper를 준비한다.<br/>
2. 제일 중요한 라인으로 여기서 `UI Message를 처리`한다. Looper.loop() 메서드에 무한 반복문이 있기 때문에 main() 메서드는 프로세스가 종료될 때까지 끝나지 않는다.

# Looper 클래스
메인 스레드의 동작을 이해하기 위해서는 `ActivityThread의 main() 메서드에서 중심이 되는 Looper를 이해하는 것이 필요`하다. 이 부분에서는 Looper에 대해 알아보자.<br/>

## 스레드 별로 Looper 생성
`Looper는 TLS(Thread Local Storage)에 저장되고 꺼내어진다`. 그 과정을 구체적으로 살펴보자<br/>
`ThreadLocal<Looper>에 set()` 메서드로 새로운 `Looper를 추가`하고, `get() 메서드로 Looper를 가져올 때 스레드별로 다른 Looper가 반환된다.`<br/>
그리고 `Looper.prepare() 에서 스레드별로 Looper를 생성한다.` 특히 메인 스레드의 `메인 Looper`는 `ActivityThread의 main() 메서드에서 Looper.prepareMainLooper() 를 호출하여 생성`한다.<br/>
Looper.getMainLooper()를 사용하면 어디서든 메인 Looper를 가져올 수 있다.<br/>

## Looper별로 MessageQueue 가짐
Looper는 각각의 MessageQueue를 가진다. `특히 메인 스레드에서는 이 MessageQueue를 통해서 UI 작업에서 경합 상태를 해결`한다. 개발 중에 큐 구조가 필요할 때 java.util.Queue의 여러 구현체를 사용할 수도 있지만 Looper를 사용하는 것도 고려해 보자. 특히 스레드별로 다른 큐를 사용할 때는, Looper를 대신 사용하는 게 더 단순해질 수 있다.

## Looper.loop() 메서드의 주요 코드
**Looper.java**
~~~java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException(
            "No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    for(;;) {
        Message msg = queue.next();//1
        if (msg == null) {//2
            return;
        }
        msg.target.dispatchMessage(msg);//3
        msg.recycle();
    }
}

public void quit() {
    mQueue.quit(false);
}

public void quitSafely() {
    mQueue.quit(true);
}
~~~
**1.** MessageQueue 에서 다음 Message를 꺼낸다.<br/>
**2.** Message가 null 이라면 리턴한다.<br/>
**3.** Message를 처리한다. target은 Handler 인스턴스이고 결과적으로 Handler의 dispatchMessage() 메서드가 Message를 처리한다.<br/>
**2.** 에서 MessageQueue 에서 꺼낸 Message가 언제 null이 될까? 바로 Looper가 종료될 때이다. Looper를 종료하는 메서드가 quit(), quitSafely()이다. 두 메서드는 구체적으로 MessageQueue의 quit(boolean safe) 메서드를 호출하고, 그 결과 `1`의 queue.next()에서 null을 리턴하고 `2`에서 for 반복문이 종료된다.<br/>

## quit()과 quitSafely() 메서드 차이
Looper API 문서에서 quit()과 quitSafely() 메서드의 차이도 확인하자. `quit() 메서드는 아직 처리되지 않은 Message를 모두 제거`한다. `quitSafely() 메서드는 sendMessageDelayed() 등을 써서 실행 타임스탬프를 뒤로 미룬 지연 Message를 처리하는데, quitSafely() 메서드를 실행하는 시점에 현재 시간보다 타임스탬프가 뒤에 있는 Message를 제거하고 그 앞에 있는 Message는 계속해서 처리`한다. quitSafely() 메서드는 젤리빈 API 레벨 18 이상에서 쓸 수 있다.

# Message 와 MessageQueue
MessageQueue는 Message를 담는 자료구조이다. `MessageQueue의 구조`는 java.util.Queue의 구현체 가운데서 ArrayBlockingQueue보다는 `LinkedBlockingQueue에 가깝다`.<br/>
ArrayBlockingQueue와 LinkedBlockingQueue를 비교해 보면, ArrayBlockingQueue는 배열에 노드를 추가하는 방식이고 LinkedBlockingQueue는 변수에 다음 노드에 대한 링크를 가지는 방식이다. 배열 구조에 비해서 `링크 구조는 일반적으로 개수 제한이 없고 삽입 속도가 빠르다`. 대신 `배열 구조는 랜덤 인덱스 접근이 가능`하고 `링크 구조는 순차 접근`을 해야 한다. `MessageQueue에는 Message가 실행 타임스탬프순으로 삽입되고 링크로 연결되어, 실행 시간이 빠른 것부터 순차적으로 꺼내어 진다`.

## Message 클래스
먼저 MessageQueue에 담기는 Message를 살펴보자. Message 클래스 내용을 보면 Message에 어떤 데이터가 전달되는지 알 수 있고, 이 데이터가 이후에 어떻게 사용되는지 이해하는 데도 도움이 된다.<br/>
**Message.java**
~~~java
public final class Message implements Parcelable {
    public int what;

    public int arg1;

    public int arg2;

    public Object obj;

    public Messenger replyTo;

    ...
}
~~~
MessageQueue에 들어가는 Message에는 퍼블릭 변수에 int, arg1, arg2, obj, replyTo, what 5개가 있다. Message를 만들 때 이 변수에 값을 넣는다.<br/>
Message에는 패키지 프라이빗 변수도 여러 개 있다. `android.os 패키지 아래에 Looper, Message, MessageQueue, Handler도 있는데, 이들 클래스에서 Message의 패키지 프라이빗 변수에 직접 접근`한다. `target 이나 callback 같은 것들이 Handler에서 postXxx(), sendXxx() 메서드를 호출할 때 Message에 담겨서 MessageQueue에 들어간다`.<br/>
postXxx(), sendXxx() 메서드에서 실행 시간(what)이 전달되고, `나중에 호출한 것이라도 타임스탬프가 앞서면 큐 중간에 삽입`된다. 이것이 `삽입이 쉬운 링크 구조를 사용한 이유`이다.

## obtain() 메서드를 통한 Message 생성
`Message를 생성할 때는 오브젝트 풀(Object Pool)에서 가져오는 Message.obtain()` 메서드나 `Handler의 obtainMessage() 메서드 사용을 권장`한다.<br/>
내부적으로 Handler의 obtainMessage()는 Message.obtain()을 다시 호출한다. 오브젝트 풀은 Message에 정적 변수로 있고(여기서도 링크로 연결됨) Message를 최대 50개까지 저장한다.<br/>
그리고 Looper.loop() 메서드에서 Message를 처리하고 나서 recycleUnChecked() 메서드를 통해 Message를 다시 초기화해서 재사용한다. 오브젝트 풀이 최대 개수에 도달하지 않았다면 오브젝트 풀에 Message를 추가한다. new Message()와 같이 기본 생성자로 생성해서 값을 채워도 동작에는 문제가 없어 보이지만 Message 처리가 끝나면 불필요하게 풀에 Message를 추가하면서 금방 풀의 최대 개수에 이른다. <br/>
Message를 풀에서 가져와서(여분이 없으면 새로 생성) 풀에 돌려줘야지 따로 생성해서 풀에 돌려주면 자원이 낭비된다.(`Handler 에서 Message를 처리하는게 아니라, 값을 전달하기 위한 용도로 Message를 대신 사용해서 주고받는 경우에만 Message 기본 생성자를 사용하자.`)

<br/>

# Handler 클래스
Handler는 Message를 MessageQueue에 넣는 기능과 MessageQueue에서 꺼내 처리하는 기능을 함께 제공한다. 여기서는 Handler가 Looper, MessageQueue와 어떤 관계가 있는지 살펴보고 Handler의 사용 방법에 대해서 알아보자.

## Handler 생성자
Handler를 사용하려면 먼저 생성자를 이해해야 한다. Handler에는 기본 생성자 외에도 Handler.Callback이 전달되는 생성자도 있고, Looper가 전달되는 생성자도 있다.
- Handler()
- Handler(Handler.Callback callback)
- Handler(Looper looper)
- Handler(Looper looper, Handler.Callback callback)

당연한 얘기지만 1~3번째 생성자는 파라미터 개수가 가장 많은 4번째 생성자를 다시 호출한다. Handler는 Looper(결국 MessageQueue)와 연결되어 있다. Looper는 이들 생성자와 어떤 관계일까?
`기본 생성자는 바로 생성자를 호출하는 스레드의 Looper를 사용하겠다는 의미`이다.(Looper는 스레드 로컬 스토리지에 들어간다). 따라서 메인 스레드에서 Handler 기본 생성자는 앱 프로세스가 시작할 때 ActivityThread에서 생성한 메인 Looper를 사용한다. `Handler 기본 생성자는 UI 작업을 할 때 많이 사용된다.`

**백그라운드 스레드에서 Handler 기본 생성자 사용하려면 Looper 필요**
그럼 백그라운드 스레드에서 Handler 기본 생성자를 사용한다면 어떨까? 이 때 Looper가 준비되어 있지 않다면 RuntimeException이 발생한다. `RuntimeException의 "Can't create handler inside thread that has not called Lopper.prepare"라는 메시지에 따른 문제`를 해결하려면, 먼저 `Looper.prepare()를 실행해서 해당 스레드에서 사용할 Looper를 준비해야한다.` 내부적으로 prepare() 메서드는 MessageQueue를 생성하는 것 외에 별다른 동작을 하지 않는다. Looper API 문서를 보면 백그라운드 스레드에서 Handler를 사용하는 샘플이 나온다.
~~~java
class LooperThread extends Thread {
    public Handler mHandler;

    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            public void handleMessage(Message msg) { //1
                //여기서 Message 처리
            }
        };
        Looper.loop();
    }
}
~~~
LooperThread에서 스레드를 시작하면 Looper.loop()에 무한 반복문이 있기 때문에 해당 스레드는 종료되지 않는다. 그리고 mHandler에서 sendXxx(), postXxx() 메서드를 사용하면 스레드 내에서 `1`을 실행한다.<br/>

**호출 위치가 메인스레드인지 확인이 쉽지 않음**
개발 중에 Looper가 준비되지 않아서 RuntimeException을 만나는 경우가 있다. `백그라운드 스레드에서 Handler 기본 생성자를 쓴 경우`이다.<br/>
메서드 호출 스택이 깊어지면 호출 위치가 메인 스레드인지 백그라운드 스레드인지 확인이 금방 안 되기도 한다. 예를 들어 어떤 메서드에서는 단순하게 TextView 의 setText()를 실행하지만, 메서드를 호출하는 곳이 여러 군데이거나 메서드 호출 스택이 깊어서 어떤 스레드에서 호출하는지 알기 쉽지 않은 상황을 가정해보자.<br/>
여러 곳에서 사용하는 메서드라면 메인 스레드뿐 아니라 백그라운드 스레드에서 생성하는지 모호한 경우가 있다.
메인 스레드에서는 메인 Looper가 이미 있어서 문제가 되지 않지만, 백그라운드 스레드에서는 대응하는 Looper가 없다면 RuntimeException을 만나게 된다.
예를 들어, 아래 코드에서 BadgeListener의 updateBadgeCount() 에서 UI를 변경한다.<br/>
~~~java
public void process(BadgeListener listener) {
    int count = ...
    linstener.updateBadgeCount(count);
}
~~~
process() 메서드는 메인 스레드에서 호출할 때는 문제가 없다. 하지만 백그라운드 스레드에서 호출한다면 CalledFromWrongThreadException 이 발생한다. 이 때 Looper와 Handler의 관계를 잘 모른다면 아래처럼 작성할 수 도 있다.
~~~java
public void process(BadgeListener listner) {
    int count = ...
    new Handler().post(new Runnable() { //1
        public void run() {
            listener.updateBadgeCount(count);//2
        }
    });
}
~~~
백그라운드 스레드에서 Looper가 연결되어 있지 않다면 `1`에서 RuntimeException이 발생한다. 백그라운드 스레드에서 Looper를 생성해도 `2`는 UI를 업데이트하는 작업이기 때문에 이번에는 CalledFromWrongThreadException이 발생한다. 메인 스레드에서만 UI를 업데이트할 수 있는데, 바로 메인 Looper와 연결된 Handler가 필요하다. 이 때 Handler의 세 번째 생성자인 Handler(Looper looper)를 사용하면 된다.
~~~java
public void process(BadgeListener listner) {
    int count = ...
    new Handler(Looper.getMainLooper()).post(new Runnable() {
        public void run() {
            listener.updateBadgeCount(count);
        }
    });
}
~~~
Handler 생성자에 Looper.getMainLooper()를 전달하면, 메인 Looper의 MessageQueue에서 Runnable Message를 처리한다. 따라서 run() 메서드의 코드는 메인 스레드에서 실행되고 UI를 문제없이 업데이트한다.

## Handler 동작
앞에서도 언급했듯이 Handler는 Message를 MessageQueue에 보내는 것과 Message를 처리하는 기능을 함께 제공한다. post(), postAtTime(), postDelayed() 메서드를 통해서 Runnable 객체도 전달되는데, Runnable도 내부적으로 Message에 포함되는 값이다.<br/>
Handler에서 Message를 보내는 메서드 목록을 살펴보자.<br/>


|        | send | post |
|:--------|:--------|:--------|
| 기본 | sendEmpty(int what) <br/> sendMessage(Message msg) | post(Runnable r) |
| Delayed | sendEmptyMessageDelayed(int what, long delayMillis) <br/> sendMessageDelayed(Message msg, long delayMillis) | postDelayed(Runnable r, long delayMillis) |
| AtTime | sendEmptyMessageAtTime(int what, long uptimeMillis) <br/> sendMessageAtTime(Message msg, long uptimeMillis) | postAtTime(Runnable r, Object token, long uptimeMillis) <br/> postAtTime(Runnable r, long uptimeMillis) |
| AtFrontOfQueue| sendMessageAtFrontOfQueue(Message msg) | postAtFrontOfQueue(Runnable r) |

- **sendEmptyMessage()**, **sendEmptyMessageDelayed()**, **sendEmptyMessageAtTime()** 메서드는 Message의 what 값만을 전달한다.
- **-Delayed()** 로 끝나는 메서드는 내부적으로 -AtTime() 메서드를 호출한다. 현재 시간 uptimeMillis에 delayMillis를 더한 값이 uptimeMillis 파라미터에 들어간다.
- **sendMessageAtFrontOfQueue()** 나 **postAtFrontOfQueue()** 메서드는 특별한 상황이 아니면 쓰지 말라는 가이드가 있다. 권한 문제나 심각한 서버 문제처럼, 앱을 더 이상 쓸 수 없는 특별한 때가 아니면 사용할 일이 없다. 남용하면 안되는 메서드이다.

## dispatchMessage() 메서드
Looper.loop() 메서드에서 호출하는 Handler의 dispatchMessage() 메서드를 보자<br/>
**Handler.java**
~~~java
public void dispatchMessage(Message msg) {
    if(msg.callback != null) {//1~
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }//1
}

private static void handleCallback(Message message) {
    message.callback.run();
}
~~~
`1~1` callback Runnable 이 있다면 그것을 실행하고 아니면 handleMessage()를 호출한다.
dispatchMessage()는 퍼블릭 메서드이다. 드물긴 하지만 sendXxx() 나 postXxx()를 쓰지 않고 dispatchMessage() 메서드를 직접 호출하기도 하는데, 이 때는 MessageQueue를 거치지 않고 직접 Message를 처리한다.











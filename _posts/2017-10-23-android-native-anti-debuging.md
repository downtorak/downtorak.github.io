---
layout:     post
title:      Android Native의 ptrace 동적디버깅 막기
date:       2017-10-23 16:00:00 +0900
categories: "Android Security"
summary:    Android Native에 대한 ptrace 동적디버깅 막는 방법
---

* content
{:toc}

### 동적디버깅
프로그래밍을 하다보면 여러 버그가 발생할 수 있다.
코드의 문법에서 발생하는 버그들은 빌드 시 대부분 빌드오류로 걸러낼 수 있지만,
프로그램의 동작 중에 발생하는 로직의 버그들은 디버깅이 쉽지않다.

이를 좀 더 쉽게하기 위해 동적디버깅 툴을 사용하여
브레이크를 걸고 콜스택과 변수 값을 확인하는 방법으로 디버깅을 한다.

이러한 동적디버깅은 개발자에게 편의를 제공하는 동시에
프로그램 해킹의 도구로 사용되기도 한다.
그래서 이러한 동적디버깅을 할 수 없도록 막음으로써
보안을 강화하는 방법에 대해 알아보자.

여기서는 Native 코드를 사용하는 Android Application 기준으로 진행하겠지만,
어차피 Android도 리눅스 기반이기 때문에 리눅스 프로그램에도 적용가능하다.


### ptrace
ptrace는 Process Trace의 약자로
리눅스 기반 생성된 프로세스가 어떻게 움직이며, 어떤 식으로 데이타를 읽고 쓰는지, 
어떤 에러를 내는지 추적을 하기위해 마련된 시스템 콜이다.
ptrace를 통해 대상의 내부 상태를 조사하고 조작하게 함으로써,
ptrace를 사용하는 프로세스가 다른 프로세스를 제어할 수 있다.
우리가 사용하는 디버깅툴(gdb, strace 등)들은 내부에서 ptrace를 사용한다.

이러한 ptrace를 통한 조작을 막는 방법은 의외로 간단하다.
프로세스에 대해 ptrace로 접근할 수 있는 프로세스는 1개로 제한되어 있다.
이 점을 이용하는 것이다. 다른 프로세스가 이 한자리를 차지하기 전에
미리 자리를 채워놓으면 된다.

리눅스 시스템에서 ptrace로 접근한 프로세스는 `/proc/(pid)/status`에서 확인할 수 있다.
status 내용 중 `TracerPid`가 ptrace로 접근한 프로세스의 PID이다.

아래의 예시에서 `TracerPid`는 `0`이다.
이것은 ptrace로 접근한 프로세스가 없음을 의미한다.
```no-highlight
# cat /proc/23785/status
Name:   ************
State:  S (sleeping)
Tgid:   23785
Pid:    23785
PPid:   143
TracerPid:      0
Uid:    10114   10114   10114   10114
Gid:    10114   10114   10114   10114
FDSize: 256
Groups: 1015 1028 3002 3003 9997 50114
VmPeak:  1307576 kB
VmSize:  1139108 kB
VmLck:         0 kB
...
```


### 동적디버깅 막기
앞서 언급한 것과 같이 다른 프로세스가 ptrace로 접근하기 전에
미리 ptrace로 접근해 놓으면 동적디버깅을 막을 수 있다.

우선 많이 알려진 방법부터 알아보면 `PTRACE_TRACEME`를 사용하는 방법이다.
```cpp
#include <sys/ptrace.h>

extern "C" jint JNIEXPORT JNICALL JNI_OnLoad(JavaVM *jvm, void *reserved) {
  ptrace(PTRACE_TRACEME, 0, NULL, NULL);
  
  ...
}
```

Android Native를 사용하기 위해 최초에 호출되는 `JNI_OnLoad` 함수에서
`ptrace(PTRACE_TRACEME, 0, NULL, NULL)`를 호출한다.
이러면 자신의 프로세스를 자신이 trace하게 되어 타 프로세스의 접근을 막을 수 있다.

그런데 이 방법은 리눅스 프로그램에서는 사용할 수 있는데 Android에서는
접근권한과 관련하여 불가한 경우가 있다.
이에 대해서는 [여기](https://groups.google.com/forum/#!topic/android-security-discuss/gxSX6bIoCTc)를 보면
Android OS 버젼에 따라 불가할 수 있다는 것을 알 수 있다.

그래서 `fork()`를 이용한 방법이 있다.
(출처: [More Android Anti-Debugging Fun](http://www.vantagepoint.sg/blog/89-more-android-anti-debugging-fun))
```cpp
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/wait.h>


static int child_pid;

void *monitor_pid(void *) {
  int status;

  waitpid(child_pid, &status, 0);

  _exit(0);

}

void anti_debug() {
  child_pid = fork();

  if (child_pid == 0) {
    int ppid = getppid();
    int status;

    if (ptrace(PTRACE_ATTACH, ppid, NULL, NULL) == 0) {
      waitpid(ppid, &status, 0);

      ptrace(PTRACE_CONT, ppid, NULL, NULL);

      while (waitpid(ppid, &status, 0)) {
        if (WIFSTOPPED(status)) {
          ptrace(PTRACE_CONT, ppid, NULL, NULL);
        }
        else {
          // Process has exited
          _exit(0);
        }
      }
    }
  }
  else {
    pthread_t t;

    pthread_create(&t, NULL, monitor_pid, (void *)NULL);
  }
}

extern "C" jint JNIEXPORT JNICALL JNI_OnLoad(JavaVM *jvm, void *reserved) {
  anti_debug();

  ...
}
```

위와 같은 방법으로 `fork()`를 이용해 child process를 생성하고 child process가
parent process를 trace하게 되어 타 프로세스의 접근을 막을 수 있다.



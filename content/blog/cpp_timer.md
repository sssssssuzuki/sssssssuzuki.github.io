+++
title = "C++で周期的な処理を行う"
date = 2022-08-03
[taxonomies]
categories = ["blog"]
tags = ["C++", "Timer"]
[extra]
toc = true
katex = true
+++

C++で周期的な処理を行う必要があったので, 様々な手法について調べてみた.

周期処理について調べると, ほとんどのサイトでOS固有のタイマーを使用した処理が出てくるが, 移植性が低い.
また, Windowsのタイマーはms単位でしか指定できない.

ので, 別の方法を模索しつつ, 各タイマーの精度を調べてみた.

# TL;DR

基本的に, 普通に[ループとstd::this_thread::sleep_untilの組み合わせ](#ruputostd-this-thread-sleep-untilnozu-mihe-wase)を使用すればいい.
移植性もあり, Windowsでも $\SI{16.666}{ms}$ 周期 ($\SI{60}{fps}$) 程度なら問題なく, $\SI{500}{us}$ 周期でもなんとかなる.
さらに, Linux/macの場合は, $\SI{100}{us}$周期 ($\SI{10000}{fps}$) とかも行ける.

Windowsでそれ以上の精度を求める場合は, PerformanceCounterとSoftware Sleepを組み合わせるしかない.

# 周期的に処理を行うための様々な方法

以下に各OS毎に周期処理を行う方法を紹介する.

なお, 共通化を図るために以下のようなインターフェースを用意した.

```cpp
template <typename T>
class Timer {
 public:
  virtual void start(T* callback, uint32_t interval_ns) = 0;
  virtual void stop() = 0;
  virtual ~Timer() {}
};
```

型`T`はメンバ関数として`callback`という名前の引数なしの関数を持てば何でも良い.
また, 周期は`interval_ns`でns単位で指定するとする.

`start`で周期処理が開始され, 指定した周期で, `T`型のクラスのメンバ関数`callback`が呼ばれる.
`stop`で周期処理を止める.

以下, 一部ヘッダやプラグマは省略する.
また, エラー処理も面倒なので省いている.

完全なソースコードは[GitHub](https://github.com/sssssssuzuki/periodic-timer-sample)においてある.

## ループとstd::this_thread::sleep_untilの組み合わせ

以下のように所定の時間まで`std::this_thread::sleep_until`で待てばいい.

この方法は各OS共通で使用できる.

```cpp
template <typename T>
class StdSleepTimer : public Timer<T> {
 public:
  StdSleepTimer() : _thread_running(false) {}

  void start(T* callback, uint32_t interval_ns) override {
    _thread_running = true;
    const auto interval = std::chrono::microseconds(interval_ns / 1000);
    _timer_callback = std::thread([this, interval, callback]() {
      auto next = std::chrono::high_resolution_clock::now();
      while (_thread_running) {
        next += interval;
        std::this_thread::sleep_until(next);
        callback->callback();
      }
    });
  }

  void stop() override {
    if (!_thread_running) return;
    _thread_running = false;
    if (_timer_callback.joinable()) _timer_callback.join();
  }

  ~StdSleepTimer() { stop(); }

 private:
  std::thread _timer_callback;
  bool _thread_running;
};
```

## (Windows) MultiMedia Timerを使う

[timeSetEvent](https://docs.microsoft.com/ja-jp/previous-versions/dd757634(v=vs.85))を使用する.

ドキュメントによるとTimerQueue Timerの使用が推奨されているが, TimerQueue Timerには問題がある (後述) ので使うならこちらを使った方がいい.

周期はms単位でしか指定できない. フラグの意味は上記ドキュメント参照.

`timeSetEvent`の第3引数で指定したコールバック関数を第1引数で指定した間隔で実行してくれる.
ちょっと面倒だが, 第4引数でコールバッククラスのポインタを渡して, コールバック関数内で元の型に戻して, 所望の関数を呼び出している. 

```cpp
template <typename T>
class MultiMediaTimer : public Timer<T> {
 public:
  MultiMediaTimer() : _timer_id(0) {}

  void start(T* callback, uint32_t interval_ns) override {
    const auto interval = interval_ns / 1000 / 1000;
    _timer_id = timeSetEvent(std::max(1u, interval), 1, timer_callback, reinterpret_cast<DWORD_PTR>(callback),
                             TIME_PERIODIC | TIME_CALLBACK_FUNCTION | TIME_KILL_SYNCHRONOUS);
  }

  void stop() override {
    if (_timer_id == 0) return;
    timeKillEvent(_timer_id);
    _timer_id = 0;
  }

  ~MultiMediaTimer() { stop(); }

 private:
  static void CALLBACK timer_callback(UINT, UINT, DWORD_PTR dw_user, DWORD_PTR, DWORD_PTR) { reinterpret_cast<T*>(dw_user)->callback(); }

  uint32_t _timer_id;
};
```

## (Windows) TimerQueue Timerを使う

こちらも$\SI{1}{ms}$単位で指定できるんだけど, 実際にはどうやっても$\SI{16}{ms}$以下の周期にならない.

MultiMedia Timerを使えばいいと思う.

```cpp
template <typename T>
class TimerQueueTimer : public Timer<T> {
 public:
  TimerQueueTimer() : _timer_queue(nullptr), _timer(nullptr) {}

  void start(T* callback, uint32_t interval_ns) override {
    const auto interval = interval_ns / 1000 / 1000;
    _timer_queue = CreateTimerQueue();
    CreateTimerQueueTimer(&_timer, _timer_queue, (WAITORTIMERCALLBACK)timer_callback, (void*)callback, 0, std::max(1u, interval), 0);
  }

  void stop() override {
    if (_timer == nullptr) return;
    if (_timer_queue == nullptr) return;
    DeleteTimerQueueTimer(_timer_queue, _timer, nullptr);
    DeleteTimerQueue(_timer_queue);
    _timer = nullptr;
    _timer_queue = nullptr;
  }

  ~TimerQueueTimer() { stop(); }

 private:
  static void CALLBACK timer_callback(PVOID lpParam, BOOLEAN TimerOrWaitFired) { reinterpret_cast<T*>(lpParam)->callback(); }

  HANDLE _timer_queue;
  HANDLE _timer;
};
```

## (Windows) ThreadPool Timerを使う

これも, TimerQueue Timerと同様で使う場面はないだろう.

```cpp
template <typename T>
class ThreadPoolTimer : public Timer<T> {
 public:
  ThreadPoolTimer() : _timer(nullptr) {}

  void start(T* callback, uint32_t interval_ns) override {
    _timer = CreateThreadpoolTimer(timer_callback, (PVOID)callback, NULL);
    ULARGE_INTEGER start{};
    start.QuadPart = (ULONGLONG)(-interval_ns / 100);
    FILETIME ft{};
    ft.dwHighDateTime = start.HighPart;
    ft.dwLowDateTime = start.LowPart;
    const auto interval = interval_ns / 1000 / 1000;
    SetThreadpoolTimer(_timer, &ft, std::max(1u, interval), 0);
  }

  void stop() override {
    if (_timer == nullptr) return;
    CloseThreadpoolTimer(_timer);
    _timer = nullptr;
  }

  ~ThreadPoolTimer() { stop(); }

 private:
  static void NTAPI timer_callback(PTP_CALLBACK_INSTANCE, PVOID Context, PTP_TIMER) { reinterpret_cast<T*>(Context)->callback(); }

  PTP_TIMER _timer;
};
```

## (Windows) Waitable Timerを使う

Waitable Timerは上記3つと異なり単位指定が$\SI{100}{ns}$単位になっている.
ただし, 実際には$\SI{500}{us}$が限界のようだ (後述).

Waitable Timerを使う場合には, `FILETIME`構造体の時刻表現に従う時刻まで`WaitForSingleObject`で待つことになる.

ただし, `SetWaitableTimer`の引数には`FILETIME`構造体ではなく`LARGE_INTEGER`を渡す必要がありややこしい.

また, 今回は使用していないが, マイナスの引数を渡すと相対時刻を表すことになる.

```cpp
template <typename T>
class WaitableTimer : public Timer<T> {
 public:
  WaitableTimer() : _thread_running(false), _timer(nullptr) {}

  void start(T* callback, uint32_t interval_ns) override {
    _timer = CreateWaitableTimer(nullptr, TRUE, nullptr);

    const LONGLONG interval = static_cast<LONGLONG>(interval_ns) / 100;
    _thread_running = true;
    _timer_callback = std::thread([this, interval, callback]() {
      FILETIME system_time;
      GetSystemTimePreciseAsFileTime(&system_time);
      LARGE_INTEGER next_time = *reinterpret_cast<LARGE_INTEGER*>(&system_time);
      while (_thread_running) {
        next_time.QuadPart += interval;
        SetWaitableTimer(_timer, &next_time, 0, nullptr, nullptr, FALSE);
        if (WaitForSingleObject(_timer, INFINITE) != WAIT_OBJECT_0) break;
        callback->callback();
      }
    });
  }

  void stop() override {
    if (!_thread_running) return;
    _thread_running = false;
    if (_timer_callback.joinable()) _timer_callback.join();
    CloseHandle(_timer);
    _timer = nullptr;
  }

  ~WaitableTimer() { stop(); }

 private:
  std::thread _timer_callback;
  bool _thread_running;
  HANDLE _timer;
}
```

## (Windows) PerformanceCounterとsoftware sleepを使う


Windowsでどうしても$\SI{500}{us}$以上の精度が欲しい場合は, Software Sleepを使うしかない.

ただし, 当然のことながら負荷は大きい.
その代わり, $\SI{1}{us}$程度の精度まで出すことができる. (そこまで求めるならWindowsを使用すべきではないが.)

以下では, `QueryPerformanceCounter`で時刻測定をしているが, `std::high_resolution_clock`でもいいかもしれない.

```cpp
template <typename T>
class PerformanceCounterSoftSleepTimer : public Timer<T> {
 public:
  PerformanceCounterSoftSleepTimer() : _thread_running(false) {}

  void start(T* callback, uint32_t interval_ns) override {
    LARGE_INTEGER f;
    QueryPerformanceFrequency(&f);
    LONGLONG interval = (f.QuadPart / 1000LL / 1000LL * interval_ns) / 1000LL;
    _thread_running = true;
    _timer_callback = std::thread([this, interval, callback]() {
      LARGE_INTEGER next_time, now;
      QueryPerformanceCounter(&next_time);
      while (_thread_running) {
        next_time.QuadPart += interval;
        while (true) {
          QueryPerformanceCounter(&now);
          if (now.QuadPart >= next_time.QuadPart) break;
        }
        callback->callback();
      }
    });
  }

  void stop() override {
    if (!_thread_running) return;
    _thread_running = false;
    if (_timer_callback.joinable()) _timer_callback.join();
  }

  ~PerformanceCounterSoftSleepTimer() { stop(); }

 private:
  std::thread _timer_callback;
  bool _thread_running;
};
```

## (Linux) POSIX Timerを使う

LinuxというかPOSIX準拠のOSならPOSIX Timerが使える.

こちらは, 周期は$\SI{1}{ns}$単位で指定できる.

```cpp
template <typename T>
class POSIXTimer : public Timer<T> {
 public:
  POSIXTimer() : _timer(nullptr) {}

  void start(T* callback, uint32_t interval_ns) override {
    struct itimerspec itval;
    struct sigevent se;

    itval.it_value.tv_sec = 0;
    itval.it_value.tv_nsec = interval_ns;
    itval.it_interval.tv_sec = 0;
    itval.it_interval.tv_nsec = interval_ns;

    memset(&se, 0, sizeof(se));
    se.sigev_value.sival_ptr = callback;
    se.sigev_notify = SIGEV_THREAD;
    se.sigev_notify_function = _callback;
    se.sigev_notify_attributes = NULL;

    timer_create(CLOCK_REALTIME, &se, &_timer);
    timer_settime(_timer, 0, &itval, NULL);
  }

  void stop() override {
    if (_timer == nullptr) return;
    timer_delete(_timer);
    _timer = nullptr;
  }

  ~POSIXTimer() { stop(); }

 private:
  static void _callback(union sigval sv) { reinterpret_cast<T*>(sv.sival_ptr)->callback(); }

  timer_t _timer;
};
```

## (mac) GCD Timerを使う

macの場合はGrand Central Dispatch (GCD) のTimerが使える.

これも, 周期は$\SI{1}{ns}$単位で指定できる.

```cpp
template <typename T>
class GCDTimer : public Timer<T> {
 public:
  GCDTimer() : _queue(nullptr), _timer(nullptr), _stopped(false) {}

  void start(T* callback, uint32_t interval_ns) override {
    _queue = dispatch_queue_create("timerQueue", 0);

    _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, _queue);
    dispatch_source_set_event_handler(_timer, ^{
      _callback(callback);
    });

    dispatch_source_set_cancel_handler(_timer, ^{
      dispatch_release(_timer);
      dispatch_release(_queue);
    });

    dispatch_time_t start = dispatch_time(DISPATCH_TIME_NOW, 0);
    dispatch_source_set_timer(_timer, start, interval_ns, 0);
    dispatch_resume(_timer);
    _stopped = false;
  }

  void stop() override {
    if (_stopped) return;
    dispatch_source_cancel(_timer);
    _stopped = true;
  }

  ~GCDTimer() { stop(); }

 private:
  void _callback(T* ptr) { ptr->callback(); }

  dispatch_queue_t _queue;
  dispatch_source_t _timer;
  bool _stopped;
};
```

# 精度比較

以下に, 上記方法で色々な周期を指定した場合の精度を比較する.

精度比較は以下のようなコールバック構造体を用意して, 1000回`callback`が呼ばれるまで待ち, `callback`の呼び出し時刻を計測した.

```cpp
struct Callback {
  std::vector<int64_t> stats;
  std::chrono::time_point<std::chrono::high_resolution_clock> begin;

  void init() {
    stats.clear();
    stats.reserve(ITERATION);
    begin = std::chrono::high_resolution_clock::now();
  }

  void callback() {
    const auto now = std::chrono::high_resolution_clock::now();
    const auto dur = std::chrono::duration_cast<std::chrono::nanoseconds>(now - begin);
    stats.emplace_back(dur.count());
  }
};
```

なお, Windowsでは`timeBeginPeriod(1)`を呼び出している.

完全な計測用のコードは[GitHub](https://github.com/sssssssuzuki/periodic-timer-sample/blob/master/bench/main.cpp)を参照されたい.

## 周期16.666667ms (60 FPS)

周期に$\SI{16.666667}{ms}$を指定した場合の呼び出し時刻のグラフが以下になる.

グラフの対角線上に乗っていれば理想的である.

- Windows

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_with_timebeginperiod/16.666667ms.svg)

    - MultiMedia Timer, TimerQueue Timer, ThreadPool Timerは単位指定が$\SI{1}{ms}$なので周期が$\SI{16}{ms}$でちょっと短くなっている.

- Linux

    ![linux60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/linux/16.666667ms.svg)

- macOS

    ![mac60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/mac/16.666667ms.svg)

## 周期1ms (1000 FPS)

周期に$\SI{1}{ms}$を指定した場合の呼び出し時刻のグラフが以下になる.

- Windows

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_with_timebeginperiod/1.0ms.svg)

    - ThreadPool Timerは$\SI{16}{ms}$未満を指定しても意味がないらしい. なお, TimerQueue TimerもThreadPool Timerに隠れて見えていないが同様である.

- Linux

    ![linux60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/linux/1.0ms.svg)

- macOS

    ![mac60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/mac/1.0ms.svg)

## 周期500us (2000 FPS)

周期に$\SI{500}{us}$を指定した場合の呼び出し時刻のグラフが以下になる.

- Windows

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_with_timebeginperiod/500.0us.svg)

    - Waitable Timerはこのあたりまで対応できる.

- Linux

    ![linux60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/linux/500.0us.svg)

- macOS

    ![mac60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/mac/500.0us.svg)

## 周期100us (10000 FPS)

周期に$\SI{100}{us}$を指定した場合の呼び出し時刻のグラフが以下になる.

- Windows

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_with_timebeginperiod/100.0us.svg)

    - ThreadPool TimerとTimerQueue Timerは論外で, MultiMedia Timerも$\SI{1}{ms}$単位なので使えない. `std::this_thread::sleep_for`とWaitable Timerを使った場合は, かなりガタつくが意外と頑張ってる. PerformanceCounterとSoftware Sleepはまだまだ大丈夫そう.

- Linux

    ![linux60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/linux/100.0us.svg)

- macOS

    ![mac60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/mac/100.0us.svg)


LinuxとmacOSはこの程度なら全く問題ない.

## 周期1us (1000000 FPS)

周期に$\SI{1}{us}$を指定した場合の呼び出し時刻のグラフが以下になる.

ここまで行くとさすがにどれも厳しそう, まあここまで使うことはないと思うけど.

- Windows

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_with_timebeginperiod/1.0us.svg)

    - PerformanceCounterとSoftware Sleepの組み合わせはかなり頑張ってるが, 途中ちょっとおかしくなる.

- Linux

    ![linux60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/linux/1.0us.svg)

- macOS

    ![mac60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/mac/1.0us.svg)

# まとめ

基本的に, [ループとstd::this_thread::sleep_untilの組み合わせ](#ruputostd-this-thread-sleep-untilnozu-mihe-wase)を使用すればいい.

ただし, Windowsかつ$\SI{0.5}{ms}$以上の高精度がほしいなら, 負荷を承知でSoftware Sleepを使う.

また, 実際にはコールバック関数で指定する処理の重さとか, PCにかかっている負荷とか, ハードウェアとかによっても変わってくるので各自の環境で実際に調べることをおすすめする.

# おまけ (timeBeginPeriodを呼ばない場合)

以下に, `timeBeginPeriod`を呼ばなかった場合のデータを載せておく.

- $\SI{16.666667}{ms}$

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_without_timebeginperiod/16.666667ms.svg)

- $\SI{1}{ms}$

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_without_timebeginperiod/1.0ms.svg)

- $\SI{500}{us}$

    ![win60fps](https://raw.githubusercontent.com/sssssssuzuki/sssssssuzuki.github.io/master/content/fig/blog/timer/win_without_timebeginperiod/500.0us.svg)

$\SI{1}{ms}$あたりから, `std::this_thread::sleep_until`とWaitable Timerを使用する場合の呼び出し間隔がガタつき始める.

興味深いことに, MultiMedia Timerとかには影響がない.

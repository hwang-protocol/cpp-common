# cpp-common —— 共用的 header-only 库

| 文件 | 内容 | Java 对应 | 用在 |
|---|---|---|---|
| `bounded_queue.h` | 有界阻塞队列 (MPMC)，满了阻塞生产者 = 背压；`close()` 后消费者取完剩余数据再退出 | `ArrayBlockingQueue` | 线程池、model-rollout 审批队列 |
| `thread_pool.h` | 固定线程数 + 有界任务队列，`submit` 返回 `std::future`，析构时优雅关闭 | `ThreadPoolExecutor` + `CallerRunsPolicy` | 测试 |
| `spsc_ring.h` | 无锁单生产者单消费者环形缓冲；head/tail 放在不同 cache line；acquire/release | LMAX Disruptor / JCTools | pretrade-risk 网关 → 分片 |
| `latency_histogram.h` | 对数分桶直方图，固定内存，O(1) 记录，误差 < 6.25%；可以合并 | HdrHistogram | market-data-pipeline、pretrade-risk |
| `sqlite.h` | `Db` / `Stmt` / `Transaction` 的 RAII 封装 (`unique_ptr` + 自定义 deleter)；事务对象析构时如果还没 commit 就自动回滚 | JDBC + try-with-resources | market-data-pipeline、post-trade-workflow、tick-validation |
| `cli.h` | `--key value` 参数解析 | | 全部 |
| `check.h` | 极简测试宏 | JUnit `assert*` | 全部测试 |

## 在其它项目里使用

用 CMake 的 FetchContent 拉取，**固定到某个 commit SHA**（不用分支名，理由和固定 GitHub Action 一样：分支可以被改，SHA 不能）：

```cmake
include(FetchContent)
FetchContent_Declare(cpp_common
    GIT_REPOSITORY https://github.com/hwang-protocol/cpp-common.git
    GIT_TAG        <commit SHA>)
FetchContent_MakeAvailable(cpp_common)

target_link_libraries(my_app PRIVATE cpp_common::common)   # 只要线程相关的
target_link_libraries(my_app PRIVATE cpp_common::sqlite)   # 还要 SQLite 封装
```

`cpp_common::sqlite` 只在找到 SQLite 时才存在（macOS 自带；Ubuntu 需要 `apt install libsqlite3-dev`）。

## 构建和测试

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
ctest --test-dir build
```

测试在 `tests/common_tests.cpp`：多生产者多消费者求和、SPSC 保序、线程池、直方图精度、事务回滚。

## CI

[ci.yml](.github/workflows/ci.yml) 调用中央流水线 [devsecops-pipelines](https://github.com/hwang-protocol/devsecops-pipelines)：
密钥扫描、Semgrep、CodeQL，以及在 Ubuntu + clang-18 上分别做普通、ASan+UBSan、TSan 三种构建测试。

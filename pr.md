# PR 草稿：LevelDb 打开损坏 userdb 时自动尝试修复（Recover）

> 状态：**草稿，待审阅**。确认后再推分支到 fork 并向上游开 PR。
> 建议分支名：`fix/userdb-corrupt-auto-recovery`
> 目标仓库：rime/librime（master）
> 本地补丁：`../fix-userdb-auto-recovery.patch`（+47/-0，仅两个文件）
> Fork 仓库：https://github.com/takomitkm/librime

---

## 摘要

librime 的自动恢复只挂在"会话挂载方案词典"（`UserDictionary::Load()`）这一条路径上。当 db 从不挂载、只被依赖组件打开时，恢复永远不会触发，日志会不断刷错。

本案例：`moran_pin.userdb` 损坏于 2026-06-04。它不在 schema_list 里，只被 moran_fixed/moran_aux 的 lua 组件打开，每个会话失败 3 次、刷 3 条 ERROR，单日累计 237 条，四个月没触发过一次修复。

修复方式应当二选一：

- **方案 A（本 PR）**：`Open()` 失败且 `IsCorruption()` 时自动 RepairDB 修复并重试。db 层生效，覆盖所有调用方，数据能救回，日志只剩一条 WARNING；
- **方案 B**：不修，但每个会话只报一次错（或静默跳过）。数据仍然不可用。

现状（反复重试 + 反复刷错）不可接受。本 PR 实现的是 A。

---

## 标题（建议）

```
fix(level_db): try to recover corrupted user db on open failure
```

（中文说明用：librime 打开损坏 userdb 失败后既不修复也不隔离：CURRENT 悬空导致每次会话重复刷 Corruption ERROR）

---

## 背景：一个真实损坏案例

- 环境：Windows 11 x64，小狼毫 beta 构建（二进制 2026-10-12 打包、2026-10-21 安装，版本资源 0.17.4.0）
- `moran_pin.userdb`（明月拼音·简化字）自 **2026-06-04** 起无法打开，glog 每个会话重复刷同一条 ERROR（实测单会话 168→237 条并随使用持续增长），该词典用户数据完全不可用，界面无任何提示：

```
E20261029 18:05:02.xxx 16208 level_db.cc:259] Error opening db 'moran_pin': Corruption: CURRENT points to a non-existent file: NotFound: C:\Users\takomi\AppData\Roaming\Rime\moran_pin.userdb/MANIFEST-000043: The system cannot find the file specified.
```

- 现场证据（`%APPDATA%\Rime\moran_pin.userdb\`）：
  - `CURRENT`（2026-02-05 写入，内容 `MANIFEST-000043`）指向**已不存在**的文件；
  - 目录中实际存在更新一代的 `MANIFEST-000315` 与空的 `000316.log`（均 2026-06-04 12:31）；
  - LevelDB 自身的 `LOG` 为 0 字节 → 每次 Open 都在读取 CURRENT 的最初阶段失败，**恢复 / RepairDB 流程从未被触发**；
  - 三个 `.ldb`（2026-01 / 03 / 05 的用户数据）完好但取不出来。

- 推断：2026-06-04 一次打开/恢复中，LevelDB 已写出新 MANIFEST-000315 并新建 000316.log，但在最后一步**原子改写 CURRENT 之前进程中断**；此后 CURRENT 悬空，每次 Open 必然在 Recover 阶段失败。

## 根因

1. LevelDB 打开序列 = 写新 MANIFEST →（新建 log）→ 原子改写 CURRENT。中断窗口落在最后一步时，CURRENT 指向已被清理删除的旧 MANIFEST，此后每次 Open 必然失败；
2. 自动恢复接线只在 `UserDictionary::Load()`（`user_dictionary.cc:171`）生效：`Open()` 失败后调度 `userdb_recovery_task`（走 `Recover()`/`RepairDB`），且维护线程运行期间跳过。moran_pin 不在 schema_list，只被 moran_fixed/moran_aux 的 lua 组件（`moran_pin*panacea_translator` 等）打开，走不到这条路径，所以失败 3 次/会话，恢复 0 次；
3. `LevelDb::Open()` 失败只刷一行 ERROR，下个会话再试再刷。本 PR 把修复下沉到 `LevelDb::Open` 层，任何调用方都能触发。

## 改动（+47/-0）

### src/rime/dict/level_db.cc（+11）

```cpp
  if (!loaded_ && status.IsCorruption()) {
    // The db may be corrupted by an interrupted write, leaving CURRENT
    // dangling. Try to salvage the data before giving up.
    LOG(WARNING) << "db '" << name() << "' is corrupted: " << status.ToString()
                 << "; trying to recover.";
    if (Recover()) {
      status = db_->Open(file_path(), readonly_);
      loaded_ = status.ok();
    }
  }
```

### test/user_db_test.cc（+36）

新增 `RimeUserDbTest.RecoverFromCorruptedDb`：造库 → 写入两条数据 → 关闭 → 把 CURRENT 改写为指向不存在的 MANIFEST → 重新 `Open()`，断言打开成功且两条数据原样取回。测试通过 `UserDb::Require("userdb")` 组件 + 基类指针访问（原因见"发现 2"）；fixture 用 `std::filesystem::remove_all` 保证可重跑（原因见"发现 3"）。

## 设计取舍

1. **只在 `status.IsCorruption()` 时触发**：锁冲突等其他失败是 IO error，不会触发修复——天然规避两个进程同时打开同一 userdb 的竞态（Weasel 多会话 / 管理员实例场景本机实测存在）；
2. **只影响 userdb 组件**（`LevelDb` 在全工程仅被 `UserDbWrapper<LevelDb>` 继承）；`OpenReadOnly()` 保持原样、无副作用（有专项测试验证只读路径优雅失败）；
3. **修复失败不自动删库**：改名隔离 + 重建 + 快照还原的完整链路已在 `UserDbRecoveryTask` 中存在，是否在维护/部署流程中接线该任务留作开放问题（见下）；
4. 行为变化对用户可见的效果：损坏的 userdb 在下一次打开时自动被 RepairDB 修复，数据尽量保留，日志从"每会话数百条 ERROR"变为一条 WARNING（修复成功时）。

## 希望维护者关注的几个发现（开放问题）

1. **恢复接线只覆盖会话挂载路径**：`userdb_recovery_task` 由 `UserDictionary::Load()` 触发，不挂载的 userdb（如本案例的 moran_pin）够不着。本 PR 在 Open 层补上缺口；要不要再加维护期全量巡检，留待讨论；
2. **`LevelDb` 没有任何 `RIME_DLL` 导出标注**（对比 `TextDb` 全量标注）：Windows 上无法直接实例化测试。尝试对齐 TextDb 风格逐方法标注后，MinGW 链接仍失败——`LevelDb` 多继承 `Recoverable`/`Transactional`，按方法 dllexport 导不出 secondary-base 的 `non-virtual thunk`（`non-virtual thunk to rime::LevelDb::Recover()` 等 undefined reference）；`TextDb` 能直接实例化是因为它单继承。本 PR 的测试因此走组件 + 虚调用（头文件零改动）。若希望 LevelDb 可直接实例化测试，可能需要类级导出或对 `UserDbWrapper<LevelDb>` 显式实例化导出，请维护者定夺；
3. **`DestroyDB` 不递归删除 `lost/` 子目录**：RepairDB 会把无法归档的旧文件移入 `lost/`，导致 `LevelDb::Remove()` 对修复过的库失效（目录残留、`Exists()` 仍为 true）。测试 fixture 因此用 `std::filesystem::remove_all`；生产路径中 `Remove()` 的调用方是否受影响，值得留意；
4. **缺 CURRENT 的场景 leveldb 1.23 会自愈**：实测删除 CURRENT 后 `Open()` 成功且数据可读（压力测试日志可证）；真正无法自愈的正是本 PR 针对的"CURRENT 悬空指向不存在文件"这一更糟的状态；
5. **并发修复无锁保护**：两进程同时判定 Corruption 并发 `RepairDB` 理论上可能（窄窗口）。当前以 `IsCorruption()` 门控已排除最常见的锁冲突误触发，但如需彻底防护，是否在 RepairDB 前后持 LOCK 文件，请维护者权衡；
6. **同类引擎对照**：fcitx5/libime 无自动修复（社区做法是删库重建或经 `libime_pinyindict` 导出文本重建）；mozc 依赖 SQLite 抗损坏，坏库同样需手工 `.recover` 或删库。librime 的修复机制本已齐备，本 PR 只是补上"接线"这一步。

## 验证

1. **机制验证（真实损坏数据，零丢失）**：WSL Ubuntu + leveldb 1.23，对真实损坏的 `moran_pin.userdb` 副本：复现同款错误 → `IsCorruption()==true` → `RepairDB` OK → 重开成功 → 5 条记录（4 条元数据 + 1 条用户词条）全部保留，`/db_name == moran_pin`；
2. **编译 + 全量测试**：Windows 11 x64，MSYS2 UCRT64（gcc 16.2.0 / cmake 4.4.5 / ninja 1.13.2 / gtest 1.18 / leveldb 1.23 / glog 0.7.1 / boost 1.92 / marisa 0.3.1 / opencc 1.4.2 / yaml-cpp 0.9.0）：
   - 终版补丁：**146/146 tests passed**（含新增用例，ctest 与直接运行双验证）；
   - 压力测试（临时用例，验证后未包含在本 PR）：**151/151 passed**；
3. **压力 / 边界场景**（临时用例验证）：
   - 50 轮"损坏 → 自动修复 → 数据校验"循环（同进程、数据累积）：11.4s 全过；
   - CURRENT 写入垃圾二进制（无换行）：修复成功、数据保留；
   - CURRENT 整个删除：Open 成功且数据存活（leveldb 自愈，见发现 4）；
   - 损坏库上 `OpenReadOnly()`：优雅失败（ERROR 一条，不崩溃），随后可写打开自动恢复；
   - 5000 条数据损坏 → 修复 → 全量点数：5000/5000，326ms；
4. clang-format（仓库自带 `.clang-format`，Chromium 风格）对改动文件全量达标；
5. MinGW/clang 工具链全量验证；**MSVC 未本地验证**（无管理员权限装 Build Tools），依赖 CI——测试特意使用 MSVC 兼容写法（无 MinGW 专属 API）。

## 已知局限（如实）

1. 仅 `IsCorruption()` 触发：`.ldb` 内部损坏但 CURRENT 完好时，`Open()` 不报错（查询期才失败），Open 层面无信号，本补丁无法覆盖；
2. `RepairDB` 本身失败时，回落到现状（继续刷 ERROR）——完整恢复链路（改名隔离 + 重建 + 快照还原）在 `UserDbRecoveryTask` 中已存在，接线与否留作开放问题 1；
3. 并发 RepairDB 的理论竞态未加锁（见开放问题 5）；
4. 测试对 `lost/` 的清理依赖 `std::filesystem::remove_all`（`LevelDb::Remove()` 对修复过的库失效，见发现 3）。

## 附件清单（提交 PR 时附上）

- `fix-userdb-auto-recovery.patch`（+47/-0）
- `repair_validation.txt`（WSL 对真实损坏副本的验证记录）
- `rime_test_final.log`（校机 146 绿）
- `rime_test_stress2.log`（校机 151 绿，含压力用例与 `[stress]` 观察输出）
- 匿名化的 `moran_pin.userdb.anon.zip`（复现样本：`user_id` 与用户词条已删除，保留"悬空 CURRENT + 新 MANIFEST 在场无引用"的现场结构，解包放入用户目录即可复现；含个人数据的原版样本未公开，仅在本地留存）

## 提交信息（建议）

```
fix(level_db): try to recover corrupted user db on open failure

When LevelDb::Open() fails with a Corruption status, run the existing
Recover() (leveldb::RepairDB) and retry before giving up.

A user db may be corrupted by an interrupted write: when a new manifest
is in place but CURRENT still refers to the removed one, every open
fails in the recovery stage and spams the log with hundreds of errors
per session, while the user data stays unusable without any visible
hint. RepairDB rebuilds CURRENT from the existing .ldb/.log files and
preserves the data.

Also add a regression test that simulates a dangling CURRENT and
expects the db to be recovered with data preserved.

Note: UserDbRecoveryTask ("userdb_recovery_task") is registered but
never invoked anywhere; wiring it into the maintenance flow is left
for further discussion.
```

## 提交步骤（待确认后执行）

1. `git checkout -b fix/userdb-corrupt-auto-recovery`
2. `git add src/rime/dict/level_db.cc test/user_db_test.cc && git commit`（用上方提交信息；`pr.md` 不入库）
3. `git remote add fork https://github.com/takomitkm/librime.git && git push fork fix/userdb-corrupt-auto-recovery`
4. 向 rime/librime 开 PR（标题与正文取自本文档）

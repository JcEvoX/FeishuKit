# 贡献指南

欢迎 PR，尤其是**适配新飞书 / Lark 版本**。这是本项目最需要社区力量的地方 —— 维护者一个人跟不动飞书的更新节奏。

## 底线（提 PR 前务必确认）

**绝不提交以下内容**（`.gitignore` 已尽量拦截，但请自查）：

1. **飞书 / Lark 的二进制或反编译产物** —— `liblark.so`、IDA `.i64`/`.idb`、jadx dump、`base-*.apk` 等。版权风险 + DMCA 触发点。用文字描述"哪个函数 / 第几列 / 什么偏移"即可。
2. **签名私钥** —— `*.keystore` / `*.jks`。
3. **抓取到的真实数据** —— 花名册 JSON、他人 profile、聊天导出。
4. **任何真实身份信息** —— 真名、真实手机号/邮箱、真实 chat_id / user_id / tenant_id。请用 `<PLACEHOLDER>` 或明显的占位符。

## 怎么给新飞书版本重新定位偏移

防撤回走 native SQL 层，抗混淆能力强；防已读走 Java + native，混淆一变就容易崩。通用流程：

1. **拿目标版本 APK** 本地分析（**不入库**）。抽出 `lib/arm64-v8a/liblark.so`。
2. **静态定位**：用 IDA / radare2 搜锚点字符串（如 SQL 语句、`is_recalled`、回执 RPC 名），从字符串交叉引用回溯到目标函数。
3. **动态验证**：`frida/` 和 `stealth-read/` 下有现成探针脚本。attach 到飞书，hook 候选函数，实测触发（发/撤回一条测试消息、打开一个会话）确认命中。
4. **落地**：把新偏移 / 新混淆符号写进对应 hook（`native/jni/` 或 `app/.../*.java`），做**版本自适应**（按 `versionCode` 分支，别写死一个版本）。
5. **验证矩阵**：注明你测过的飞书版本号 + 设备 / Android 版本，PR 描述里贴上。

### 先查建表序，再动列位（重要）

`messages` / `feed_channel` 的**列序会随版本变**（例：7.72.10 把 `messages.content` 从第 9 位移到第 6 位）。
这类改动**不需要 IDA**，直接在 `liblark.so` 里搜建表语句就能拿到权威列序：

```bash
# 在 liblark.so 里定位建表 SQL（附带完整列定义，顺序即 0-based 列位）
python3 - <<'PY'
import re
raw = open('liblark.so','rb').read()
for tbl in (b'messages', b'feed_channel'):
    m = re.search(rb'CREATE TABLE\s+`?' + tbl + rb'`?\s*\(', raw)
    if m:
        s = raw[m.start():m.start()+4000].decode('utf-8','replace')
        s = s[:s.find(');')]
        cols = re.findall(r'^\s*,?\s*([a-zA-Z_]\w*)\s+(BIGINT|INTEGER|TEXT|BLOB|BOOLEAN|REAL|DOUBLE)', s, re.M)
        print(f'--- {tbl.decode()} ({len(cols)} 列) ---')
        for i,(n,t) in enumerate(cols): print(f'{i:3d}  {n:28s} {t}')
PY
```

拿到列位后：
- **优先**依赖运行时的**列名表解析**（`REPLACE INTO \`messages\` (\`id\`,\`chat_id\`,…)`）—— 它自动适配，无需改代码；
- 只有当解析**失败**才用兜底常量。新版本建表序与现有兜底不同时，**补一组 `COLS_*` / `FEED_*` 常量并按 `versionCode` 选中**
  （见 `native/jni/antirecall.cpp` 的 `COLS_V7` / `COLS_V77210` / `pick_fallback_cols()`），别直接改掉旧常量导致老版本回归。

### 白标包名

企业二次开发 / 白标版会改 `applicationId`（例：`com.ss.android.lark.saxmsa667`），
`isLarkFamily()` 认不出。确认 `isLarkApp()` 的类探测（`com.bytedance.lark.sdk.Sdk` /
`com.bytedance.lark.pb.basic.v1.Command`）在目标包里存在即可，无需为每个白标加包名白名单。

## 代码规范

- 保持现有风格；注释可中文。
- 新功能优先做成**按版本自适应**，不要为单一版本写死。
- 逆向结论 / 踩坑复盘欢迎补进对应的 `*.md`，方便下一个人跟版本。

## 提交

- 个人 fork → PR。commit message 讲清"改了什么 + 在哪个飞书版本上验证过"。

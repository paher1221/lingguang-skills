---
name: omnibot-customize
description: 维护用户的 OmniBot 安卓应用个人定制（paher1221/OmniBot，my-version 分支）。删除或下架智能体、调整启动行为、精简设置模块、触发 GitHub Actions 出包并交付安装包时使用；上游同步与新增定制也适用。
---

# OmniBot 个人定制维护

## 仓库与环境

- 上游：`omnimind-ai/OmniBot`；个人仓库：`paher1221/OmniBot`（fork）。
- 本地仓库：`/root/workspace/OmniBot`，分支 `my-version` 保存全部定制；`main` 仅作上游镜像，同步用 merge。
- Remote URL 内嵌 PAT，勿泄露、勿写进任何文件或输出。提取方式（/tmp/pat.txt 会被清理，每次重新提取）：
  ```bash
  cd /root/workspace/Eta && git remote get-url origin | sed 's#https://paher1221:##; s#@github.com.*##' > /tmp/pat.txt
  ```
- 架构：原生 Android 壳 + Flutter 界面（`ui/` 是 Flutter 模块）。界面残留去 `ui/lib`（Dart）找，底层遗留去 `app/`、`baselib/`、`assists/`（Kotlin）找。

## 智能体体系（改动的核心知识）

- 智能体清单：`app/src/main/assets/acp/agents.json`。当前保留：小万（xiaowan-acp）、Codex、OpenCode、Pi。
- **下架智能体的正确方式**：把 id 加进 `retiredAgentIds` 并删除对应条目。运行时自动清理存量配置、界面列表消失、旧对话打开时返回友好的 `agent_retired` 提示且历史保留。不要硬删运行时代码——Na 牵扯 30+ 文件，深度耦合聊天/通知/历史同步。
- 界面智能体列表来自 `agent/list` → `AcpAgentProfileStore.catalogPayload()`，按退役名单过滤。
- Kimi Code Web / OpenCode Web 是插件不是智能体：`app/.../bot/plugin/official/agentweb/`（AgentWebPlugin.kt 定义工具与动作，AgentWebRuntimeModels.kt 定义服务与启动配置，AgentWebRuntimeManager.kt 解析就绪 URL）。
- 终端环境安装入口在 `ui/lib/features/home/pages/termux_setting/termux_setting_page.dart` 的 `_EnvironmentDefinition` 列表；原生侧 `EnvironmentSetupLogic.kt` 全是 `in requested` 条件判断，只删 Dart 条目是安全的。

## 启动默认智能体（2026-11-24 新增功能）

- 设置项：设置 → 杂项 → "启动时" → "进入指定智能体"，选中后出现智能体下拉框。
- 数据：`ChatStartupBehavior.specifiedAgent`（`ui/lib/models/chat_startup_behavior.dart`）+ `StorageService.kChatStartupAgentIdKey`。
- 生效点只有两处（2026-11-25 用户明确：**设置只管打开 App 时默认进哪个智能体，新建对话必须跟随左上角当前选中的智能体，不得读启动设置**）：
  1. `chat_page_lifecycle.dart` `_resolveConversationThreadTarget`：无传入目标时按设置开新对话。
  2. 左上角选择器默认值：`home_drawer.dart` `_homeFilter` 初始为 xiaowan；`home_drawer_home.dart` `_readHomeAgentCatalog` 首次加载目录时用 `_pendingHomeStartupAgentId` 应用指定智能体。注意 `_homeFilters` 会过滤掉未安装（installed != true 或 status == missing）的智能体，未安装的回落小万。
- `home_drawer_actions.dart` `_openNewConversation` 保持原样（用 `_homeFilter.agentId`，即当前选中智能体）。曾按启动设置覆盖（17ccb1e02），用户反馈行为不对，已 revert（678675936）。

## 已完成的定制（提交记录）

- `f359c9089`：退役 Claude Code / Kimi Code / Na 三个智能体；删除 Kimi Code Web 全套；终端环境页移除 claude_code、kimi 安装入口。
- `c2b92a1a5`：删除设置页与"我的"页的"账号与 AI 服务"入口（保留登录页给首次引导）；新增启动默认智能体设置。
- `17ccb1e02`：首页加号新建对话遵循启动智能体设置（后被 678675936 revert，见上）。
- `0ce89a031`：打开 App 时左上角选择器默认落在指定智能体。
- `678675936` + `4ca0611ee`：revert 17ccb1e02；revert 顺带删掉了 `home_drawer.dart` 的 `chat_startup_behavior.dart` import，但 0ce89a031 的选择器逻辑仍需要它，补回后构建通过。

## 构建与发版

- 沙箱 JVM 被代理重签 TLS，Gradle 不可用，一律走 GitHub Actions。
- 出包用 `release.yml`（只编译不跑测试），手动触发：
  ```bash
  curl -s -X POST -H "Authorization: token $PAT" \
    -H "Accept: application/vnd.github+json" \
    https://api.github.com/repos/paher1221/OmniBot/actions/workflows/release.yml/dispatches \
    -d '{"ref":"my-version"}'
  ```
- fork 仓库 Actions 默认关闭，首次需开启；签名密钥已写入仓库 Secrets（RELEASE_KEYSTORE/RELEASE_PASSWORD/RELEASE_KEY_ALIAS），个人密钥备份在 `/root/workspace/keystore-backup/omnibot-release.jks`。
- 构建约 25-35 分钟。盯梢方式：后台循环查 `GET /repos/paher1221/OmniBot/actions/runs/<id>`，status 变 completed 即结束。
- `ci.yml` 会跑 `flutter test`、`flutter analyze` 和 Gradle 单测——改公共代码时注意测试引用（如 ui/test 里大量 kimi 引用），但出包不受影响。

## 踩坑记录

- **删除代码必须逐符号 grep 复核**：曾误删 `assists/api/bean/LMBean.kt`（其 ResultBean 被 HttpController 使用）导致构建失败。教训：批量"核实"被拒绝后重跑单个命令，不等于整批已验证。
- 界面引用智能体 id 的地方多是条件判断（`== 'na-cloud'`），清单删掉后不触发，安全；但 `agent_brand_icon.dart`、`home_drawer_conversation_list.dart` 里的映射要保留（旧对话展示用）。
- `model_vendor_catalog.dart` 里的 kimi/moonshot 是模型供应商（Moonshot API），与智能体无关，不要删。
- 签名与官方版不同，装包前必须卸载原版。
- **revert 提交后必须全量编译验证**：revert 可能删掉仍被其他提交使用的 import/引用（如上例），也可能遇到 Maven Central 429 限流导致构建在依赖解析阶段就挂（日志特征：`Received status code 429`，30 秒内失败），限流类失败直接重新触发即可。
- 构建日志下载：`GET /actions/runs/<id>/logs` 需加 `-L` 跟随 302 跳转，否则拿到空文件。
- APK 交付：artifact 下载的 zip 实为 APK 本体，改名 `.apk` 后再包一层 zip 交付（publish_file 不支持 APK）。

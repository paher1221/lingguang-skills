---
name: eta-fork-sync
description: 维护用户的 Eta 安卓应用个人分支（paher1221/Eta，my-version 分支）。上游仓库更新后同步代码、重打个人定制补丁、触发构建并交付安装包时使用；也适用于在该仓库新增或修改功能后发版。
---

# Eta 个人分支维护

## 仓库与环境

- 本地仓库：`/root/workspace/Eta`，分支 `my-version`。
- Remote URL 内嵌 PAT，勿泄露、勿写进任何文件或输出。提取方式：
  `git remote get-url origin | grep -oP '(?<=://)[^@]+(?=@)' | sed 's/^paher1221://'`
- 上游为 Eta 原项目；个人定制以补丁形式维护在 `docs/patches/`。

## 已有定制补丁

全部个人改动按主题收录在 `docs/patches/`，总索引与重打指南见 **`docs/patches/README.md`**，动手前先读它：

1. `01-strip-xposed-root-device-tools.patch`：剔除 Xposed 模块、Root 设备控制、手机控制、无障碍与 GUI 工具族。
2. `02-ci-manual-build.patch`：手动构建流程与缺签名降级。
3. `03-remove-system-assistant-overlay.patch`：删除系统助手浮窗链路（SYSTEM_ALERT_WINDOW 随之移除）。
4. `04-remove-home-suggestion-cards.patch`：移除主页空会话建议卡片。
5. `05-remove-voice-module.patch`：移除语音模块（保留 handoff 常量）。
6. `06-global-system-prompt-and-docs.patch`：系统提示词移至角色模块全局生效 + 补丁目录本身。
- `full-personal-modifications.patch`：以上全部的合并差异（分叉基线 `cf98d1e0`）。
- 每个补丁的要点、优先级规则和踩坑都写在 README 的「各补丁要点与踩坑」一节。

## 上游同步流程

1. `git remote -v` 确认上游 remote（`Mangi-11/Eta`）；没有就添加。
2. `git fetch upstream` 后合并/变基到 `my-version`；冲突时按 README 的核对清单手工恢复意图，不要盲目保一边。
3. 冲突时按序逐个 `git apply --3way docs/patches/0X-*.patch`，定位是哪个主题被上游改动波及。
4. 补丁涉及文件被上游大改时，优先按 README 要点手工重做，再重新生成补丁文件：
   `git diff <基线提交>..<功能提交> > docs/patches/<名称>.patch`，并同步更新 README 表格。
5. 推送前自查：字符串资源撇号必须转义（如 `provider\'s`）；删除 Composable 后清理未用导入。
6. 新增功能或剔除后，记得生成新补丁并更新 README，保持补丁集与分支一致。

## 构建与发版

- 本地沙箱无法完成 Gradle 构建（拉不到工具链插件），一律走 GitHub Actions。
- `android-release.yml` 只在 push 到 `main` 或 `v*` 标签时自动触发；`my-version` 分支需手动触发：
  `POST /repos/paher1221/Eta/actions/workflows/android-release.yml/dispatches`，body `{"ref":"my-version"}`（Bearer PAT）。
- 构建成功后从 run 的 artifacts 下载 APK（release 约 8.5MB，debug 约 41MB 未混淆）。
  注意：artifact 下载接口返回的 zip 实际就是 APK 本体，需先复制改名 `.apk` 再包一层 zip 交付（publish_file 不支持 APK）。
- 正式发版：递增 `versionName` → 推 `v<versionName>` 标签 → 自动出草稿 → PATCH `draft=false` 发布。
  签名用仓库已配置的 4 个 `ETA_RELEASE_*` Secrets，正式版必须沿用既有密钥。

## 交付约定

- APK 以 zip 工件卡片交付，文件名用中文描述版本与功能。
- 面向用户的答复不暴露 PAT、内部提交哈希可按需提及但不必罗列。

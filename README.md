# wda-builder

在 GitHub Actions 的 macOS runner 上编译 WebDriverAgent (WDA)，产出可在 Windows 侧安装到 iPhone 的 IPA。
适用于「无 Mac + iOS 17 及以上（含 iOS 18 / iOS 26）」场景。iOS 17 起 Apple 锁死了旧端口转发，WDA 必须走 tunnel，该要求在 18/26 上依旧成立。

## 两种工作流

- `build-wda-unsigned.yml`：产出**未签名** IPA，下载后用爱思/Sideloadly 用自己的 Apple ID 重签（账号不进 CI）。
- `build-wda-signed.yml`：云端用你的 P12 + mobileprovision 直接自签，下载即是绑了 UDID 的 IPA。

两者都用 `workflow_dispatch`，在 Actions 页面手动点 **Run workflow** 触发，无需配置触发分支。

## 方案 A（未签名）用法

1. 建空仓库推上这两个 workflow 文件。
2. Actions → Build Unsigned WDA IPA → Run workflow（可改 repo/branch/bundle_id）。
3. 下载 Artifact `WebDriverAgent-Unsigned-IPA.zip`，解压得到 `WDA-unsigned.ipa`。
4. Windows 侧用爱思/Sideloadly 选自己的 Apple ID 重签并安装到 iPhone。

## 方案 B（云端自签）前置

本地一次性准备（需要能导出 P12 的环境：Mac / 钥匙串 / 爱思）：

1. Apple Developer 后台把 iPhone **UDID** 加入设备列表（个人免费账号经 Xcode 自动签名也可生成，但需先在某处跑过一次 Xcode 签名）。
2. 建 iOS App ID（通配 `com.yourname.*` 或精确 `com.yourname.WebDriverAgentRunner`）。
3. 建 iOS Development 证书 → 下载 `.cer` → 钥匙串导出 `.p12`（设密码）。
4. 建 Development Provisioning Profile（绑证书+设备+App ID）→ 下载 `.mobileprovision`。
5. 两个文件 base64：
   - `base64 -i your.p12`
   - `base64 -i your.mobileprovision`

Repo → Settings → Secrets and variables → Actions 存四个：

- `P12_B64`：p12 的 base64
- `P12_PASSWORD`：导出 p12 时设的密码
- `PROFILE_B64`：mobileprovision 的 base64
- `WDA_BUNDLE_ID`：如 `com.yourname.WebDriverAgentRunner`

然后 Actions → Build Signed WDA IPA → Run workflow，下载 `WDA-signed.ipa` 直接装。

## 注意事项

- `macos-latest` 自带较新 Xcode，足够编 iOS 17/18 的 WDA；不要选 macos-13 等老镜像。
- iOS 17+ 必须删 `Frameworks/XC*` 文件，两个 workflow 已处理。
- 个人免费账号自签 7 天过期；CI 只负责编包，不延长有效期。换成付费证书则 yaml 不变、有效期一年。
- 不要用别人编的 IPA：provisioning 与 Team ID + UDID 强绑定，别人签的到你手机必报 Unable to install。
- iOS 17/18 在 Windows 侧启动 WDA 需要走 **go-ios tunnel**（见仓库外补充的 Windows 侧命令）。

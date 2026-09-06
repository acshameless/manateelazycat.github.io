---
layout: post
title: Firefox 插件注册、检测和自动签名方法
categories: [Tech, Web]
---

最近给 Firefox 做插件，才发现 Firefox 和 Chrome 的离线插件安装方式完全不一样。

Chrome 可以打开开发者模式，直接加载解压后的插件。Firefox 正式版和 Firefox ESR 安装 XPI 时会强制检查 Mozilla 签名，未签名的 XPI 即使代码完全正常，也会提示插件损坏或者签名无法验证。

`about:debugging` 虽然可以临时加载未签名插件，但 Firefox 重启以后插件就没了，只适合开发调试，不能用来给用户发布插件。

下面记录一下 Mozilla Firefox 插件的注册、上传检测和获取 API 信息的方法。

#### 注册 Mozilla 开发者账号

先打开 Mozilla 插件开发者中心：

https://addons.mozilla.org/developers/

点击登录，用 Mozilla Account 登录。没有账号就先注册一个，然后同意 Firefox Add-ons 的开发者协议。

Mozilla 不需要提前单独注册一个插件 ID，第一次上传插件以后，开发者中心会自动创建插件项目。但是 manifest 里面一定要写固定的 Gecko ID，例如：

```json
{
  "browser_specific_settings": {
    "gecko": {
      "id": "my-extension@example.com",
      "strict_min_version": "140.0"
    }
  }
}
```

这个 ID 相当于 Firefox 插件的身份证。后续更新版本必须继续使用同一个 ID，不能每个版本都随机换。

#### 上传插件并进行检测

登录开发者中心以后，点击“提交新附加组件”，Mozilla 会让你选择发布方式：

1. 在 Mozilla Add-ons 上发布：插件会出现在 Firefox 插件商店里，适合公开发布。
2. 自行分发：Mozilla 只负责检测和签名，签名后的 XPI 由开发者自己放到网站上给用户下载。

如果只是想在自己的网站上提供 Firefox 插件下载，就选择“自行分发”，也就是 unlisted 渠道。

然后上传打包好的 XPI。XPI 本质上就是 ZIP 文件，但是要注意 `manifest.json` 必须直接位于压缩包根目录，不能在外面多套一层文件夹。

上传以后 Mozilla 会自动检测插件。检测失败就按照页面上的错误逐个修改，常见问题有：

1. `manifest.json` 格式错误，或者使用了 Firefox 不支持的 Chrome 字段。
2. Gecko ID 缺失，或者新版本和旧版本使用了不同的 ID。
3. 上传过相同的版本号。每次上传新版都必须增加 manifest 中的 `version`。
4. 插件申请了权限，但是没有正确声明数据收集用途。
5. 压缩包中包含源代码、临时文件或者不需要的构建文件。

Firefox 新版对插件数据收集声明检查得比较严格。比如插件需要读取网页活动和登录信息时，可以在 Gecko 配置中声明：

```json
{
  "browser_specific_settings": {
    "gecko": {
      "id": "my-extension@example.com",
      "strict_min_version": "140.0",
      "data_collection_permissions": {
        "required": ["websiteActivity", "authenticationInfo"],
        "has_previous_consent": false
      }
    }
  }
}
```

具体声明要和插件真正读取的数据保持一致，不要直接照抄上面的权限。检测页面报错时，也要以 Mozilla 当时给出的字段说明为准。

检测通过以后继续提交。自行分发的插件不需要上架商店，Mozilla 处理完成后会生成一个已经签名的 XPI，下载这个文件才可以在正式版 Firefox 和 Firefox ESR 中长期安装。

安装签名 XPI 的方法是打开 `about:addons`，点击右上角齿轮，选择“从文件安装附加组件”。

#### 获取 Firefox Add-ons API 信息

每次都在开发者中心手工上传和下载太麻烦，Mozilla 提供了 API，可以用 `web-ext` 自动上传、检测和下载签名后的 XPI。

登录开发者中心后，打开 API 凭据页面：

https://addons.mozilla.org/developers/addon/api/key/

点击生成新的凭据，页面会显示两项信息：

1. JWT issuer：这就是 API Key，一般长得像 `user:123456:xxx`。
2. JWT secret：这就是 API Secret。

API Key 不是插件的 Gecko ID，也不是 Mozilla 账号。API Secret 相当于账号密码，不要写进源码，不要提交到 Git，也不要放到公开日志里。如果密钥泄漏了，马上回到这个页面重新生成凭据，让旧密钥失效。

我一般把它们分别保存成两个文件：

```text
firefox_addons_api_key.txt
firefox_addons_api_secret.txt
```

然后限制文件权限，并加入 `.gitignore`：

```bash
chmod 600 firefox_addons_api_key.txt firefox_addons_api_secret.txt
```

#### 用 web-ext 自动签名

先确保待签名目录中包含 `manifest.json` 和插件需要的所有文件，然后执行：

```bash
export WEB_EXT_API_KEY="$(<firefox_addons_api_key.txt)"
export WEB_EXT_API_SECRET="$(<firefox_addons_api_secret.txt)"

npx --yes web-ext@latest sign \
  --channel unlisted \
  --source-dir ./firefox-extension \
  --artifacts-dir ./artifacts
```

`unlisted` 表示自行分发。`web-ext` 会把插件上传给 Mozilla，等待检测和签名，再把签名后的 XPI 下载到 `artifacts` 目录。

不要把 API Key 和 Secret 直接写在命令行参数中，否则可能进入 Shell 历史或者被进程列表看到。用环境变量传递会稳妥一些，自动构建程序也应该避免打印这两个变量。

每次生成新版 Firefox 插件前，要先增加 manifest 中的 `version`。同一个 Gecko ID 和同一个版本号不能重复签名，否则 Mozilla API 会直接拒绝。

#### 怎么确认 XPI 已经签名？

可以解压 XPI，检查里面是否存在 Mozilla 的签名文件：

```bash
unzip -l my-extension.xpi | grep META-INF
```

正常会看到类似下面的文件：

```text
META-INF/mozilla.rsa
META-INF/mozilla.sf
META-INF/cose.sig
```

不过最终还是要在正式版 Firefox 或 Firefox ESR 中实际安装一次。能通过 `about:addons` 正常安装，并且重启 Firefox 后插件仍然存在，才算整个签名和发布流程真正跑通。

简单总结一下：开发阶段用 `about:debugging` 临时加载，交付用户时走 unlisted 自动签名，想公开推广时再发布到 Firefox 插件商店。三种方式不要混在一起，就不会一直被 Firefox 的签名错误折腾。

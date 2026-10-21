# 人生执行导航站 · 公网发布与管理员路径

> 一句话：这份包只有「静态页面 + 反扒声明 + 安全头」；改内容只能在你本机，公网侧没有任何管理入口。

## 一、上传什么（就这 7 个文件）

来源：`life-guide\public\`

| 文件 | 作用 |
|---|---|
| `index.html` | 整站单文件、零外链 |
| `robots.txt` | 放行搜索引擎；屏蔽 `/admin`、`/_tools`；挡 GPTBot / CCBot / Bytespider |
| `_headers` | Netlify / Cloudflare Pages 安全头：`X-Frame-Options: DENY`、CSP、`Permissions-Policy`、`nosniff` |
| `.well-known/security.txt` | 侵权与安全问题联系方式 |
| `CONTENT-POLICY.md` | 授权与反扒说明（允许什么、禁止什么、抓取的现实） |
| `PUBLISH.md` | 本文件 |
| `SHA256SUMS.txt` | 每个文件的 sha256 + 页面内容指纹 |

**绝不要上传**：`life-guide\data\`、`life-guide\_tools\`、`geektools\`（含 `guide-admin-token.txt`）、`life-guide\.cache\`、`*.bak-*`、`*.zip`。
`check-public.mjs` 会替你扫一遍这些泄露面。

## 二、管理员路径（改内容的唯一入口）

```powershell
# 1) 本地管理台
node geektools\console.mjs --no-open          # http://127.0.0.1:8123/guide/

# 2) 改一条内容（写 yml → 自动备份 → 自动重建）
node life-guide\_tools\admin-edit.mjs --id hukou --field title --value "新标题"
node life-guide\_tools\admin-edit.mjs --id hukou --show          # 只看当前值

# 3) 或直接调接口（必须带令牌）
$tok = (Get-Content geektools\guide-admin-token.txt -Raw).Trim()
'{"id":"hukou","field":"title","value":"新标题"}' | Set-Content .tmp\w.json -Encoding UTF8
curl.exe -s -X POST -H "x-guide-token: $tok" -H "Content-Type: application/json" --data-binary "@.tmp\w.json" http://127.0.0.1:8123/guide/api/item
```

- 令牌文件：`geektools\guide-admin-token.txt`（36 位随机；删掉会自动重建，旧令牌立即失效）
- 接口只接受**本机回环地址** + `x-guide-token`：缺令牌 401，非本机 403
- 字段白名单：`title` / `verified_at`（扩展改 `console.mjs` 的 `GUIDE_WRITE_FIELDS`）
- 页面**故意没有编辑按钮**：用户只有收藏、笔记（存本地浏览器）、卡点反查

## 三、改完重新发布

```powershell
node life-guide\_tools\build-life-guide.mjs --strict   # 重建 dist（打印内容指纹）
node life-guide\_tools\pack-public.mjs                 # 重新生成 public\ 与 SHA256SUMS
node life-guide\_tools\check-public.mjs                # 上传前总检，必须全绿
```

GitHub Pages：把 `public\` 的内容推到 `gh-pages` 根目录（或 Settings → Pages 指定目录）。仓库里只放 `public\` 的内容，源码另存私有仓库或本地。

## 四、反扒做到了什么 / 做不到什么（诚实版）

做到的：复制内容自动带出处与指纹｜被 iframe 套壳自动破框｜页面底部 12 位内容指纹可与 `SHA256SUMS.txt` 对照｜`robots.txt` 挡 AI 抓取 UA｜CSP `default-src 'none'` 防注入外链｜`referrer` 只带 origin｜无头环境只打标记不封禁（不误伤搜索引擎与无障碍工具）。

做不到：**浏览器能渲染的内容，理论上任何人都能抓走**。上面全是「提高成本 + 留追溯线索」。真要挡批量抓取和流量攻击，必须托管层做：

- Cloudflare（免费版够用）：Bot Fight Mode + Rate Limiting（如每 IP 每分钟 60 次）+ 被打时临时开 Under Attack Mode
- Netlify：已带 `_headers`；面板里可加密码保护 / 速率限制（付费档）
- 要「只给特定人看」：静态站做不到，得上带登录的服务

## 五、出事时的处置顺序

1. 先排除缓存：`Ctrl+F5`，或对比页脚指纹与 `SHA256SUMS.txt`
2. 有人镜像/冒名：用指纹证明来源 → 跑 `security.txt` 联系方式或平台侵权投诉（GitHub DMCA / Netlify abuse）
3. 被刷流量：Cloudflare 开 Under Attack Mode + Rate Limiting；必要时下线公网站点（本地 8123 不受影响）
4. 内容被改还署别人名：指纹不一致就是证据；重跑 `build → pack → check` 恢复

## 六、域名怎么选（我实测过的候选，2026-10-22）

先说结论：**优先 howto-live.com**；想要工具感选 howto-live.app；想在国内社交平台好传播选 enguide.cn。

| 候选 | 实测（DNS + HTTPS） | 判断 |
|---|---|---|
| howto-live.com | 无解析记录 | **可用概率高**（最终以注册商查询为准），.com 通用、好念 |
| howto-live.app | 无解析记录 | 备选；.app 强制 HTTPS，工具类站点气质对 |
| howtolive.dev / howto-live.dev | 无解析记录 | 备选；.dev 同样强制 HTTPS，偏技术向 |
| enguide.cn | 无解析记录 | 备选；中文拼音，短、好口播，国内分享友好（挂 GitHub Pages / Netlify 无需备案；挂国内服务器才要） |
| howto-live.xyz | 无解析记录 | 便宜，但长期可信度一般，做备用跳转可以 |
| howtolive.guide | 已注册（有站，HTTPS 200） | ✗ |
| lifeguide.cn | 已注册（有解析） | ✗ |
| lifeguide.app | 已注册（有站） | ✗ |
| howtolive.cc | 已注册（301 跳转） | ✗ |

- **DNS 只能证明「有解析=被注册」，不能证明「没解析=可注册」**——下单前在注册商搜索框确认一次。
- 参考价（2026 年常见区间）：.com ¥60–90/年、.app ¥110–140/年、.dev ¥110–140/年、.cn ¥30–60/年、.xyz ¥10–30/年。
- 一定要开 **WHOIS 隐私保护**（现在多数注册商默认免费）。
- 注册商建议：**Cloudflare Registrar**（成本价、无附加费）｜阿里云万网（国内付钱方便）｜NameSilo（便宜、隐私免费）。
- 先用二级域上线也行：Netlify 送 xxx.netlify.app、GitHub Pages 送 用户名.github.io/仓库名；等名字定了再绑自有域名（Netlify: Domain management → Add custom domain；GitHub: Pages → Custom domain）。
- 绑自有域名后记得开 **Enforce HTTPS**，并在 DNS 里：根域用 A/CNAME 到托管商，www 用 CNAME 指过去。

## 七、传到 GitHub（手把手，公网只放 public 内容）

原则：**仓库里只放\public\ 里那 8 个文件**，源码（\data\、\_tools\、\geektools\、令牌）不进公开仓库。

\\\powershell
# 1) 在 GitHub 网页先建一个空仓库，例如 life-guide（不要勾初始化 README）

# 2) 只把 public 目录变成仓库根
cd D:\27302\Documents\deep work\life-guide\public
git init -b main
git add index.html robots.txt _headers netlify.toml CONTENT-POLICY.md PUBLISH.md SHA256SUMS.txt .well-known/security.txt
git commit -m "初始发布：人生执行导航站（内容指纹 155b4b2b7088）"

# 3) 推到刚建的仓库（把 <你的用户名> 换成自己的）
git remote add origin https://github.com/<你的用户名>/life-guide.git
git push -u origin main
\\\

> 千万别在 \life-guide\\\ 根目录 \git add .\——那会把 \data\\\、\_tools\\\、\.cache\ 全带上。只在 \public\\\ 里 add。

4) 开 GitHub Pages：仓库 **Settings → Pages → Source: Deploy from a branch → Branch: \main\ / \(root)\ → Save**，
   一分钟后访问 \https://<你的用户名>.github.io/life-guide/\。
5) 绑自有域名：同页 **Custom domain** 填你的域名 → 到 DNS 商加记录：
   - 根域（\xample.com\）：A 记录指向 \185.199.108.153 / 185.199.109.153 / 185.199.110.153 / 185.199.111.153\
   - \www\：CNAME 指向 \<你的用户名>.github.io\
   - 回来勾上 **Enforce HTTPS**（证书自动签发，通常几分钟）
6) 以后每次更新：

\\\powershell
node life-guide\_tools\build-life-guide.mjs --strict
node life-guide\_tools\pack-public.mjs
node life-guide\_tools\check-public.mjs          # 必须 34 PASS
cd life-guide\public; git add -A; git commit -m "内容更新：<一句话>"; git push
\\\

## 八、GitHub 社区礼节（少踩坑）

- **README.md 写清楚**：这是什么、给谁用、数据什么时候核实的、内容指纹怎么核对、纠错去哪儿提。别人信任来源，才会收藏。
- **许可证要明确**：内容用 \CC BY-NC-SA 4.0\（署名-非商业-相同方式共享）最省事；代码（如果你以后开源工具）用 MIT。纯内容也可以写「All rights reserved，转载需授权」——但那样别人不能合法引用，传播会慢一些。
- **开 Issues 收纠错**：加一个 issue 模板（「哪一条过期了 / 官方链接挂了 / 说法有误」三行就够）。政策类内容一定会有过期，让别人帮你报错是最便宜的维护方式。
- **打一个 Release**：tag \1.0.0\，附上 \SHA256SUMS.txt\，别人下载后能自行校验。
- **不要放进仓库的东西**：真实姓名/学校/单位/手机号、\guide-admin-token.txt\、\data\\\、\_tools\\\、\.cache\\\、任何 \*.bak-*\。commit 历史里的东西删不干净，宁可第一次就别提交。
- **不要为了 star 去刷**：内容站被举报搬运很容易翻车；把「内容指纹 + 更新时间 + 官方来源」写清楚，比任何推广都稳。
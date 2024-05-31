# twikoo deployment on Cloudflare workers

这是 Cloudflare Worker 上的 twikoo 部署。与 Vercel/Netlify + MongoDB 等其他部署相比，它大大改善了冷启动延迟 ( 6s-> <0.5s)。延迟改进主要来自对 Cloudflare Worker 的大量优化以及 HTTP 服务器和数据库之间的集成环境 (Cloudflare D1)。

## 部署步骤

1. 安装 npm 包：
  ```shell
  git clone https://github.com/1436223139/twikoo-cloudflare.git
  cd twikoo-cloudflare.git && npm install
  ```
2. 由于 Cloudflare Workers 的免费套餐对包大小有严格的 1MiB 限制，因此我们需要手动删除一些包以使包保持在限制范围内。由于Cloudflare Workers 的Node.js(#兼容性问题)，这些包无论如何都无法使用。
  ```shell
  echo "" > node_modules/jsdom/lib/api.js
  echo "" > node_modules/tencentcloud-sdk-nodejs/tencentcloud/index.js
  echo "" > node_modules/nodemailer/lib/nodemailer.js
  ```
3. 登录您的 Cloudflare 账户：
  ```shell
  npx wrangler login
  ```
3. 创建 Cloudflare D1 数据库：
  ```shell
  npx wrangler d1 create twikoo
  ```
4. 找到CloudFlare控制台，twikoo这个新创建的D1数据库下，数据库 ID：(#f3bxxxa4-dxx5-4597-b678-f7f7xxx163f0)，编辑wrangler.toml并将database_id替换为你自己的数据库ID:
  ```shell
  vim wrangler.toml
  ```
5.导入Sql至新创建的D1数据库中
  ```shell
  npx wrangler d1 execute twikoo --remote --file=./schema.sql
  ``` 
在输出中检查“database_name”和“database_id”两行，和wrangler.toml做对比确保无误
6. 部署 Cloudflare worker:
  ```shell
  npx wrangler deploy --minify
  ```
7. 如果一切顺利，您将在命令行中看到类似以下内容： `https://twikoo-cloudflare.<your user name>.workers.dev` 您可以访问该地址。如果一切设置完美，您将在浏览器中看到类似以下内容的行：
  ```
  {"code":100,"message":"Twikoo 云函数运行正常，请参考 https://twikoo.js.org/frontend.html 完成前端的配置","version":"1.6.33"}
  ```
8. 当你设置前端时，第 6 步中的地址 (包括前缀 `https://`) 应该用作 `envId` 中的字段 `twikoo.init`.

## 已知限制

由于 Cloudflare 工作器仅与 Node.js [部分兼容](https://developers.cloudflare.com/workers/runtime-apis/nodejs/) , 因此由于兼容性问题，twikoo Cloudflare 部署存在某些功能限制：

1. 环境变量 (`process.env.XXX`) 不能用于控制应用程序的行为。
2. 腾讯云无法集成。
3. 无法根据 IP 地址找到位置 (软件包的兼容性问题 `@imaegoo/node-ip2region`).
4. 由于包的兼容性问题，包 `dompurify` 不能用于清理注释`jsdom`。相反，我们使用[`xss`](https://www.npmjs.com/package/xss)包进行 XSS 清理。
5. 在此部署中，我们不规范`/some/path/`和 `/some/path`之间的 URL 路径。这是因为编写 Cloudflare D1 SQL 查询来统一这两种路径并不容易。如果您的网站可以 `/` 在同一页面上同时拥有带尾随和不带尾随的路径，则可以在中明确设置该 `path` 字段 `twikoo.init`.

## 配置电子邮件通知

由于`nodemailer` 由于nodemailer软件包的兼容性问题，通过 SMTP 发送通知的电子邮件集成无法直接工作。相反，在此工作器中，我们支持通过 SendGrid 的 HTTPS API 发送电子邮件通知。要通过 SendGrid 启用电子邮件集成，您可以按照以下步骤操作：
1. 确保您有一个可用的 SendGrid 帐户（SendGrid 提供免费套餐，每天最多可发送 100 封电子邮件），并创建一个 API 密钥。
2. 再配置中设置以下字段：
  * `SENDER_EMAIL`: 发件人的电子邮件地址。需要在 SendGrid 中验证。
  * `SENDER_NAME`: 显示为发件人的姓名。
  * `SMTP_SERVICE`: `SendGrid`.
  * `SMTP_USER`: 提供一些非空值。
  * `SMTP_PASS`: API 密钥。
3. 或者，您可以设置其他配置值来自定义通知电子邮件的外观。
4. I在配置页面中，单击 `Send test email` 按钮以确保集成正常运行。
5. 在您的电子邮件提供商中，确保收到的电子邮件不会被归类为垃圾邮件。

---

If you encounter any issues, or have any questions for this deployment, you can send an email to tao@vanjs.org.

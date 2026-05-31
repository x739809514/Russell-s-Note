你已经创建了 X App，下一步是在本机给 xurl 注册这个 App 并完成 OAuth 授权。
    
    我刚检查了一下，你这里已经安装了 xurl：
    
    /home/russell/.local/bin/xurl
    
    但还没有注册任何 X App：
    
    No apps registered. Use 'xurl auth apps add' to register one.
    
    接下来请你自己在终端运行下面几步。注意：不要把 Client ID / Client Secret 发给我，也不要贴到聊天里。
    
    步骤 1：确认 X App 的回调地址
    
    去 X Developer Portal：
    
    https://developer.x.com/en/portal/dashboard
    
    打开你的 App，找到 User authentication settings，确认：
    
    Callback URI / Redirect URI 设置为：
    
    http://localhost:8080/callback
    
    Website URL 可以随便填你的站点，比如：
    
    https://example.com
    
    App permissions 建议至少选择：
    
    Read and write
    
    如果你以后要私信，才需要 DM 权限。
    
    步骤 2：注册 App 到 xurl
    
    在你的终端运行：
    
    bash
    xurl auth apps add my-x-app --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET
    
    
    把：
    - YOUR_CLIENT_ID 换成 X 后台的 Client ID
    - YOUR_CLIENT_SECRET 换成 X 后台的 Client Secret
    
    例如格式是这样，但不要把真实值发给我：
    
    bash
    xurl auth apps add my-x-app --client-id abc123 --client-secret xyz789
    
    
    步骤 3：执行 OAuth 登录
    
    运行：
    
    bash
    xurl auth oauth2 --app my-x-app YOUR_X_USERNAME
    
    
    把 YOUR_X_USERNAME 换成你的 X 用户名，不带 @ 或带 @ 都可以。
    
    例如：
    
    bash
    xurl auth oauth2 --app my-x-app yourname
    
    
    它会打开浏览器，让你登录 X 并授权。
    
    如果浏览器没有自动打开，终端通常会给一个链接，你复制到浏览器打开即可。
    
    步骤 4：设置默认 App
    
    授权成功后运行：
    
    bash
    xurl auth default my-x-app YOUR_X_USERNAME
    
    
    例如：
    
    bash
    xurl auth default my-x-app yourname
    
    
    步骤 5：验证
    
    运行：
    
    bash
    xurl auth status
    xurl whoami
    
    
    如果成功，xurl whoami 应该能显示你的 X 账号信息。
    
    完成后告诉我一句：
    
    “授权好了”
    
    然后我会帮你验证可用状态，并开始搭建你的 X 运营流程，比如：
    
    1. 先做账号定位
    2. 配置要监控的主题关键词
    3. 搜索近期热门推文
    4. 分析趋势
    5. 生成首批推文草稿
    6. 后续再决定是否定时自动执行
    
    如果你想直接复制粘贴，按这个模板运行就行：
    
    bash
    xurl auth apps add my-x-app --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET
    
    xurl auth oauth2 --app my-x-app YOUR_X_USERNAME
    
    xurl auth default my-x-app YOUR_X_USERNAME
    
    xurl auth status
    
    xurl whoami
    
    
    再次提醒：Client ID / Client Secret 只在你自己的终端里输入，不要发到聊天里。

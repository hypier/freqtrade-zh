## CORS

这一部分内容仅在跨域场景下需要配置（即你的多个机器人API运行在 `localhost:8081`、`localhost:8082` 等不同端口），并希望将它们合并到一个 FreqUI 实例中。

??? info "技术说明"
    所有基于网页的前端界面都必须遵守 [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) —— 跨域资源共享协议。
    由于大部分请求频率交易API都需要认证，设置正确的CORS策略对于避免安全问题至关重要。
    此外，标准规定对于带有凭证的请求不得使用 `*` 作为 CORS 策略，因此必须正确设置该项。

用户可以通过 `CORS_origins` 配置项允许来自不同源URL的访问该机器人API。
它由一个允许访问的URL列表组成，允许这些URL访问机器人的API资源。

假设你的应用部署在 `https://frequi.freqtrade.io/home/`，那么以下配置就变得必要：

```jsonc
{
    //...
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": ["https://frequi.freqtrade.io"],
    //...
}
```

在以下（非常常见的）场景中，FreqUI 在 `http://localhost:8080/trade` 可访问（这是你在浏览器导航到 freqUI 时在导航栏看到的地址）。
![freqUI url](assets/frequi_url.png)

对此情况的正确配置应为 `http://localhost:8080` —— 包括端口的主URL部分。

```jsonc
{
    //...
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": ["http://localhost:8080"],
    //...
}
```

!!! Tip "末尾斜杠"
    在 `CORS_origins` 配置中，不允许包含结尾的斜杠（例如 `"http://localhost:8080/"`）。
    这样的配置将不起作用，CORS错误仍会存在。

!!! Note
    我们强烈建议同时将 `jwt_secret_key` 设置为随机且只有你自己知道的值，以防止未授权访问你的机器人。
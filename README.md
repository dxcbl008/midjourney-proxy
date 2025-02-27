# midjourney-proxy

代理 MidJourney 的discord频道，实现api形式调用AI绘图

## 使用前提
1. 科学上网
2. docker环境
3. 注册 MidJourney，创建自己的频道，参考 https://docs.midjourney.com/docs/quick-start
4. 添加自己的机器人: [流程说明](./docs/discord-bot.md)

## 快速启动

1. 下载镜像
```shell
docker pull novicezk/midjourney-proxy:1.0
```
2. 启动容器，并设置参数
```shell
docker run -d --name midjourney-proxy \
 -p 8080:8080 \
 -e mj.discord.guild-id=xxx \
 -e mj.discord.channel-id=xxx \
 -e mj.discord.user-token=xxx \
 -e mj.discord.bot-token=xxx \
 --restart=always \
 novicezk/midjourney-proxy:1.0
```
3. 访问 http://ip:8080/mj/trigger/submit 提交绘图任务

## 注意事项
1. 启动失败请检查全局代理或HTTP代理，排查 [JDA](https://github.com/DV8FromTheWorld/JDA) 连接问题
2. 若回调通知接口失败，请检查网络设置，容器中的宿主机IP通常为172.17.0.1
3. 在 [Issues](https://github.com/novicezk/midjourney-proxy/issues) 中提出其他问题或建议
4. 感兴趣的朋友也欢迎加入交流群讨论一下
 <img src="https://raw.githubusercontent.com/novicezk/midjourney-proxy/main/docs/wechat-qrcode.png" width = "300" height = "300" alt="交流群二维码" align=center />

## 配置项

| 变量名 | 非空 | 描述 |
| :-----| :----: | :---- |
| mj.discord.guild-id | 是 | discord服务器ID |
| mj.discord.channel-id | 是 | discord频道ID |
| mj.discord.user-token | 是 | discord用户Token |
| mj.discord.bot-token | 是 | 自定义机器人Token |
| mj.discord.mj-bot-name | 否 | mj机器人名称，默认 "Midjourney Bot" |
| mj.notify-hook | 否 | 任务状态变更回调地址 |
| mj.translate-way | 否 | 中文prompt翻译方式，可选null(默认)、baidu、gpt |
| mj.baidu-translate.appid | 否 | 百度翻译的appid |
| mj.baidu-translate.app-secret | 否 | 百度翻译的app-secret |
| mj.openai.gpt-api-key | 否 | gpt的api-key |
| mj.openai.timeout | 否 | openai调用的超时时间，默认30秒 |
| mj.openai.model | 否 | openai的模型，默认gpt-3.5-turbo |
| mj.openai.max-tokens | 否 | 返回结果的最大分词数，默认2048 |
| mj.openai.temperature | 否 | 相似度(0-2.0)，默认0 |

## API接口说明

### 1. `/trigger/submit` 提交任务
POST  application/json
```json
{
    // 动作: 必传，IMAGINE（绘图）、UPSCALE（选中放大）、VARIATION（选中变换）
    "action":"IMAGINE",
    // 绘图参数: IMAGINE时必传
    "prompt": "猫猫",
    // 任务ID: UPSCALE、VARIATION时必传
    "taskId": "1320098173412546",
    // 图序号: 1～4，UPSCALE、VARIATION时必传，表示第几张图
    "index": 3,
    // 自定义字符串: 任务中保留
    "state": "test:22"
}
```
返回 `Message`，code=1表示提交成功，其他时description为错误描述
```json
{
  "code": 1,
  "description": "成功",
  "result": "8498455807619990"
}
```
result: 任务ID，用于后续查询任务或提交变换任务

### 2. `/trigger/submit-uv` 提交选中放大或变换任务
POST  application/json
```json
{
    // 自定义参数: 任务中保留
    "state": "test:22",
    // 任务描述: 选中ID为1320098173412546的第2张图片放大
    // 放大 U1～U4 ，变换 V1～V4
    "content": "1320098173412546 U2"
}
```
返回结果同 `/trigger/submit`

### 3. `/task/{id}/fetch` GET 查询单个任务
```json
{
    // 动作: IMAGINE（绘图）、UPSCALE（选中放大）、VARIATION（选中变换）
    "action":"IMAGINE",
    // 任务ID
    "id":"8498455807628990",
    // 绘图参数
    "prompt":"猫猫",
    // 绘图参数英文
    "promptEn":"cat",
    // 执行的命令
    "description":"/imagine cat",
    // 自定义参数
    "state":"test:22",
    // 提交时间
    "submitTime":1682473784826,
    // 结束时间
    "finishTime":null,
    // 生成图片的url, 成功时有值
    "imageUrl":"https://cdn.discordapp.com/attachments/xxx/xxx/xxxx__xxxx.png",
    // 任务状态: NOT_START（未启动）、IN_PROGRESS（执行中）、FAILURE（失败）、SUCCESS（成功）
    "status":"IN_PROGRESS"
}
```

### 4. `/task/list` GET 查询所有任务
***任务缓存1天后删除***
```json
[
  {
    "action":"IMAGINE",
    "id":"8498455807628990",
    "prompt":"猫猫",
    "promptEn":"cat",
    "description":"/imagine cat",
    "state":"test:22",
    "submitTime":1682473784826,
    "finishTime":null,
    "imageUrl":null,
    "status":"IN_PROGRESS"
  }
]
```

## `mj.notify-hook` 任务变更回调
POST  application/json
```json
{
    "action":"IMAGINE",
    "id":"8498455807628990",
    "prompt":"猫猫",
    "promptEn":"cat",
    "description":"/imagine cat",
    "state":"test:22",
    "submitTime":1682473784826,
    "finishTime":null,
    "imageUrl":null,
    "status":"IN_PROGRESS"
}
```

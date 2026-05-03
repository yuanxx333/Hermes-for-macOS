<h1 align="center">Mac 安全部署Hermes</h1>
<p align="left">项目描述</p>
mac OS（m系列）本地安全搭建Hermes AI Agent，使用docker安全隔离密钥与目录，能防止密钥泄漏，让Agent只能docker挂载目录内文件。m1 实测通过。

# 1. 安装Linux虚拟环境
因为 Docker 依赖 Linux 内核特性（cgroups、namespace）。macOS 内核不支持，所以必须在安装一个轻量 Linux 虚拟机。
Colima 是专门给 macOS 设计的Linux虚拟环境，Docker 运行其中，而且对 M1 原生支持很好。

## 安装 Colima 和 docker cli

`brew install colima docker docker-compose`

**colima 常用命令**
```
colima start #启动 
colima stop #停止
colima status #查看状态  
```
验证,在宿主机的命令窗口
docker --version && docker-compose version

        
# 2. 创建目录
```
home  
  |____hermes （Agent配置目录，在宿主机）  
       |____docker-compose.yml （docker 配置）  
	     |____.env （API key 等不想泄漏的密钥文件，在docker之外，很安全）  
  |____.hermes（docker容器挂载目录，运行在docker中的Hermes只能操作该目录,存放Hermes 所有状态的，包括配置、记忆、技能、会话历史）  
```
**使用的命令**
```
mkdir -p ~/hermes 
mkdir -p ~/.hermes

#创建 .env 文件
cd ~/hermes
vim .env
OPENROUTER_API_KEY=sk-or-替换为你的key
#ANTHROPIC_API_KEY=sk-ant-替换为你的key
#OPENAI_API_KEY=sk-替换为你的key
TELEGRAM_BOT_TOKEN=your bot token
TELEGRAM_ALLOWED_USERS=your user id
TELEGRAM_MODE=polling

#设置 .env 权限，防止被其他用户读取
chmod 600 ~/hermes/.env
```
		  
# 3. 初始化 Hermes（首次运行）
#docker run 会自动完成以下步骤：  
1. 检查本地 nousresearch/hermes-agent 镜像  
2. 没有就自动从 Docker Hub 拉取  
3. 拉取完启动容器，运行 setup 向导  


```
#首先启动linux虚拟环境才能docker run 
colima start --cpu 2 --memory 6 --arch aarch64
cd ~/hermes
docker run -it --rm \
  --name hermes \
  -v ~/.hermes:/opt/data \
  --env-file .env \
  nousresearch/hermes-agent setup
注意1：docker-compose.yml中的配置是docker容器的大小，colima start命令中的大小是虚拟机的大小。docker大小设置的比虚拟机大是没用的。
注意2：初始化后不要直接运行，关闭后使用日常启动命令再次启动
```
# 4. 日常启动
**1.启动Linux虚拟机**  
`colima start --cpu 2 --memory 6 --arch aarch64`  

**2.启动docker容器**  
每次读取最新.env内容，需要连接外部聊天软件必须加gateway run
```
docker run -it \
  --name hermes \
  -v ~/.hermes:/opt/data \
  --env-file ~/hermes/.env \
  -p 8642:8642 \
  nousresearch/hermes-agent gateway run 
```
**3.与Hermes交互**  
1. 用聊天工具（telegram、weixin）与Hermes远程交互  

2. 用命令终端交互  
新开命令窗口，执行以下命令即可，不影响telegram与Hermes交互  
`docker exec -it hermes /opt/hermes/.venv/bin/hermes --tui`  

**4.查看Hermes日志**  
```
#查看日志，新开命令窗口
docker logs hermes -f
```

**5.关闭虚拟机与容器**  
```
docker stop hermes
colima stop
```



# 5. 进阶
**1.使用telegram时上下文长度问题**  

在telegram bot中聊天窗口消息一直在叠加，那么AI是有上下文长度限制的，如何清理或者压缩Hermes的上下文
Hermes 会自动处理上下文，但你也可以手动控制，使用/reset 或者 /clear 清除当前对话历史（AI会忘记之前的聊天内容）
Hermes 的自动处理更智能，当然一些无意义的闲聊可以清除掉
可以输入/help 获取帮助

**2.skill管理**  

1. 直接在技能目录下创建 SKILL.md 文件 Skill 文件存放在 ~/.hermes/skills/ 下  
2. 也可以通过命令行安装 hub 上的 skill  
```
hermes skills search <关键词>    # 搜索 skill
hermes skills install <ID>      # 安装 skill
```
3. 修改skill不需要重启容器，Skill 是每次会话（session）启动时加载的，不是容器启动时加载的。也可以执行/reset。

**3.Hermes实战**  
Hermes 本身就是"Agent 工厂"  
Hermes 可以创建更多 Agent 来完成子任务。 但不是传统的多线程，而是多 Agent 并发架构
每个子 Agent 都是：  
- 🧠 完整的 AI 会话（有自己的上下文窗口）
- 🛠️ 独立的工具集（可以 ssh、git、浏览网页）
- 🚫 互不干扰（A 的错误不会污染 B 的上下文）
- ⚡ 可以并行运行（最多同时跑 3 个）
举个实际例子  

你跟我说：  
"帮我写一个 Docker 监控面板，前端用 React，后端用 FastAPI，数据库用 SQLite"  
我会这样执行：  
① 分析需求 → 写实施计划  
    保存到 docs/plans/monitor-panel.md  

② 并行派发子 Agent：  
   ┌─ Agent A: FastAPI 后端（数据库模型 + API）  
   ├─ Agent B: React 前端（组件 + 路由）  
   ├─ Agent C: Docker SDK 集成（容器状态读取）  
   └─ Agent D: 测试 + 文档  

③ 每个任务完成后，自动审核：  
   规范检查 ✓ → 质量检查 ✓  

④ 最终集成验证 → 代码提交 → 告诉你完成  
总结一句话  

你只需要说出要做什么，我会自动拆解任务、创建多个子 Agent 并行干活、审核质量，最终交付完整项目。  
这就是 Hermes 的多 Agent 编排能力 🚀  

# 6.问题及解决方案
1. docker 无法启动  
若使用docker run无法启动，则停止并删除旧容器  
`docker stop hermes && docker rm hermes`

2. 选择deep seek model  
在.env文件中配置OPENROUTER_API_KEY=你的deep seek API key  
在交互界面中输入/model ，选择DeepSeek --> 选择deepseek-v4-flash  
不要选deepseek-chat和deepseek-reasoner将要下线。  

3. 查看当前Hermes加载了哪些环境变量  
`docker exec hermes env`


4. tg连通问题
- 获取tg UserID  
去 Telegram 搜索 @userinfobot，发送任意消息，它会返回你的数字 ID  
- 生成tg bot并获取bot token  
去 Telegram 搜索 @BotFather，点击进入输入/newbot，记录bot token  
- 连不上tg  
在初始化时就需要填写tg 的信息，只要正常启动了都能联通。  
连不通则进行一下测试，确认token是否正确，外网是否联通。
#测试宿主机是否连通tg  
`curl "https://api.telegram.org/bot<your bot token>/getWebhookInfo"`  
#测试容器是否能联通tg  
`docker exec hermes curl -s https://api.telegram.org/bot<your bot token>/getMe`  
若还连不通  
确认.env文件TELEGRAM_BOT_TOKEN为你的bot token  
然后先清除webhook,网页访问，之后再尝试是否联通。 
`https://api.telegram.org/bot<your bot token>/deleteWebhook`  


# 7. model选择使用本地ollama+gemma4
重新初始化选择本地模型（推荐gemma4:e4b）
1. 当出现provider的选项时选择LM Studio
2. 下一步需要输入LM_API_KEY时直接按enter跳过
3. 出现需要输入Base URL时，输入http://host.docker.internal:11434/v1
4. model name输入gemma4:e4b
注意：要使用本地模型则运行hermes前需要先本地运行ollama与gemma4
```
ollama serve
ollama pull gemma4:e4b
ollama run gemma4:e4b
```

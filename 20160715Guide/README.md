本文介绍拉取SGDevX/articles项目，并完成编辑、提交和推送的基本操作。

# 1. 拉取SGDevX/articles到本地
## 1.1 fork中央仓库到自己GitHub账户
注册一个GitHub账号，并在网页端登录。之后到[https://github.com/SGDevX/articles](https://github.com/SGDevX/articles) 点击右上角Fork按钮：

![Fork 中心仓库](01fork.png)

在弹出窗口上点击自己的账号，即可完成fork操作。

## 1.2 克隆个人仓库到本地
找到自己的个人仓库url：登录自己的GitHub，进入articles，点击“Clone or Download”

![个人仓库url](02url.png)

然后克隆该仓库：在windows下，安装[git for windows](https://git-for-windows.github.io/)，如果使用命令行，装这一个就足够了。如果使用GUI界面，可以再装一个[Tortoise Git](https://tortoisegit.org/) 。在macOS下我使用SourceTree。

之后就可以在本地Clone副本中编辑、修改以及提交。此时提交是提交到自己的个人仓库。

# 2. 将个人仓库中的修改推送给中央仓库
登录自己的GitHub并进入articles项目，点击“New pull request”按钮：

![New pull request](03NewPullRequest.png)

在新的页面中，确认basefork中选择了中央仓库的主干，点击“Create pull request”：

![Create pull request](04CreatePullRequest.png)

填写提交说明，并点击“Create pull request”按钮，一个推送请求就完成了。

# 3. 将推送请求合入主线
此时中央仓库将收到推送请求：

![Pull requests](05ReceivePull.png)

点击该条请求，可以点击“Merge pull request”将此次推送请求合入主线，或者如果有疑问，可以暂不合并，只是提交Comment，请求方将收到该Comment，双方讨论一致后，committer再将request合并到中央仓库。

# 4. 已经fork和clone的用户，当中央仓库更新后应怎么操作？
已经从中央仓库fork并clone到本地的用户，当中央仓库更新后，更新本地副本是不会得到中央仓库的更新内容的，因为此时他是从自己的个人仓库拉取内容。
正确的做法应该是先将中心仓库添加到项目的remote地址，取名为upstream，然后从upstream拉取内容到本地的clone副本，然后再把内容推送到个人仓库。这样就完成了个人仓库与中心仓库的同步。
之后的事情就和前面介绍的一致了。
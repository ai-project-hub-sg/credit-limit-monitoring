对添加在pixel的opneai账号预估额度进行监控，当出现额度异常时，自动暂停调度(可选)，并发出邮件警告。
1.go语言做后端
2.需要一个前端界面
3.sqlite做数据库
4.需要登录鉴权
5.默认登录账号为fuck_openai,默认登录密码为pixel_test
6.登录后立即调整修改密码页面，登录密码不允许与默认一致
7.除初始化登录外，每次登录都需要检测登录密码是否与默认密码一致。如果一致需要弹窗提醒用户，非正常使用情况，你的密码与初始密码一致，存在安全隐患，请立即修改！
8.允许用户在修改登录密码后修改登录账号。但是修改登录账号前需要验证登录密码是否已经被修改。默认登录密码不允许修改登录账号。
9.项目启动时，会默认生成两个hex 16位密码，一个用于加密sqlite。一个用于加盐加密sqlite中保存的机密信息，比如说密钥啊这些
10.允许用户在输入密码后查看两个密钥，也允许用户输入密码后修改这两个密钥。查看和修改需要分别输入密码。 
11.修改数据库密钥和加密密钥会停止数据库提供服务。特别是更换加密密钥会导致数据库中所有机密信息用老密钥解密，用新密钥加密。可能会需要一点时间，所以用户尝试更换密钥时，需要弹窗提醒用户，更换密钥会暂停数据库服务，轮换加密需要时间
12.源数据的获取会以json形式落盘在本地。抽取进入数据库后，可以被删除。用户可以在web设置保留几天。同时可以设置最大缓存用量。超出缓存后会动态删除最老的已经抽取的记录，给新记录腾出空间
13.轮换数据库密钥期间源数据的抽取不会暂停。但是etl进数据库的工作会被暂停。等待数据库轮换密钥的工作结束在继续
14.每x秒，默认为60，用户可以配置页面自定义设置从https://ai-pixel.online/api/v1/accounts/{account_id}/usage?source=local&timezone=Asia%2FShanghai 获取一次数据。 每次数据都需要解析后存在数据库中。
15.从usage的response中解析出seven_day的utilization、requests、tokens、cost、user_cost
16.根据解析出的数据，计算两个指标，需要一并存储进入数据库，作为一条记录,指标1:credit_limit_by_cost= cost/utilization,指标2:credit_limit_by_delta，这个指标计算需要从数据库获取最新的utilization和cost，然后计算(cost-cost_in_sql)/(utilization-utilization_in_sql)。如果出现了除以0的情况，结果记-999
17.每次新的数据进入数据库后，需要计算credit_limit_by_cost比起上一次是增加，减少，基本持平？如果减少超过x%,用户可以设置，默认为5%。((new_credit_limit_by_cost/last_credit_limit_by_cost)-1<-x%)就会给用户发额度异常预警邮件。 同时提高采样频率，提高到x秒一次，默认30秒，用户可以在配置界面设置。
18.每次新的数据进入数据库后，需要计算credit_limit_by_delta比起三档预设值的差异，如果credit_limit_by_delta为-999跳过本次比较，说明用量还没到1%，utilization未更新，这个数据没有更新，默认为3000,2400,1600,用户可以设置，当credit_limit_by_delta低于最低档时，默认会自动暂停异常账号调度。用户可以设置默认暂停或者不暂停。这个需要现在配置中设置全局开启或者关闭。然后可以在账号级别设置不同的额度和是否自动暂停
19.暂停调度的api在docs下的pixel_api.md中
20.其他需要的api如pixel的登录鉴权，获取account_id。见C:\Users\tiank\Desktop\coding_object\ai-pixel-analysis的docs下的api.md
21.本项目先生成一个windows的包测试功能。后续会打包成liunx的包部署在服务器运行。由nginx反代后提供web服务。

待确认事项的确认值
1.0版
1.utilization 的真实单位是0-100的百分数。值57，表示57%
2.自动关闭调度后恢复默认采样间隔。如果不设置自动关闭调度，那么当第一次触发credit_limit_by_cost后。设置一个计数器为0，当utilization比上一次加1时，说明已经是新的1%开始。再次到1%跳变时，说明这是一个全新的1%。而且是加速采样后的完整1%。如果这个1%仍然异常。可以邮件通知用户确认异常。由于未设置自动停止调度，建议立刻人工暂停，避免损失扩大。然后恢复默认采样频率。然后设置一个指示器。是否额度异常恢复。只有当额度异常恢复，才会触发下一轮的自动采样间隔增加。否则在一轮额度异常中无需反复调整采样频率。没有意义。
3.同一异常的邮件冷却、恢复判定和收件人范围。 异常邮件一个事件只发一封，无需反复发送。收件人由用户在web界面配置。不同的pixel账户，甚至同一个pixel账户管理的不同accout都可以配置不同的通知人。恢复判断了为credit_limit_by_cost上升。credit_limit_by_delta大于中档设置。
4.两个“16 位”密钥是 16 个十六进制字符，还是 16 字节后编码为 32 个字符。示例openssl rand -hex 16，可以采取go的实现方案。
5.SQLite 加密采用 SQLCipher、文件级加密，还是由运行环境提供磁盘加密。使用SQLCipher加密数据库
6.“数据库密钥”和“机密信息加密/加盐密钥”分别保护哪些字段，是否允许独立轮换。 允许独立轮转。本项目中机密信息主要web界面的登录密码，pixel的账号和密码,pixel登录后换取的授权令牌。
7.SMTP 配置和 Pixel 凭据是否由本地页面输入，是否支持多个 Pixel 租户。 支持多个pixel租户。每个pixel可能有多个管理账号。本次支持类型platform为openai的账号。
8.全局自动暂停与账号级设置的优先级是否固定为“全局关闭优先，账号级只能进一步关闭”。 全部关闭后，账号级不生效。账号级只在全局设为开启时才生效。
9.默认账号和密码是否只用于 Windows 测试包；Linux 首次启动是否改为安装时设置。默认账号密码linux安装时也默认。等用户第一次登录后修改。
10.原始缓存最大用量按字节、文件数还是两者同时限制。原始缓存最大用量指的是占用的存储字节数。

1.3-draft版
1.credit_limit_by_cost的计算规则修改。由于cost会持续增加，但是utilization在不足1%时，不会改变。所以会造成每一次utilization刚满1%时，会出现一次制度性的credit_limit_by_cost下降。可能出现假下降误报。修改为不再实时计算credit_limit_by_cost。而是当检测到utilization跳变时。将跳变当次记为N，将前一次记录记为O。credit_limit_by_cost=（cost_o\*0.5+cost_n\*0.5）/utilization_o\*100
2.credit_limit_by_delta与credit_limit_by_cost类似。不再计算每一次记录。而是当utilization跳变时，才计算。而且他应当记录上一次utilization跳变时的cost。计算两次utilization跳变之间cost差值。然后计算出这1%对应的credit_limit_by_delta。另外如果是初始化或者重启后，最初的1%不做预估，因为最初的cost可能并不对应刚跳变的utilization。计算出来的数值偏差估计无法保证。所以要从刚记录到utilization后开始计算。
3.量级示例，cost=318.24 utilization=16 cost/utilization的量级为318.24/16*100=1989
4.credit_limit_by_cost和credit_limit_by_delta分别发一封异常通知。
5.数据库的两份密钥保存在根目录下的.env文件中。作为环境变量被读取。
6.无需ssh远程连接。只允许web访问。数据库从dbx读取需要先用户自行登录服务器下载db文件，在本地dbx访问。无需开发ssh到服务器的数据库。
7.新增两个指标credit_limit_by_cost_real_time,等于最新一条记录的cost/utilization/*100，credit_limit_by_delta_real_time，等于(最新一条记录的cost-最近一次utilization跳变的cost)\*100

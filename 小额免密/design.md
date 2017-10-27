[TOC]

# 小额免密技术方案

## 改造思路

- ~~风控~~沿用现有支付原有的实时风控，增加资金种类维度
- 满足小额免密条件的交易在商户扫码下单后自动完成进行支付（快捷交易除外）
- 在SESSION中增加小额免密标志以供APP判断是否需要跳密码框
- 后台支付交易同样判别SESSION中的标志来选择是否需要进行验密流程

### 原交易流程

- 商户侧

    ```mermaid
    sequenceDiagram
        participant merc
        participant front
        participant pay

        merc->>front:扫码下单(bsdkmca1.TSDK4090400)
        front->>front:检查付款码
        front->>pay:消费下单
        pay-->>front:返回订单号
        front-->>pay:申请支付令牌
        pay-->>front:返回支付令牌
        front->>front:更新付款码
        loop 循环检查订单状态
            front-->>front:检查订单状态
            alt 订单被支付
                front-->>merc:返回订单支付状态
            else 订单未被支付
                note over merc,front:Continue
                
            end
        end
    ```

- 用户侧

    ```mermaid
    sequenceDiagram
        participant app
        participant front
        participant pay

        loop 循环查询待支付订单
            app->>front:查询待支付订单(eappmca1.APP4080805)
            front-->>app:返回查询结果
            
            alt 有待支付订单
                app->>app:唤起密码框
                app->>app:唤起短信验证交互流程(仅快捷)
                app->>front:支付确认(eappmca.EAPP4080806)
                front->>pay:账户/快捷/卡包支付
                pay-->>front:返回支付结果
                front-->>app:返回支付结果
            else  没有订单
                note over app,front:Continue
            end
        end
    ```


### 改造后交易流程

- 商户侧
    ```mermaid
    sequenceDiagram
        participant merc
        participant front
        participant pay

        merc->>front:扫码下单(bsdkmca1.TSDK4090400)
        front->>front:检查付款码
        front->>pay:消费下单
        note over merc,pay:**************************** 改造点 START ******************************
        pay->>pay:检查小额免密风控
        pay->>pay:更新小额免密标志到订单表
        note over merc,pay:**************************** 改造点 END ******************************
        pay-->>front:返回订单号
        front-->>pay:申请支付令牌
        pay-->>front:返回支付令牌
        front->>front:更新付款码

        note over merc,pay:**************************** 改造点 START ******************************

        front->>front:判断是否免密

        alt 免密
            front->>pay:账户/卡包支付
            pay->>front:返回支付结果
        else 非免密
            note over front,pay:该干嘛就干嘛
        end
        note over merc,pay:*************************** 改造点 END ******************************

        loop 循环检查订单状态
            front-->>front:检查订单状态
            alt 订单被支付
                front-->>merc:返回订单支付状态
            else 订单未被支付
                note over merc,front:Continue
                
            end
        end
    ```

- 用户侧
    ```mermaid
    sequenceDiagram
        participant app
        participant front
        participant pay

        loop 循环查询待支付订单
            app->>front:查询待支付订单(eappmca1.APP4080805)

            note over app,pay:*************************** 改造点 START ******************************
            front-->>app:返回查询结果 & 免密标志(SESSION)
            
            alt 有支付订单
                app->>app:判断订单状态

                alt 订单已支付
                    app->>app:直接显示支付订单成功信息

                else 订单未支付

                    app->>app:判断免密标志

                    alt 非免密
                        app->>app:唤起密码框
                        app->>app:唤起短信验证交互流程(仅快捷)
                        app->>front:支付确认(eappmca.EAPP4080806)
                        front->>pay:账户/快捷/卡包支付
                        pay-->>front:返回支付结果
                        front-->>app:返回支付结果
                    else 免密
                        app->>app:判断是否快捷
                        alt 快捷
                            app->>app:唤起短信验证交互流程(仅快捷)
                            app->>front:支付确认(eappmca.EAPP4080806)
                            front->>pay:快捷支付
                            pay-->>front:返回支付结果
                            front-->>app:返回支付结果
                        else 非快捷
                            app-->>app:直接显示订单支付结果
                        end
                    end
                end

            else  没有订单
                note over app,front:Continue
            end

            note over app,pay:*************************** 改造点 END ******************************

        end
    ```



## 风控改造

- 增加一套小额免密规则（自己写JAVA代码） `肖景才`

    - [x] 从产品获取风控组件源码
    - [ ] 风控组件改造
        1. 新建四条小额免密风控规则，但对应的java包名跟原产品日累，月累，日限额，月限额相同 
        2. 新建风控类型：小额免密
        3. 新建业务类型规则维护：小额免密类型 + 四条风控规则
        4. 新建资金种类：小额免密（风控累计维度是按照账户类型 + 资金种类 + 交易类型 + 业务类型 等来累计）
            > 可行性待验证（现风控组件可能将资金属性写死成现金），若不可行则自行开发风控组件 `已验证可行`
        5. 新建小额免密风控参数维护界面

- 运营增加独立的小额免密参数维护界面（可复制账户交易参数维护维护界面） `肖景才`

    <img src="http://o6fhqh1oe.bkt.clouddn.com/2017073115014905205923.png" style="zoom:50%" />

    > 运营系统->实时风控控制->风控参数维护->账户交易参数维护
    > 在维护界面中去做以下修改
    > - 账户类型：界面隐藏，后台固定值:100-个人账户 
    > - 资金种类：后台固定值:1-现金
    > - 交易类型：后台固定值:02-消费
    > - 业务类型：后台固定值:P0001-小额免密
    > - 实名标志：界面隐藏，后台固定值:02-强实名 
    > - 收付标志：后台固定值:D-付款方
    > - 限制级别：界面隐藏，后台固定值:1-用户级别限制
    > - 业务受理渠道：与原界面相同
    > - 单笔最大金额：后台固定值:200
    > - 单笔最小金额：后台固定值:0
    > - 日限额：与原界面相同
    > - 月限额：与原界面相同
    > - 日累计：与原界面相同
    > - 月累计：与原界面相同




- 支付核心增加小额免密判定接口 `肖景才` `com.murong.ecp.app.rpm.action.brpmpub8.T2015620`
    
    - 接口

        - 请求

            名称   | 长度   | 必输   | 备注
            ------ | :----: | :----: | --------
            cre_dt | 8      | M      | 创建日期
            ord_no | 20     | M      | 订单号

        - 返回

            名称    | 长度   | 必输   | 备注
            ------  | :----: | :----: | --------
            nnp_flg | 1      | M      | 免密标志 Y:免密  N:不免密

    - 交易流程 
        ```mermaid
            graph LR

            请求((开始))
            订单查询(订单查询)
            订单状态判断{订单状态判断}
            交易金额判断{交易金额判断}
            小额风控检查(小额风控检查<br/>chkRtmRsk)
            风控结果判断{风控结果判断}
            设置免密标志_Y(设置免密标志<br/>DDP_FLG:Y 免密)
            设置免密标志_N(设置免密标志<br/>DDP_FLG:N 不免密)
            更新免密标志(更新免密标志<br/>订单表DDP_FLG=Y)
            结束((结束))

            请求-->订单查询
            订单查询-->订单状态判断
            订单状态判断--待支付-->交易金额判断
            交易金额判断-- <=200 -->小额风控检查
            小额风控检查-->风控结果判断
            风控结果判断--满足-->设置免密标志_Y
            设置免密标志_Y-->更新免密标志
            更新免密标志-->结束

            设置免密标志_N-->结束
            风控结果判断--不满足-->设置免密标志_N
            交易金额判断-- >200 -->设置免密标志_N
            订单状态判断--非待支付-->设置免密标志_N
        ```

- 支付核心增加小额免风控累计接口 `肖景才` `com.murong.ecp.app.rpm.action.brpmpub8.T2015621`
    
    - 接口

        - 请求

            名称   | 长度   | 必输   | 备注
            ------ | :----: | :----: | --------
            cre_dt | 8      | M      | 创建日期
            ord_no | 20     | M      | 订单号

        - 返回

            名称    | 长度   | 必输   | 备注
            ------  | :----: | :----: | --------
                    |        |        |

- 小额免密金额标准 `业务`
    > <=200RMB

- 风控参数设置（可参考产品需求） `业务`

    | 日频次 | 日累计金额 | 月频次 | 月累计金额 |
    | :--:   | :---:      | :--:   | :---:      |
    | 10     | 2000       | 100    | 10000      |

## 线下扫码支付改造 

> 商户下单判断一旦满足小额免密直接支付 `郑骁`

### 改造图示

- 商户扫码下单请求（改造前）
    ```mermaid
    graph LR
        商户SDK(商户SDK)
        扫码下单(扫码下单<br/>TSDK4090400)
        登记远程消费订单(登记远程消费订单<br/>brpmpub8.2015301)

        商户SDK-->扫码下单
        扫码下单-->登记远程消费订单
        登记远程消费订单-->原有流程(原有流程)
        原有流程--返回商户token-->商户SDK
    ```
- 商户扫码下单请求（改造后）
    > 针对账户支付、卡包支付一旦满足小额免密后条件后(快捷需要短信验证码，故不做处理)，就后台自动发起支付请求完成支付
    ```mermaid
    graph LR
        商户SDK(商户SDK)
        扫码下单(扫码下单<br/>TSDK4090400)
        登记远程消费订单(登记远程消费订单<br/>brpmpub8.2015301)
        小额免密检查服务(小额免密检查服务)
        是否满足小额免密{是否满足小额免密}
        账户支付(账户支付<br/>bsdkmca1.SDK4090320)
        卡包支付(卡包支付<br/>bsdkmca1.SDK4090660)
        判断支付方式{判断支付方式}
        原有流程(原有流程)

        商户SDK-->扫码下单
        扫码下单-->登记远程消费订单
        登记远程消费订单-->小额免密检查服务
        小额免密检查服务-->是否满足小额免密
        是否满足小额免密--是-->判断支付方式
        判断支付方式--账户支付-->账户支付
        判断支付方式--卡包支付-->卡包支付
        判断支付方式--快捷支付-->原有流程
        是否满足小额免密--否-->原有流程
        账户支付-->原有流程
        卡包支付-->原有流程
        原有流程--返回商户支付结果/token-->商户SDK

        style 登记远程消费订单 fill:#f9f,stroke:#333,stroke-width:1px;
        style 是否满足小额免密 fill:#20E912,stroke:#333,stroke-width:1px;
        style 小额免密检查服务 fill:#20E912,stroke:#333,stroke-width:1px;
        style 判断支付方式 fill:#20E912,stroke:#333,stroke-width:1px;
        style 账户支付 fill:#20E912,stroke:#333,stroke-width:1px;
        style 卡包支付 fill:#20E912,stroke:#333,stroke-width:1px;
    ```



- APP发起支付确认（改造前）
    ```mermaid
    graph LR
        用户APP(用户APP)
        支付确认(支付确认<br/>eappmca.EAPP4080806)
        账户支付(账户支付<br/>bsdkmca1.SDK4090320)
        快捷支付(快捷支付<br/>bsdkmca1.SDK4090340)
        卡包支付(卡包支付<br/>bsdkmca1.SDK4090660)
        支付密码校验(支付密码校验<br/>burmpub1.0011471)
        判断支付类型{判断支付类型}
        密码验证(密码验证)
        原有流程(原有流程)
        

        用户APP-->支付确认
        支付确认-->判断支付类型
        判断支付类型-->账户支付
        判断支付类型-->快捷支付
        判断支付类型-->卡包支付

        账户支付-->支付密码校验
        快捷支付-->支付密码校验
        卡包支付-->支付密码校验

        支付密码校验-->密码验证
        密码验证-->原有流程
        原有流程--返回支付结果-->用户APP
    ```

- APP发起支付确认（改造后）
    ```mermaid
    graph LR
        用户APP(用户APP)
        支付确认(支付确认<br/>eappmca.EAPP4080806)
        账户支付(账户支付<br/>bsdkmca1.SDK4090320)
        快捷支付(快捷支付<br/>bsdkmca1.SDK4090340)
        卡包支付(卡包支付<br/>bsdkmca1.SDK4090660)
        支付密码校验(支付密码校验<br/>burmpub1.0011471)
        判断支付类型{判断支付类型}
        判断是否验密{判断是否验密}
        密码验证(密码验证)
        原有流程(原有流程)

        用户APP-->支付确认
        支付确认-->判断支付类型
        判断支付类型-->账户支付
        判断支付类型-->快捷支付
        判断支付类型-->卡包支付

        账户支付-->判断是否验密
        快捷支付-->判断是否验密
        卡包支付-->判断是否验密

        判断是否验密--需要-->支付密码校验
        判断是否验密--不需要-->原有流程

        支付密码校验-->密码验证
        密码验证-->原有流程
        原有流程--返回支付结果-->用户APP

        
        style 卡包支付 fill:#f9f,stroke:#333,stroke-width:1px;
        style 账户支付 fill:#f9f,stroke:#333,stroke-width:1px;
        style 快捷支付 fill:#f9f,stroke:#333,stroke-width:1px;

        style 支付密码校验 fill:#20E912,stroke:#333,stroke-width:1px;
        style 判断是否验密 fill:#20E912,stroke:#333,stroke-width:1px;
        style 密码验证 fill:#20E912,stroke:#333,stroke-width:1px;
    ```

- APP查询支付结果（改造前）
    ```mermaid
    graph LR
        用户APP(用户APP)
        查询支付结果(查询支付结果<br/>eappmca1.APP4080805)

        用户APP--每秒触发-->查询支付结果
        查询支付结果--返回支付结果-->用户APP

    ```

- APP查询支付结果（改造后）
    ```mermaid
    graph LR
        用户APP(用户APP)
        查询支付结果(查询支付结果<br/>eappmca1.APP4080805)

        用户APP--每秒触发-->查询支付结果
        查询支付结果--返回支付结果&是否免密-->用户APP

        style 查询支付结果 fill:#f9f,stroke:#333,stroke-width:1px;
    ```

### 代码改造

- 封装小额免密专用组件
    > 本质上就是调用现有风控组件，但资金属性送小额免密类型

- 付款码商户下单接口改造 `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090400`

    ```java 
    +246:
    } else {
        logger.info("else begin");
        PUBATCUtil.GDASetMsgIgnore(bizCtx, "SCM60001", "");
        logger.info("else end");
        return;
    }

    /** 此处增加支付相关代码 **/

    /** 调用小额免密检查服务 **/

    /** 若满足小额免密，并且又是账户支付或卡包支付，则调用支付接口进行支付 **/

    logger.info("循环查询该笔订单的状态    begin");
    Utils.setData(bizCtx, "count_index", "1");
    while(true){
        if(Utils.eval(bizCtx, "intcmp($count_index,1,50)")){
            logger.info("intcmp($count_index,1,50)  begin");
            PUBATCUtil.queryForObject(bizCtx, "qry_ord_inf", "ordinf");
    ```

- 账户支付请求交易改造 `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090320`

    ```mermaid
    graph LR
        TSDK4090320(bsdkmca1.TSDK4090320<br/>账户支付请求)-->T2011210(brpmpub1.T2011210<br/>订单支付)
    ```

    ```java
    /** 从SESSION获取免密标志，若为免密则跳过如下代码 **/
    +44:
    WEBATCUtil.getSession(bizCtx, "$sess_nm", sessMap);
    PUBATCUtil.logETF(bizCtx);

    logger.info("组装AES 解密所需的数据 begin");
    Utils.setData(bizCtx, "inp_dat.ocx_key", "$ocx_key");// 随机因子
    Utils.setData(bizCtx, "inp_dat.pswd_o", "$paypwd");// 登录密码
    logger.info("组装AES 解密所需的数据 end");

    logger.info("AES使用随机因子对密码进行解密   开始 ......");
    this.Pinconvert.process(bizCtx, "inp_dat", "out_dat|msg_cd|");
    logger.info("AES使用随机因子对密码进行解密  成功   结束 ......");
    Utils.setData(bizCtx, "pay_pswd", "@O.out_dat.pswd_c");

    Utils.setData(bizCtx, "pwd_typ", "PAY");
    logger.info("call package:[MCA.DecipherPwd]");
    DecipherPwd.process(bizCtx, "pay_pswd|pwd_typ|", "pay_pswd|msg_cd");
    if (Utils.eval(bizCtx, "is_succ(@O.msg_cd)=0")) {
        logger.info("step(3): if(is_succ(@O.msg_cd)=0) begin");
        PUBATCUtil.GDASetMsg(bizCtx, "@O.msg_cd", "");
        logger.info("step(3): if(is_succ(@O.msg_cd)=0) end");
    }
    /** 从SESSION获取免密标志，若为免密则跳过如下代码 **/
    +102:
    logger.info("调用交易验密     begin  ......");
    PUBATCUtil.callThirdServiceIgnore(bizCtx, "burmpub1", "0011471");
    if (Utils.eval(bizCtx, "~retcod=0")) {
        logger.info("step(45): if(~retcod=0) begin");
        if (Utils.eval(bizCtx, "is_succ($gda.msg_cd)=0")) {
            logger.info("step(46): if(is_succ($gda.msg_cd)=0) begin");
            PUBATCUtil.GDASetMsgIgnore(bizCtx, "$gda.msg_cd",
                    "$gda.msg_dat");
            logger.info("step(46): if(is_succ($gda.msg_cd)=0) end");
            return;
        }
        logger.info("step(45): if(~retcod=0) end");
    } else {
        logger.info("else begin");
        PUBATCUtil.GDASetMsgIgnore(bizCtx, "SCM60001", "");
        logger.info("else end");
        return;
    }
    logger.info("验证支付密码通过    end  ......");
    ```

- 快捷支付交易改造  `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090340`
    ```mermaid
    graph LR
        TSDK4090340(bsdkmca1.TSDK4090340<br/>下单快捷支付请求1.0)-->T0501041(bpaybrm1.T0501041<br/>快捷支付.支付原订单)
        T0501041-->2023020(bpwmppd3.2023020<br/>有账户快捷支付)
    ```
    
    ```java
    /** 从SESSION获取免密标志，若为免密则跳过如下代码 **/
    +83:
    logger.info("组装AES 解密所需的数据 begin");
    Utils.setData(bizCtx, "inp_dat.ocx_key", "$ocx_key");// 随机因子
    Utils.setData(bizCtx, "inp_dat.pswd_o", "$paypwd");// 登录密码
    logger.info("组装AES 解密所需的数据 end");

    logger.info("AES使用随机因子对密码进行解密   开始 ......");
    this.Pinconvert.process(bizCtx, "inp_dat", "out_dat|msg_cd|");
    logger.info("AES使用随机因子对密码进行解密  成功   结束 ......");
    Utils.setData(bizCtx, "pay_pswd", "@O.out_dat.pswd_c");

    Utils.setData(bizCtx, "gda.bus_typ", "0012");
    Utils.setData(bizCtx, "sign_key", "NO");
    Utils.setData(bizCtx, "pwd_typ", "PAY");
    logger.info("call package:[MCA.DecipherPwd]");
    DecipherPwd.process(bizCtx, "pay_pswd|pwd_typ|", "pay_pswd|msg_cd");
    if (Utils.eval(bizCtx, "is_succ(@O.msg_cd)=0")) {
        logger.info("step(9): if(is_succ(@O.msg_cd)=0) begin");
        PUBATCUtil.GDASetMsg(bizCtx, "@O.msg_cd", "");
        logger.info("step(9): if(is_succ(@O.msg_cd)=0) end");
        return;
    }
    Utils.setData(bizCtx, "pay_pswd", "@O.pay_psw");

    /** 从SESSION获取免密标志，若为免密则跳过如下代码 **/
    +139:
    logger.info("调用交易验密     begin  ......");
    PUBATCUtil.callThirdServiceIgnore(bizCtx, "burmpub1", "0011471");
    if (Utils.eval(bizCtx, "~retcod=0")) {
        logger.info("step(45): if(~retcod=0) begin");
        if (Utils.eval(bizCtx, "is_succ($gda.msg_cd)=0")) {
            logger.info("step(46): if(is_succ($gda.msg_cd)=0) begin");
            PUBATCUtil.GDASetMsgIgnore(bizCtx, "$gda.msg_cd","$gda.msg_dat");
            logger.info("step(46): if(is_succ($gda.msg_cd)=0) end");
            return;
        }
        logger.info("step(45): if(~retcod=0) end");
    } else {
        logger.info("else begin");
        PUBATCUtil.GDASetMsgIgnore(bizCtx, "SCM60001", "");
        logger.info("else end");
        return;
    }
    logger.info("验证支付密码通过    end  ......");
    ```

- 卡包支付交易改造 `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090660`
    ```mermaid
    graph LR
        TSDK4090660(bsdkmca1.TSDK4090660<br/>卡包支付接口)-->T2011106(brpmpub1.T2011106<br/>网上订单登记-录入金额组成)
        T2011106-->0000832(bcrdpub1.0000832<br/>卡包支付)
    ```

    ```java
    +56:
    /** 从SESSION获取免密标志，若为免密则跳过如下代码 **/
    logger.info("组装AES 解密所需的数据 begin");
    Utils.setData(bizCtx, "inp_dat.ocx_key", "$ocx_key");// 随机因子
    Utils.setData(bizCtx, "inp_dat.pswd_o", "$paypwd");// 登录密码
    logger.info("组装AES 解密所需的数据 end");

    logger.info("AES使用随机因子对密码进行解密   开始 ......");
    this.Pinconvert.process(bizCtx, "inp_dat", "out_dat|msg_cd|");
    logger.info("AES使用随机因子对密码进行解密  成功   结束 ......");
    Utils.setData(bizCtx, "pay_pswd", "@O.out_dat.pswd_c");

    Utils.setData(bizCtx, "gda.bus_typ", "0012");
    Utils.setData(bizCtx, "sign_key", "NO");
    Utils.setData(bizCtx, "pwd_typ", "PAY");
    logger.info("二层解密    begin");
    logger.info("call package:[MCA.DecipherPwd]");
    DecipherPwd.process(bizCtx, "pay_pswd|pwd_typ|", "pay_pswd|msg_cd");
    if (Utils.eval(bizCtx, "is_succ(@O.msg_cd)=0")) {
        logger.info("step(9): if(is_succ(@O.msg_cd)=0) begin");
        PUBATCUtil.GDASetMsg(bizCtx, "@O.msg_cd", "");
        logger.info("step(9): if(is_succ(@O.msg_cd)=0) end");
        return;
    }
    logger.info("二层解密  成功  end");
    /** 从SESSION获取免密标志，若为免密则跳过如下代码 **/
    +116:
    logger.info("调用交易验密     begin  ......");
    PUBATCUtil.callThirdServiceIgnore(bizCtx, "burmpub1", "0011471");
    if (Utils.eval(bizCtx, "~retcod=0")) {
        logger.info("step(45): if(~retcod=0) begin");
        if (Utils.eval(bizCtx, "is_succ($gda.msg_cd)=0")) {
            logger.info("step(46): if(is_succ($gda.msg_cd)=0) begin");
            PUBATCUtil.GDASetMsgIgnore(bizCtx, "$gda.msg_cd","$gda.msg_dat");
            logger.info("step(46): if(is_succ($gda.msg_cd)=0) end");
            return;
        }
        logger.info("step(45): if(~retcod=0) end");
    } else {
        logger.info("else begin");
        PUBATCUtil.GDASetMsgIgnore(bizCtx, "SCM60001", "");
        logger.info("else end");
        return;
    }
    logger.info("验证支付密码通过    end  ......");
    ```

- 支付成功后小额免密风控累计
    
    - 账户支付 `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090320`
        ```java
        +435:
        Utils.setData(bizCtx, "paytm", "get_date_time()");
        Utils.setData(bizCtx, "lastuptm", "get_date_time()");
        Utils.setData(bizCtx, "status", "@O.status");
        Utils.setData(bizCtx, "message_code", "@O.message_code");
        Utils.setData(bizCtx, "message_desc", "@O.message_desc");
        Utils.setData(bizCtx, "ret_sign", "@O.ret_sign");
        Utils.setData(bizCtx, "ret_cert", "@O.ret_cert");
        Utils.setData(bizCtx, "ret_data", "@O.ret_data");
        Utils.setData(bizCtx, "char_set", "@O.char_set");
        Utils.setData(bizCtx, "sign_type", "RSA");
        Utils.setData(bizCtx, "ordno", "$ord_no");
        Utils.setData(bizCtx, "ppdoutord", "$ord_no");
        Utils.setData(bizCtx, "mercordno", "$merc_ord_no");
        logger.info("拼装加密数据    暂时没有用到   end ");
        Utils.setData(bizCtx, "tx_nm", "APP-现金余额支付");
        Utils.setData(bizCtx, "tx_cd", "$gda.tx_cd");
        Utils.setData(bizCtx, "ip_addr", "@MSG.SIP");
        PUBATCUtil.invokeServiceIgnore(bizCtx, "0015070", "burmpub5",
                "burmpub5", "notify");

        /** 此处调用小额免密风控累计接口 **/

        PUBATCUtil.logETFIgnore(bizCtx);
        PUBATCUtil.GDASetMsg(bizCtx, "MCA00000", "");
        ```

    - 快捷支付 `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090340`
        ```java
        +485:
        logger.info("调用交易   去支付   begin ......");
        PUBATCUtil.callThirdServiceIgnore(bizCtx, "bpaybrm1", "0501041");
        if (Utils.eval(bizCtx, "~retcod!=0")) {
            logger.info("step(231): if(~retcod!=0) begin");
            PUBATCUtil.GDASetMsg(bizCtx, "MCA00005", "调用支付平台失败");
            logger.info("step(231): if(~retcod!=0) end");
            return;
        }
        if (Utils.eval(bizCtx, "is_succ($gda.msg_cd)=0")) {
            logger.info("step(234): if(is_succ($gda.msg_cd)=0) begin");
            PUBATCUtil.GDASetMsg(bizCtx, "$gda.msg_cd", "$gda.msg_dat");
            logger.info("step(234): if(is_succ($gda.msg_cd)=0) end");
            return;
        }
        logger.info("调用交易   去支付   end ......");
        
        Utils.setData(bizCtx,"paytm","get_date_time()");
        Utils.setData(bizCtx, "mercid", "$merc_id");
        Utils.setData(bizCtx, "token", "$token");
        Utils.setData(bizCtx, "ordamt", "$totamt");
        Utils.setData(bizCtx, "mercordno", "$merc_ord_no");
        Utils.setData(bizCtx, "ppdoutord", "$ppd_ord_no");
        Utils.setData(bizCtx, "qryneed", "$qryneed");
        Utils.setData(bizCtx, "tx_nm", "APP-快捷支付");
        Utils.setData(bizCtx, "tx_cd", "$gda.tx_cd");
        Utils.setData(bizCtx, "ip_addr", "@MSG.SIP");
        PUBATCUtil.invokeServiceIgnore(bizCtx, "0015070", "burmpub5",
                "burmpub5", "notify");

        /** 此处调用小额免密风控累计接口 **/

        PUBATCUtil.GDASetMsg(bizCtx, "MCA00000", "");
        ```

    - 卡包支付 `com.murong.ecp.app.egw.action.bsdkmca1.TSDK4090660`

        ```java
        +263:
        logger.info("调用交易   去支付   begin ......");
        PUBATCUtil.callThirdServiceIgnore(bizCtx, "brpmpub1", "2011106");
        if (Utils.eval(bizCtx, "~retcod!=0")) {
            logger.info("step(231): if(~retcod!=0) begin");
            PUBATCUtil.GDASetMsg(bizCtx, "MCA00005", "调用支付平台失败");
            logger.info("step(231): if(~retcod!=0) end");
            return;
        }
        if (Utils.eval(bizCtx, "is_succ($gda.msg_cd)=0")) {
            logger.info("step(234): if(is_succ($gda.msg_cd)=0) begin");
            logger.info("step(234): if(is_succ($gda.msg_cd)=0) end");
            return;
        }
        logger.info("调用交易   去支付   end ......");
        
        Utils.setData(bizCtx, "tx_nm", "APP-卡包支付");
        Utils.setData(bizCtx, "tx_cd", "$gda.tx_cd");
        Utils.setData(bizCtx, "ip_addr", "@MSG.SIP");
        PUBATCUtil.invokeServiceIgnore(bizCtx, "0015070", "burmpub5",
                "burmpub5", "notify");
        
        /** 此处调用小额免密风控累计接口 **/

        PUBATCUtil.GDASetMsgIgnore(bizCtx, "MCA00000", "");
        PUBATCUtil.logETFIgnore(bizCtx);
        logger.info("=================================>>卡包支付接口  end");
        ```

- 扫码支付查询交易改造
    > 需要返订单【是否免密标志】

    `com/murong/ecp/app/egw/mapper/bappmca1.TAPP4080806Mapper.xml`
    ```sql
    <!--此段SQL需要把小额免密标志字段一起查出来-->
    +3:
    <select id="qryOrdInf">
        <![CDATA[
        select ord_no,ord_amt,ord_sts,pay_cap_mod,merc_ord_no,cre_dt||cre_tm pay_tm,goods_nm,merc_nm from t_rpm_ord where ord_no=#{ord_no} and cre_dt=#{cre_dt}
        ]]>
    </select>
    ```

    `com.murong.ecp.app.egw.action.bappmca1.TAPP4080805`
    ```java
    +68:
    PUBATCUtil.readRecordBind(bizCtx, "qryOrdInf", "");
        if (Utils.eval(bizCtx, "~retcod=0")) {

            /** 此处需要把查询出来的免密标志添加到SESSION中去 **/
            
            logger.info("step(11): if(~retcod=0) begin");
            PUBATCUtil.GDASetMsg(bizCtx, "MCA00000", "交易成功");
            logger.info("step(11): if(~retcod=0) end");
        } else if (Utils.eval(bizCtx, "~retcod=2")) {
            logger.info("step(11): if(~retcod=2) begin");
            PUBATCUtil.GDASetMsg(bizCtx, "MCA10017", "未找到记录");
            logger.info("step(11): if(~retcod=2) end");
            return;
        } else {
    ```

- 付款码确认支付交易改造

    `com.murong.ecp.app.egw.action.bappmca1.TAPP4080806`

    ```java
    +81:
        PUBATCUtil.queryForObject(bizCtx, "qry_ord_sts");
        logger.info("ord_sts="+Utils.getData(bizCtx, "$ord_sts"));
        if(Utils.eval(bizCtx, "~retcod=0")){

            /** 此处需要判断订单是否已被支付(BD状态) **/

            if(Utils.eval(bizCtx, "is_equal_string($ord_sts,BE,BF)")){
                logger.info("if(is_equal_string($ord_sts,BE,BF)) begin");
                PUBATCUtil.GDASetMsgIgnore(bizCtx,"RPM75111","订单已关闭，无法支付");
                logger.info("if(is_equal_string($ord_sts,BE,BF)) end");
                return;
            }
            if(Utils.eval(bizCtx, "is_noequal_string($ord_sts,BA,BB,BM)")){
                logger.info("if(is_noequal_string($ord_sts,BA,BB,BM)) begin");
                PUBATCUtil.GDASetMsgIgnore(bizCtx, "MCA60025", "订单状态异常");
                logger.info("if(is_noequal_string($ord_sts,BA,BB,BM)) end");
                return;
            }
        }else {
            logger.info("step(13): else begin");
            PUBATCUtil.GDASetMsg(bizCtx, "RPM22201", "订单不存在");
            logger.info("step(13): else end");
            return;
        }
    ```

## APP改造

- 小额免密开关设置  `王雷、陈孝豪`
    > 按产品原型开发即可

- 轮训交易针对小额免密的支付成功的订单返回交互界面需改造 `王雷、陈孝豪`

    1. 查询交易改造
        > 商户扫码完成后的查询交易需要判别后台返回的支付结果，一旦支付结果为成功时，直接显示支付结果，无需再调用密码框跟支付流程
        > 若支付方式为账户支付，则需要判别小额免密标志是否免密，若免密则跳过密码输入框，直接进入短信验证的交互，最后调用支付接口完成支付
        > 详细请参照流程图

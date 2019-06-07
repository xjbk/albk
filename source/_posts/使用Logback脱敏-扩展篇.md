---
title: 使用Logback脱敏-扩展篇
date: 2019-06-07 15:34:54
tags:
---

#### 介绍
本文是一次数据泄漏之后的一点儿思考，系统日志对于后端系统而言是非常重要的，但是大多数开发人员在打印日志时，是非常随意的，不会去想太多，觉得日志打印的越多，排查总是越是方便。但是一些关键信息在打印日志时一定要注意，如果这些信息被人利用，可能会存在很大的风险。
下面我举例几个场景，想想这样的场景中存在的风险
>1、用户名、密码、交易密码等敏感信息，如果这些信息明文打印在日志，这些信息被人拿到之后，就存在很大的安全隐患，这种隐患可能会直接导致用户的损失，影响公司的形象。

>2、薪资系统中，姓名、薪资等敏感信息，在每次列表、详情查询的时候不加考虑全部打印日志，只要有日志查看权限的人员中就可能查看到大部分人的薪资情况，导致公司薪资信息泄漏，可能会导致同一团队的人员的薪资泄漏，造成人员流失等

其实对于这些场景，我们只要在日志打印时，进行脱敏处理就可以，最近公司存在一起数据泄漏的事件，在这个事件之后，公司也比较重视这一块，BOSS也因此很不开心，在此事之后，下班之余看了自己扩写了一个日志脱敏的工具，因为公司内部打印日志统一用的logback，本次扩展也是针对logback进行扩展


#### 主要思路
    每个团队甚至每个人打印日志的习惯和风格都不一样，基于日志格式输出的多样性，
    所以本人觉得最好的办法是通过配置脱敏规则，然后在日志输出之前进行脱敏处理
 整理了一下我们系统中日志打印的风格，大数数是以下三种
 * log.info("xxxxx: {} " , JSONObject.toJsonString(result));
 * log.info("xxxxx: {} " , result );
 * log.info("xxxxxx: " + result );
### 实现过程
 > 第一步：针对上述日志整理，我们先定义一下自己的脱敏规则，增加配置文件logback-desensitization-rule.properties，配置如下
 这里需要一些正则的知识，大家请自行补一下
```
#JSON字段中的mobile , telphone关键字对应的内容进行脱敏
RULE_REG_1=(\"mobile\"|\"telphone\")(:\")(\\w{3})(\\w{4})(\\w{4})*(\")&$1$2$3****$5$6
#JSON字段中的"证件号码"关键字对应的内容进行脱敏
RULE_REG_2=(\"idcard\")(:\")(\\w{2})(\\w{1,})(\\w{2})(\")&$1$2$3*********$5$6
#JSON字段中的"密码"对应的内容进行全部*显示 
RULE_REG_3=(\"password\")(:\")(\\w+)(\")&$1$2*****$4
#JSON字段中的"用户名"对应的内容除第一位，其它位脱敏显示  
RULE_REG_4=(\"customerName\"|\"userName\"|\"name\")(:\")([\u4E00-\u9FA5]{1})[\u4E00-\u9FA5]{1,}(\")&$1$2$3**$4
#log.info("password:{}",password);类似这样的日志，关键字后的8位中，后五位脱敏显示 
RULE_REG_5=(password|mobile)([:|=|,| ]+)(\\w{3})(\\w{5})&$1$2$3*****
```
> 第二步：继承MessageConverter实现日志脱敏
```
package com.bk.framework.extension.logback;

import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.classic.pattern.MessageConverter;
import ch.qos.logback.classic.spi.ILoggingEvent;
import com.google.common.collect.Lists;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.ArrayUtils;
import org.slf4j.LoggerFactory;

import java.util.List;
import java.util.Map;
import java.util.regex.Pattern;

/**
 * @author BK
 * @version V2.0
 * @description: logback扩展消息转换器  支持正则脱敏
 * @date 2019/6/1 21:23
 */
@Slf4j
public class DesensitizationMessageConverter extends MessageConverter {
    private static volatile List<RuleConfig> configList;

    @Override
    public String convert(ILoggingEvent event) {
        initRuleConfig();
        return doConvert(event);
    }

    /**
     * 初始化规则配置
     */
    private void initRuleConfig() {
        if (configList == null) {
            synchronized (DesensitizationMessageConverter.class) {
                if (configList == null) {
                    configList = Lists.newArrayList();
                    Map<String, String> propertyMap = ((LoggerContext) LoggerFactory.getILoggerFactory()).getCopyOfPropertyMap();
                    for (String s : propertyMap.keySet()) {
                        if (s.startsWith("RULE_REG_")) {
                            String[] array = propertyMap.get(s).split("&");
                            if (ArrayUtils.isNotEmpty(array) && array.length == 2) {
                                configList.add(new RuleConfig(array[0], array[1]));
                            }
                        }
                    }
                    log.info("desensitization rule config init end ! ");
                }
            }
        }
    }

    /**
     * 日志内容转换
     *
     * @param event
     * @return
     */
    private String doConvert(ILoggingEvent event) {
        String result = event.getFormattedMessage();
        if (configList != null) {
            for (RuleConfig ruleConfig : configList) {
                result = ruleConfig.apply(result);
            }
        } else {
            result = super.convert(event);
        }
        return result;
    }

    @Data
    private class RuleConfig {
        private String reg;
        private String replacement;

        RuleConfig(String reg, String replacement) {
            this.reg = reg;
            this.replacement = replacement;
        }

        String apply(String message) {
            return Pattern.compile(reg).matcher(message).replaceAll(replacement);
        }
    }
}

```
> 第三步：增加logback.xml文件，整合规则与转换逻辑
* 1、引入我们第一步配置的脱敏规则属性文件（这里要注意，属性的scope，默认是local，详细配置可以参考logback官网）
* 2、配置我们第二步增加的转换器，同时配置conversionWord="xxx"，当日志输出的pattern中引用%xxx时，会通过此转换顺进日志输出的转换
配置如下
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property  scope="context" resource="./logback-desensitization-rule.properties"/>

    <conversionRule conversionWord="msg"
                    converterClass="com.bk.framework.extension.logback.DesensitizationMessageConverter"/>
    <conversionRule conversionWord="rid" converterClass="com.bk.framework.extension.logback.ClassicConverterExt"/>
    <property name="CONSOLE_LOG_PATTERN"
              value="%date{yyyy-MM-dd HH:mm:ss} | %boldYellow(%rid) | %highlight(%-5level) | %boldYellow(%thread) | %boldGreen(%logger) | %msg%n"/>
    <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="stdout"/>
    </root>
</configuration>
```


#### 测试效果
1. 编辑测试类
```aidl
package com.bk.framework.extension;

import lombok.extern.slf4j.Slf4j;

/**
 * @author BK
 * @version V2.0
 * @date 2019-06-03 23:43
 */
@Slf4j
public class ExtensionApplication {

    public static void main(String[] args) {
        log.info("mobile:{}", "13588889999");
        log.info("userInfo:{}", "{\n" + "            \"userName\":\"罗志祥\"，\n" + "            \"idcard\":\"321183197701017846\",\n" + "            \"password\":\"luozhixiang1234\",\n" + "            \"mobile\":\"18888888888\"\n" + "        }");
    }
}
```
2. 运行main方法
![运行结果](title/2019_06_06_img1.jpg "运行结果")
>可以看到，只需要增加一个转换器，配置几个正则，就可以实现日志的脱敏，希望可以帮到大家


#### 关于优化的一点儿思路
其实到现在，我们可以实现日志脱敏，但是做法有些丑陋，不是很美观，可以推荐一个思路
> 通过DTD或得schema文件 ，定义自己的XML格式文件
* 设置对于json，xml，等不同式日志的脱敏关键字的replacement等信息，主要目的是为了简化配置，封装正则的配置难度，后台根据配置自动生成正则和替换规则
* 在logback start时，解析配置文件，读取配置生成相应的正则，进行脱敏

#### 一点儿建议

1. 过大对象一般不建议打日志
    对于一些过大的对象，比如查询列表的结果建议不要打印，如果需要打印的话，建议在DEBUG级别打印
一般这种日志，对于问题排查意义不大，而且这种日志导致日志文件巨增，同时如果日志被窃取会存在信息泄漏的风险
2. 打印日志时一定要有危机意识
    哪些日志可以打印，如些日志打印会有风险，一定要做到心里有数
3. 打印error级别日志时，建议使用log.error("xxxxxx" ,e );
   不要使用log.error("xxxxxx" ,e.getMessage() );

#### 参与贡献

https://logback.qos.ch/manual/configuration.html

#### 写在最后

这点儿东西写了三遍，简书是不是最近有问题，图片展示不了，没有发布的内容总是丢失，这已经是第三遍了。

> 欢迎大家讨论，本人才疏学浅，有不正确的地方还请斧正。
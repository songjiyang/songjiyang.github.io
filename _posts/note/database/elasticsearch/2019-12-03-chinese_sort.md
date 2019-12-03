---
bg: 'myes.jpg'
layout: post
title:  "5.1.1版本elasticsearch中英文排序"
crawlertitle: "elasticsearch"
summary: ""
date:   2019-12-03 12:00:00 +0800
categories: posts
tags: 'elasticsearch'
author: 宋天
---


5.1.1版本elasticsearch中英文排序问题排查及解决

# 环境
- elasticsearch：5.1.1
- 分词插件mmseg: 5.1.1
- 拼音插件pinyin: 5.1.1

# 配置

```
"analyzer": {
          "pinyin_raw": {
            "type": "pinyin",
            "keep_first_letter": false,
            "keep_full_pinyin": false,
            "keep_joined_full_pinyin": true,
            "keep_none_chinese_in_joined_full_pinyin": true,
            "keep_none_chinese_in_first_letter": true,
            "none_chinese_pinyin_tokenize": false,
            "lowercase": true
          }
          ...

```

# 需求
 想对文件名进行按照字典顺序排序，文件名有中文和英文，中文使用pinyin插件转化为拼音
 
# 问题
1. 纯英文或纯数字混合分词结果多一个空格
   
 ```
 http://ec2-52-81-100-48.cn-north-1.compute.amazonaws.com.cn:9200/test_new/_analyze?text=36kr&analyzer=pinyin_raw
 
 
 {
    "tokens": [
        {
            "token": "36kr",
            "start_offset": 0,
            "end_offset": 3,
            "type": "word",
            "position": 0
        },
        {
            "token": "",
            "start_offset": 0,
            "end_offset": 0,
            "type": "word",
            "position": 1
        }
    ]
}
```

2. 英文和中文混合时，不能正确拼接在一起, 且英文总是排在最前面

```
http://ec2-52-81-100-48.cn-north-1.compute.amazonaws.com.cn:9200/test_new/_analyze?text=哈哈dc&analyzer=pinyin_raw

{
    "tokens": [
        {
            "token": "dc",
            "start_offset": 1,
            "end_offset": 3,
            "type": "word",
            "position": 0
        },
        {
            "token": "haha",
            "start_offset": 0,
            "end_offset": 4,
            "type": "word",
            "position": 1
        }
    ]
}

本来应该是hahadc
```

# 解决过程
1. 问题1之前pinyin插件的issue里面有人提到过，增加一个空白过滤器就可以去掉这个空白字符 [issue](https://github.com/medcl/elasticsearch-analysis-pinyin/issues/163), 我自己也尝试过有效，但这个过滤器可以不加，升级es可以解决，所以下面测试的配置还是上面的配置。
2. 问题2查看pinyin插件的配置文档看到keep_none_chinese_in_joined_full_pinyin这个配置会将非中文和英文连接起来，但结果却没有，经过排查，发现pinyin插件5.1.1是不支持这个参数，由5.2.0的relase的commit可以看出是在这个版本才支持keep_none_chinese_in_joined_full_pinyin [pinyin插件5.2.0commit记录](https://github.com/medcl/elasticsearch-analysis-pinyin/compare/v5.2.0...master), 并且解决了[#80issue](https://github.com/medcl/elasticsearch-analysis-pinyin/issues/80), 在这个issue里面同时出现了上述两个问题，拼音插件的作者回复下载5.2.2可以解决上述问题

所以最终打算下载es5.2.2，并按照mmseg和pinyin5.2.2

# 测试5.2.2分词

在相同配置的情况下

```
http://52.82.10.41:9200/test_new/_analyze?analyzer=pinyin_raw&text=36kr

{
    "tokens": [
        {
            "token": "36kr",
            "start_offset": 0,
            "end_offset": 4,
            "type": "word",
            "position": 0
        }
    ]
}



http://52.82.10.41:9200/test_new/_analyze?analyzer=pinyin_raw&text=哈哈dc
{
    "tokens": [
        {
            "token": "hahadc",
            "start_offset": 0,
            "end_offset": 6,
            "type": "word",
            "position": 0
        }
    ]
}
```

已经解决了上述问题并且满足了我们的需要


# 结论

分词出现空白和中英文连接不生效的情况是elasticsearch和插件版本较低导致， 我们的最终解决方式是升级elasticsearch和插件到5.2.2， 因为升级es可以会对应用整体造成影响，升级之前需要先评估应用对es的依赖性。
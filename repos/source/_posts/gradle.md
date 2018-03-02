---
title: Gradle 中文乱码问题
date: 2018-03-02 13:58:12
categories: Android
tags: [Gradle, 乱码问题]
---

## Gradle

使用gradle读取gradle.properties文件时容易出现中文乱码问题, 后来在stackoverflow上找到答案：
[https://stackoverflow.com/questions/44632375/referencing-a-japanese-chinese-string-in-gradle-properties-from-strings-xml-resu](https://stackoverflow.com/questions/44632375/referencing-a-japanese-chinese-string-in-gradle-properties-from-strings-xml-resu)


```java

COMPILE_SDK_VERSION = 26
BUILD_TOOL_VERSION = 26.0.2
MIN_SDK_VERSION = 17
TARGET_SDK_VERSION = 19
VERSION_CODE = 4
VERSION_NAME = 1.1.6
APP_NAME_ONLINE = 应用名称
APP_NAME_DEVElOP = 应用名称Dev
APP_NAME_PRE_ONLINE = 应用名称Pre

```   

直接使用

```java

  // 字符格式转换 
  def transform(String targetStr) {
      return new String(targetStr.getBytes("iso8859-1"), "UTF-8")
  }


  productFlavors {
        develop {
            applicationIdSuffix '.develop'
            buildConfigField "boolean", "developMode", "true"
            resValue "string", "app_name", transform(APP_NAME_DEVElOP)
        }

        preOnline {
            applicationIdSuffix '.preonline'
            buildConfigField "boolean", "developMode", "false"
            resValue "string", "app_name", transform(APP_NAME_PRE_ONLINE)
        }

        online {
            buildConfigField "boolean", "developMode", "false"
            resValue "string", "app_name", transform(APP_NAME_ONLINE)
        }
    }
```
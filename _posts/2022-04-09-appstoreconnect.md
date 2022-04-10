---
layout:     post
title:      如何抓取appstoreconnect数据
category:   [golang,ruby]
tags:       [spaceship,appstoreconnect]
description: 本文主要介绍如何通过API方式拉取appstoreconnect的数据
---


## App Store Connect API

不需要登录，也不需要提供开发者账号密码，它是通过私钥生成 token 来访问 API 的

它是官方 API，维护有保障，即使后续苹果调整了网页内容，其相应 API 也不会受影响。

不过它的功能受限很大，提供的 API 很少，主要包括下面这些：

- **TestFlight**. Manage beta builds of your app, testers, and groups.

- **Xcode Cloud**. Read Xcode Cloud data, manage workflows, and start builds.

- **Users and Roles**. Send invitations for users to join your team. Adjust their level of access or remove users.

- **Provisioning**. Manage bundle IDs, capabilities, signing certificates, devices, and provisioning profiles.

- **App Metadata**. Create new versions, manage App Store information, and submit your app to the App Store.

- **App Clip Experiences**. Create an App Clip and manage App Clip experiences.

- **Reporting**. Download sales and financial reports.

- **Power and Performance Metrics**. Download aggregate metrics and diagnostics for App Store versions of your app

提供的这些 API 远远不能满足我们的方案，然后去 github 上搜索还没有有其他解决方案。

大概思路就是有没有一种方案就是模拟人为操作页面，通过监控访问接口，把对应的返回数据解析出来。

首先找到了 Python Selenium，不过它是通过浏览器来实现的，对于一个后台服务来说，不太方便

然后找了 fastlane Spaceship，它是用 ruby 语言写的一个第三方库

## Spaceship

Spaceship 非常强大，所有你在浏览器上对 Apple Developer Center 和 App Store Connect 的操作几乎都可以通过 Spaceship 命令解决，所有 App Store Connect API 能实现的功能，Spaceship 几乎都能实现，有些 App Store Connect API 没有的功能，Spaceship 也能实现。

它主要是通过模拟网页操作，分析和抓取网页的接口和数据来实现相应功能。通过接口访问即可，不需要浏览器的支持

Overview of the used API endpoints

- https://idmsa.apple.com:
  - Used to authenticate to get a valid session

- https://developerservices2.apple.com:
  - Get a list of all available provisioning profiles
  - Register new devices

- https://developer.apple.com:
  - List all devices, certificates, apps and app groups
  - Create new certificates, provisioning profiles and apps
  - Disable/enable services on apps and assign them to app groups
  - Delete certificates and apps
  - Repair provisioning profiles
  - Download provisioning profiles
  - Team selection

- https://appstoreconnect.apple.com:
  - Managing apps
  - Managing beta testers
  - Submitting updates to review
  - Managing app metadata

- https://du-itc.appstoreconnect.apple.com:
  - Upload icons, screenshots, trailers ...

- https://is[1-9]-ssl.mzstatic.com:
  - Download app screenshots from App Store Connect

虽然它这么强大，但是也有一些缺点：

如果你的开发者账号开了双重认证，那登录的时候还需要验证码。验证码这个东西使整体可用性大大降低

它是非官方 API，所以后续如果苹果有改动，可能就用不了了

不过目前对于我们来说，这些缺点也不是很大问题

通过一段时间的调试，终于可以跑通了！

由于平时主要使用 golang，没用过 ruby，所以尽可能地减少 ruby 的使用。

我们最终的实现方案是用 ruby 模拟登录后，把 cookie 信息保存到本地文件中，然后再用 golang 读取 cookie，加到 header 头中，再通过 http 发起请求，后续的处理完全用 golang 就可以了

## 安装 ruby

我是在 mac 电脑上开发的，所以安装也是基于 mac 系统的

### 使用 homebrew 安装 rbenv

```
brew install rbenv
```

### 安装 ruby

```
rbenv install 2.7.0
```

安装成功后，我们让其在本地环境中生效：

```
rbenv shell 2.7.0
```

rbenv 常用命令解释

```
rbenv install --list # 列出所有 ruby 版本
rbenv install 2.0.0-p247 # 安装需要的ruby版本
rbenv versions # 所有已安装的ruby版本
rbenv global 1.8.7-p352 # 设置全局ruby版本
rbenv local 1.9.7 # 设置当前文件夹ruby版本
rbenv uninstall 2.00 #卸载指定ruby版本
```

输入上述命令后，可能会有报错。rbenv 提示我们在 .zshrc 中增加一行 eval "$(rbenv init -)" 语句来对 rbenv 环境进行初始化。如果报错，我们增加并重启终端即可。

```
$ ruby -v
ruby 2.7.0p0 (2019-12-25 revision 647ee6f091) [x86_64-darwin20]
$ which ruby
/Users/lee980/.rbenv/shims/ruby
```

## ruby 模拟登录

我用的 IDE 是 rubyMine，不需要收费

首先，需要把我们要调用的第三方库引进来

```
sudo chown -R tony /Library/Ruby/Gems
gem install fastlane
gem install webmock
```

具体调用代码如下

test.rb

```
require 'Spaceship'
require "webmock"

def itc_read_fixture_file(filename)
  File.read(File.join('./', filename))
end

def stub_request(*args)
  WebMock::API.stub_request(*args)
end

token = Spaceship::ConnectAPI::Token.create()
Spaceship::ConnectAPI.token = token

# 登录
Spaceship::Tunes.login(ENV['APPSTORECONNECT_USERNAME'], ENV['APPSTORECONNECT_PASSWORD'])
puts "login success"

return
```

## golang 拉取数据

代码

```
package item

import (
    "bufio"
    "encoding/json"
    "fmt"
    "io"
    "os"
    "strings"
    "tag_manage_api/common"
    "tag_manage_api/components/logger"
    "tag_manage_api/config"
    "tag_manage_api/services"
    "tag_manage_api/utils/http_client"
    "time"
)

/**
* HandleWebReferrer 处理数据
* 1. 拉取数据列表
* 2. 删除 ADAM_ID、DATE 的旧数据
* 3. 插入新的数据
*/

func HandleWebReferrer(adamId, date string) error {
    appData := GetSourceListOfWeb(adamId, date)
    if appData == nil {
        logger.Log.Errorf("get webReferre error")
        return fmt.Errorf("get webReferre error")
    }
    if len(appData.Results) == 0 {
        logger.Log.Infof("webReferre is empty")
        return nil
    }

    today := time.Now().Format(common.FormatDate)
    //删除数据
    err := services.DeleteWebReferrerDataByDate(today, date, adamId)
    if err != nil {
        return err
    }

    //byteList, _ := json.Marshal(appData)
    //logger.Log.Infof("appData:%s", string(byteList))

    for idx, result := range appData.Results {
        logger.Log.Infof("HandleWebReferrer idx:%d", idx)
        addData := &services.WebReferrerData{
            Dt:          today,
            InstallDate: date,
            AdamId:      adamId,
            Referrer:    result.DomainReferrer,
        }
        for measure, data := range result.Data {
            switch measure {
                case "sessions":
                addData.Sessions = int(data.Value)
                case "impressionsTotal":
                addData.Impressions = int(data.Value)
                case "totalDownloads":
                addData.Downloads = int(data.Value)
                case "proceeds":
                addData.Proceeds = data.Value
            }
        }
        //插入
        err = services.AddWebReferrerData(addData)
        if err != nil {
            return err
        }
    }

    return nil
}

/**
* HandleAppReferrer 处理数据
* 1. 拉取数据列表
* 2. 删除 ADAM_ID、DATE 的旧数据
* 3. 插入新的数据
*/

func HandleAppReferrer(adamId, date string) error {
    appData := GetSourceListOfApp(adamId, date)
    if appData == nil {
        logger.Log.Errorf("get appReferre error")
        return fmt.Errorf("get appReferre error")
    }
    if len(appData.Results) == 0 {
        logger.Log.Infof("appReferre is empty")
        return nil
    }

    //删除数据
    today := time.Now().Format(common.FormatDate)
    err := services.DeleteAppReferrerDataByDate(today, date, adamId)
    if err != nil {
        return err
    }

    //byteList, _ := json.Marshal(appData)
    //logger.Log.Infof("appData:%s", string(byteList))

    for idx, result := range appData.Results {
        logger.Log.Infof("HandleAppReferrer idx:%d", idx)
        addData := &services.AppReferrerData{
            Dt:          today,
            InstallDate: date,
            AdamId:      adamId,
            Referrer:    result.AppReferrer,
            Title:       result.Title,
            IconUrl:     result.IconUrl,
        }
        for measure, data := range result.Data {
            switch measure {
                case "sessions":
                addData.Sessions = int(data.Value)
                case "impressionsTotal":
                addData.Impressions = int(data.Value)
                case "totalDownloads":
                addData.Downloads = int(data.Value)
                case "proceeds":
                addData.Proceeds = data.Value
            }
        }
        //插入
        err = services.AddAppReferrerData(addData)
        if err != nil {
            return err
        }
    }

    return nil
}

/**
* HandleAppTotalData 处理数据
* 1. 拉取数据列表
* 2. 删除ADAM_ID、DATE 的就数据
* 3. 插入新的数据
*/

func HandleAppTotalData(adamId, date string) error {
    appData := GetMeasuresOfTotal(adamId, date)
    if appData == nil {
        logger.Log.Errorf("get appData error")
        return fmt.Errorf("get appData error")
    }
    if len(appData.Results) == 0 {
        logger.Log.Infof("app data is empty")
        return nil
    }

    //删除数据
    today := time.Now().Format(common.FormatDate)
    err := services.DeleteAppTotalDataByDate(today, date, adamId)
    if err != nil {
        return err
    }

    //byteList, _ := json.Marshal(appData)
    //logger.Log.Infof("appData:%s", string(byteList))

    addData := &services.AppTotalData{
        Dt:          today,
        InstallDate: date,
        AdamId:      adamId,
    }

    for _, cpp := range appData.Results {
        var value float64
        for _, data := range cpp.Data {
            if data.Date[:10] == date {
                value = data.Value
                break
            }
        }
        switch cpp.Measure {
            case "pageViewCount":
            addData.PageView = int(value)
            case "activeDevices":
            addData.ActiveDevices = int(value)
            case "sessions":
            addData.Sessions = int(value)
            case "payingUsers":
            addData.PayingUsers = int(value)
            case "crashes":
            addData.CRASHES = int(value)
            case "impressionsTotal":
            addData.Impressions = int(value)
            case "totalDownloads":
            addData.Downloads = int(value)
            case "proceeds":
            addData.Proceeds = value
            case "conversionRate":
            addData.ConversionRate = value
        }
    }
    //插入
    err = services.AddAppTotalData(addData)

    return err
}

/**
* HandleCPP处理CPP数据
* 1. 拉取CPP列表
* 2. 判断CPP是否存在sf，如果不存在则插入
* 3. 根据CPP_ID、DATE 拉取报表数据
* 4. 删除CPP_ID、DATE 的就数据
* 5. 插入新的数据
*/

func HandleCPP(adamId, date string) error {
    //获取CPPInfo信息，如果不存在则写入info表
    cppInfo := GetCppList(adamId)
    if cppInfo == nil {
        logger.Log.Errorf("get cppList error")
        return fmt.Errorf("get cppList error")
    }
    if len(cppInfo.Results) == 0 {
        logger.Log.Infof("cpp info is empty")
        return nil
    }
    cppIds := make([]string, 0, len(cppInfo.Results))
    for _, cpp := range cppInfo.Results {
        cppIds = append(cppIds, cpp.Id)
        data := &services.CPPInfo{
            CPPId:   cpp.Id,
            CPPName: cpp.Name,
            AdamId:  adamId,
        }
        if services.IsCPPInfoExist(data) {
            logger.Log.Infof("cpp_id:%s cpp_name:%s has exist", cpp.Id, cpp.Name)
            continue
        }
        //插入表
        services.AddCPPInfo(data)
    }

    cppData := GetMeasuresOfCPP(adamId, date, cppIds)
    if cppData == nil {
        logger.Log.Errorf("get cppData error")
        return fmt.Errorf("get cppData error")
    }
    if len(cppData.Results) == 0 {
        logger.Log.Infof("cpp data is empty")
        return nil
    }

    //删除数据
    today := time.Now().Format(common.FormatDate)
    err := services.DeleteCPPDataByDate(today, date, cppIds)
    if err != nil {
        return err
    }

    cppDataMap := map[string]*services.CPPData{}

    //byteList, _ := json.Marshal(cppData)
    //logger.Log.Infof("cppIds:%+v cppData:%s", cppIds, string(byteList))

    for _, cpp := range cppData.Results {
        cppId := cpp.Cohort.ProductPage
        addData, ok := cppDataMap[cppId]
        if !ok {
            addData = &services.CPPData{
                Dt:          today,
                InstallDate: date,
                AdamId:      adamId,
                CPPId:       cppId,
            }
            cppDataMap[cppId] = addData
        }
        var value float64
        for _, data := range cpp.Data {
            if data.Date[:10] == date {
                value = data.Value
                break
            }
        }
        switch cpp.Measure {
            case "pageViewCount":
            addData.Impressions = int(value)
            case "totalDownloads":
            addData.Downloads = int(value)
            case "conversionRate":
            addData.ConversionRate = value
        }
    }
    //插入
    for _, addData := range cppDataMap {
        err = services.AddCPPData(addData)
        if err != nil {
            return err
        }
    }

    return nil
}

/***** 下面是获取原始数据 *****/

func GetSourceListOfApp(adamId string, date string) *common.SourceList {
    headers := map[string]string{
        "Content-Type":   "application/json",
        "Accept":         "application/json, text/javascript",
        "X-Requested-By": "dev.apple.com",
    }
    body := map[string]interface{}{
        "dimension": "appReferrer",
        "adamId":    []string{adamId},
        "startTime": nil,
        "endTime":   date + "T00:00:00Z",
        "frequency": "day",
        "limit":     1,
        "measures":  []string{"impressionsTotal", "totalDownloads", "proceeds", "sessions"},
    }
    cookies := getCookie(config.Config.AppstoreConnect.CookieFile)
    url := "https://appstoreconnect.apple.com/analytics/api/v1/data/sources/list"

    respStatus, respBody := http_client.Post(url, headers, cookies, body)
    if respStatus != 200 {
        logger.Log.Errorf("respStatus:%d respData:%s", respStatus, string(respBody))
        return nil
    }
    result := &common.SourceList{}
    err := json.Unmarshal(respBody, result)
    if err != nil {
        logger.Log.Errorf("unmarshal error::%s result:%s", err.Error(), string(respBody))
        return nil
    }

    return result
}

func GetSourceListOfWeb(adamId string, date string) *common.SourceList {
    headers := map[string]string{
        "Content-Type":   "application/json",
        "Accept":         "application/json, text/javascript",
        "X-Requested-By": "dev.apple.com",
    }
    body := map[string]interface{}{
        "dimension": "domainReferrer",
        "adamId":    []string{adamId},
        "startTime": nil,
        "endTime":   date + "T00:00:00Z",
        "frequency": "day",
        "limit":     1,
        "measures":  []string{"impressionsTotal", "totalDownloads", "proceeds", "sessions"},
    }
    cookies := getCookie(config.Config.AppstoreConnect.CookieFile)
    url := "https://appstoreconnect.apple.com/analytics/api/v1/data/sources/list"

    respStatus, respBody := http_client.Post(url, headers, cookies, body)
    if respStatus != 200 {
        logger.Log.Errorf("respStatus:%d respData:%s", respStatus, string(respBody))
        return nil
    }
    result := &common.SourceList{}
    err := json.Unmarshal(respBody, result)
    if err != nil {
        logger.Log.Errorf("unmarshal error::%s result:%s", err.Error(), string(respBody))
        return nil
    }

    return result
}

func GetMeasuresOfTotal(adamId string, date string) *common.MeasuresListResult {
    headers := map[string]string{
        "Content-Type":   "application/json",
        "Accept":         "application/json, text/javascript",
        "X-Requested-By": "dev.apple.com",
    }
    body := map[string]interface{}{
        "adamId":    []string{adamId},
        "startTime": nil,
        "endTime":   date + "T00:00:00Z",
        "frequency": "day",
        "measures":  []string{"pageViewCount", "iap", "activeDevices", "sessions", "payingUsers", "crashes", "impressionsTotal", "impressionsTotalUnique", "totalDownloads", "proceeds", "conversionRate"},
    }
    cookies := getCookie(config.Config.AppstoreConnect.CookieFile)
    url := "https://appstoreconnect.apple.com/analytics/api/v1/data/app/detail/measures"

    respStatus, respBody := http_client.Post(url, headers, cookies, body)
    if respStatus != 200 {
        logger.Log.Errorf("respStatus:%d respData:%s", respStatus, string(respBody))
        return nil
    }
    result := &common.MeasuresListResult{}
    err := json.Unmarshal(respBody, result)
    if err != nil {
        logger.Log.Errorf("unmarshal error::%s result:%s", err.Error(), string(respBody))
        return nil
    }

    return result
}

// GetCppList CPP列表页面
func GetCppList(adamId string) *common.CPPList {
    body := map[string]string{}
    url := "https://appstoreconnect.apple.com/analytics/api/v1/app-info/" + adamId + "/cpp"
    headers := map[string]string{
        "Content-Type":   "application/json",
        "Accept":         "application/json, text/javascript",
        "X-Requested-By": "dev.apple.com",
    }
    cookies := getCookie(config.Config.AppstoreConnect.CookieFile)

    respStatus, respBody := http_client.Get(url, headers, cookies, body)
    if respStatus != 200 {
        logger.Log.Errorf("respStatus:%d respData:%s", respStatus, string(respBody))
        return nil
    }
    result := &common.CPPList{}
    err := json.Unmarshal(respBody, result)
    if err != nil {
        logger.Log.Errorf("unmarshal error::%s result:%s", err.Error(), string(respBody))
        return nil
    }

    return result
}

func GetMeasuresOfCPP(adamId, date string, cppIds []string) *common.MeasuresListResult {
    headers := map[string]string{
        "Content-Type":   "application/json",
        "Accept":         "application/json, text/javascript",
        "X-Requested-By": "dev.apple.com",
    }
    body := map[string]interface{}{
        "adamId":    []string{adamId},
        "startTime": nil,
        "endTime":   date + "T00:00:00Z",
        "frequency": "day",
        "measures":  []string{"pageViewCount", "totalDownloads", "conversionRate"},
        "dimensionFilters": []map[string]interface{}{
            {
                "dimensionKey": "productPage",
                "optionKeys":   cppIds,
            },
        },
    }
    cookies := getCookie(config.Config.AppstoreConnect.CookieFile)
    url := "https://appstoreconnect.apple.com/analytics/api/v1/data/app/detail/measures"

    respStatus, respBody := http_client.Post(url, headers, cookies, body)
    if respStatus != 200 {
        logger.Log.Errorf("respStatus:%d respData:%s", respStatus, string(respBody))
        return nil
    }
    result := &common.MeasuresListResult{}
    err := json.Unmarshal(respBody, result)
    if err != nil {
        logger.Log.Errorf("unmarshal error::%s result:%s", err.Error(), string(respBody))
        return nil
    }

    return result
}

func getCookie(filepath string) map[string]string {
    // 读取一个文件的内容
    file, err := os.Open(filepath)
    if err != nil {
        fmt.Println("open file err:", err.Error())
        return map[string]string{}
    }
    // 处理结束后关闭文件
    defer file.Close()

    // 使用bufio读取
    r := bufio.NewReader(file)
    var str string
    var name, value string

    result := map[string]string{}

    for {
        // 分行读取文件  ReadLine返回单个行，不包括行尾字节(\n  或 \r\n)
        data, _, err := r.ReadLine()

        // 读取到末尾退出
        if err == io.EOF {
            break
        }
        if err != nil {
            fmt.Println("read err", err.Error())
            break
        }
        str = string(data)
        if strings.Contains(str, "ruby/object:HTTP::Cookie") {
            continue
        }
        parts := strings.Split(str, ":")
        if len(parts) != 2 {
            continue
        }
        if strings.TrimSpace(parts[0]) == "name" {
            name = strings.TrimSpace(parts[1])
            continue
        }
        if strings.TrimSpace(parts[0]) == "value" {
            value = strings.TrimSpace(parts[1])
            result[name] = value
            continue
        }
    }

    return result
}

```

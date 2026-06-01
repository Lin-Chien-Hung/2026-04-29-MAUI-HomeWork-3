# ISSUE
HTTPS SSL 憑證，怎麼迴避這個問題 ? OR 驗證
ssl 發行、server & crt 匯入 or 迴避
Community Toolkit 套件影片 or 網站

# .NET MAUI HTTPS / SSL 憑證處理與 Community Toolkit 筆記

---

# 目錄

- [HTTPS SSL 憑證驗證流程](#https-ssl-憑證驗證流程)
- [SSL 錯誤原因](#ssl-錯誤原因)
- [正式環境處理方式](#正式環境處理方式)
- [測試環境處理方式](#測試環境處理方式)
- [迴避 SSL 驗證方式](#迴避-ssl-驗證方式)
- [Certificate Pinning](#certificate-pinning)
- [Android SSL 設定](#android-ssl-設定)
- [Localhost HTTPS 開發](#localhost-https-開發)
- [Community Toolkit](#community-toolkit)
- [常用 Community Toolkit 元件](#常用-community-toolkit-元件)
- [面試重點整理](#面試重點整理)

---

# HTTPS SSL 憑證驗證流程

當 MAUI 呼叫 HTTPS API 時：

```text
MAUI App
    ↓
HTTPS Request
    ↓
Server SSL Certificate
    ↓
Certificate Validation
    ↓
成功 / 失敗
```

驗證內容：

```text
✔ 憑證是否過期

✔ 憑證是否可信任

✔ HostName 是否一致

✔ Certificate Chain 是否完整

✔ Root CA 是否存在
```

---

# SSL 錯誤原因

常見錯誤：

```text
The SSL connection could not be established
```

```text
RemoteCertificateChainErrors
```

```text
AuthenticationException
```

---

## 可能原因

### 1. Self-Signed Certificate

自簽憑證

```text
Server 自己發自己
```

Client 不信任。

---

### 2. 憑證過期

```text
Certificate Expired
```

---

### 3. HostName 不符

例如：

```text
憑證：
api.company.com

實際：
192.168.1.100
```

驗證失敗。

---

### 4. Root CA 未安裝

```text
Root CA Missing
```

---

### 5. 憑證鏈不完整

```text
Root CA
    ↓
Intermediate CA
    ↓
Server Certificate
```

缺少任何一層都可能失敗。

---

# 正式環境處理方式

## 使用正式 CA 憑證

推薦：

- DigiCert
- Sectigo
- Let's Encrypt

---

## 憑證架構

```text
Root CA
    ↓
Intermediate CA
    ↓
Server Certificate
```

---

## 匯入 IIS

```text
IIS
    ↓
Server Certificates
    ↓
Import
    ↓
server.pfx
```

---

## 匯入 Linux Nginx

```nginx
ssl_certificate server.crt;
ssl_certificate_key private.key;
```

---

# 測試環境處理方式

## 建立 Self-Signed Certificate

PowerShell：

```powershell
New-SelfSignedCertificate `
-DnsName "192.168.1.100" `
-CertStoreLocation "cert:\LocalMachine\My"
```

---

## 匯出

```text
server.pfx
server.crt
```

---

## 匯入 Server

```text
IIS
    ↓
Server Certificates
    ↓
Import
```

---

## 匯入 Client

### Windows

```text
Trusted Root Certification Authorities
```

---

### Android

```text
設定
    ↓
安全性
    ↓
安裝憑證
```

---

# 迴避 SSL 驗證方式

> ⚠️ 僅限開發環境

---

## HttpClientHandler

```csharp
var handler = new HttpClientHandler();

handler.ServerCertificateCustomValidationCallback =
(
    message,
    cert,
    chain,
    errors
) =>
{
    return true;
};

var client = new HttpClient(handler);
```

---

## DI 註冊方式

```csharp
builder.Services.AddSingleton<HttpClient>(sp =>
{
    var handler = new HttpClientHandler();

    handler.ServerCertificateCustomValidationCallback =
        (msg, cert, chain, err) => true;

    return new HttpClient(handler);
});
```

---

## 風險

此寫法等於：

```text
不驗證任何憑證
```

包括：

```text
✔ 偽造憑證

✔ 過期憑證

✔ 不可信任憑證

✔ MITM 攻擊
```

---

## 正式環境禁止

```text
❌ Production 禁止

❌ 上架禁止

❌ 金融系統禁止
```

---

# Certificate Pinning

## 概念

只信任指定憑證。

---

### 驗證 Thumbprint

```csharp
handler.ServerCertificateCustomValidationCallback =
(
    message,
    cert,
    chain,
    errors
) =>
{
    return cert.Thumbprint ==
           "AABBCCDDEEFF112233";
};
```

---

## 流程

```text
Client
    ↓
取得 Server Certificate
    ↓
比對 Thumbprint
    ↓
一致才允許連線
```

---

## 使用場景

```text
銀行系統

支付系統

醫療系統

企業內網系統
```

---

# Android SSL 設定

Android 7+ 開始有 Network Security Config。

---

## 建立

```text
Platforms/Android/Resources/xml
```

建立：

```text
network_security_config.xml
```

---

## 設定

```xml
<?xml version="1.0" encoding="utf-8" ?>

<network-security-config>

    <base-config
        cleartextTrafficPermitted="true">

        <trust-anchors>

            <certificates src="system"/>

        </trust-anchors>

    </base-config>

</network-security-config>
```

---

## AndroidManifest.xml

```xml
<application
android:networkSecurityConfig="@xml/network_security_config">
```

---

# Localhost HTTPS 開發

.NET 提供開發憑證。

---

## 檢查

```bash
dotnet dev-certs https --check
```

---

## 建立

```bash
dotnet dev-certs https --trust
```

---

## 清除

```bash
dotnet dev-certs https --clean
```

---

## 重建

```bash
dotnet dev-certs https --clean

dotnet dev-certs https --trust
```

---

# Community Toolkit

## 官方文件

https://learn.microsoft.com/dotnet/communitytoolkit/maui/

---

## GitHub

https://github.com/CommunityToolkit/Maui

---

## 安裝

```bash
dotnet add package CommunityToolkit.Maui
```

---

## MauiProgram.cs

```csharp
using CommunityToolkit.Maui;

builder
    .UseMauiApp<App>()
    .UseMauiCommunityToolkit();
```

---

# 常用 Community Toolkit 元件

## Popup

### 顯示彈窗

```csharp
await this.ShowPopupAsync(
    new LoadingPopup()
);
```

---

## Toast

### 顯示提示訊息

```csharp
await Toast.Make(
    "登入成功"
).Show();
```

---

## Snackbar

### 底部通知

```csharp
await Snackbar.Make(
    "資料已儲存"
).Show();
```

---

## MediaElement

### 播放影片

```xml
<toolkit:MediaElement
    Source="video.mp4"
    AutoPlay="True"
    ShowsPlaybackControls="True" />
```

---

## DrawingView

### 手寫簽名

```xml
<toolkit:DrawingView />
```

---

## Expander

### 展開收合

```xml
<toolkit:Expander>

    <toolkit:Expander.Header>
        <Label Text="更多資訊" />
    </toolkit:Expander.Header>

    <Label Text="內容..." />

</toolkit:Expander>

```

---

## EventToCommandBehavior

MVVM 常用。

```xml
<Entry>

    <Entry.Behaviors>

        <toolkit:EventToCommandBehavior
            EventName="Completed"
            Command="{Binding SaveCommand}" />

    </Entry.Behaviors>

</Entry>
```

---

# 面試重點整理

## HTTPS SSL 憑證

### 正式環境

```text
使用合法 CA

匯入完整 Certificate Chain

安裝 Server Certificate
```

---

### 測試環境

```text
使用 Self-Signed Certificate

Client 安裝 Root CA
```

---

### 開發環境

```text
ServerCertificateCustomValidationCallback

略過 SSL 驗證
```

---

### 高安全需求

```text
Certificate Pinning

Thumbprint 驗證
```

---

# Community Toolkit 是什麼？

.NET MAUI 官方 UI 擴充套件。

提供：

```text
Popup

Toast

Snackbar

MediaElement

DrawingView

Expander

Behavior

Converter
```

目的：

```text
減少自行開發 UI 元件

提升開發效率

符合 MVVM 架構
```

---

# 一句話記憶法

```text
SSL

正式環境：
CA 憑證

測試環境：
Self-Signed + 匯入 Root CA

開發環境：
可暫時略過驗證


Community Toolkit

官方 UI 工具包

Popup + Toast + Snackbar + MediaElement
```

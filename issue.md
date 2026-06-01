# ISSUE

1. HTTPS SSL 憑證，怎麼迴避這個問題 ? OR 驗證

2. ssl 發行、server & crt 匯入 or 迴避

3. Community Toolkit 套件影片 or 網站

# MAUI HTTPS / SSL 憑證處理指南

## 目的

本文件說明 .NET MAUI App 與 API Server 之間 HTTPS 通訊的 SSL 憑證處理方式。

常見有以下三種方案：

1. 正式 SSL 憑證（推薦）
2. 匯入 Server 憑證
3. 開發階段略過 SSL 驗證

---

# 架構說明

```text
┌─────────────────┐
│   MAUI APP      │
└────────┬────────┘
         │ HTTPS
         ▼
┌─────────────────┐
│ ASP.NET Core API│
│ IIS / Kestrel   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ SSL Certificate │
│ PFX / CRT       │
└─────────────────┘
```

---

# 方案一：正式 SSL 憑證（推薦）

## 適用情境

* 正式環境（Production）
* 客戶驗收環境（UAT）
* 雲端部署環境

## 憑證來源

可向以下 CA（Certificate Authority）申請：

* DigiCert
* GlobalSign
* GoDaddy
* Let's Encrypt
* 公司內部 CA

範例：

```text
https://api.company.com
```

避免使用：

```text
https://localhost
https://192.168.1.100
```

原因：

SSL 驗證時會檢查：

* 憑證是否過期
* 憑證是否可信任
* 網域是否一致
* 憑證鏈是否完整

---

# 方案二：Server 匯入 SSL 憑證

## 建立開發憑證

執行：

```bash
dotnet dev-certs https --trust
```

匯出 PFX：

```bash
dotnet dev-certs https -ep ./cert/server.pfx -p Password123
```

---

## ASP.NET Core 設定

appsettings.json

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://0.0.0.0:5001",
        "Certificate": {
          "Path": "cert/server.pfx",
          "Password": "Password123"
        }
      }
    }
  }
}
```

---

## IIS 匯入憑證

步驟：

```text
IIS Manager
    ↓
Server Certificates
    ↓
Import
    ↓
選擇 PFX
    ↓
網站 Binding
    ↓
指定 HTTPS 憑證
```

---

# 方案三：安裝 CRT 根憑證

## 適用情境

* MES 系統
* 工廠內網
* 企業內部 APP
* 測試環境

將：

```text
RootCA.crt
```

安裝至：

### Windows

```text
受信任的根憑證授權單位
```

### Android

```text
設定
→ 安全性
→ 安裝憑證
```

### iOS

```text
設定
→ 一般
→ VPN與裝置管理
→ 信任憑證
```

完成後：

```text
MAUI APP
     ↓
HTTPS API
     ↓
SSL 驗證成功
```

不需修改程式碼。

---

# 方案四：略過 SSL 驗證（僅限開發）

⚠️ 僅限開發環境使用

禁止部署至正式環境。

---

## HttpClient 寫法

```csharp
#if DEBUG

var handler = new HttpClientHandler
{
    ServerCertificateCustomValidationCallback =
        (message, cert, chain, errors) => true
};

var client = new HttpClient(handler);

#else

var client = new HttpClient();

#endif
```

---

## DI 注入寫法

MauiProgram.cs

```csharp
builder.Services.AddSingleton<HttpClient>(serviceProvider =>
{
#if DEBUG

    var handler = new HttpClientHandler
    {
        ServerCertificateCustomValidationCallback =
            (message, cert, chain, errors) => true
    };

    return new HttpClient(handler);

#else

    return new HttpClient();

#endif
});
```

---

# 方案五：Certificate Pinning（高安全性）

## 適用情境

* 銀行 APP
* 醫療系統
* 政府專案
* 金流系統

範例：

```csharp
var handler = new HttpClientHandler();

handler.ServerCertificateCustomValidationCallback =
(request, cert, chain, errors) =>
{
    var thumbprint = cert?.GetCertHashString();

    return thumbprint ==
           "ABCD1234567890ABCDEF1234567890";
};
```

優點：

* 防止中間人攻擊（MITM）
* 防止假憑證
* 防止 DNS 劫持

---

# 建議使用方式

## 開發環境

```text
✔ 略過 SSL 驗證
或
✔ 安裝開發憑證
```

---

## 測試環境（QA/UAT）

```text
✔ 公司 Root CA
✔ 安裝 CRT
```

---

## 正式環境（Production）

```text
✔ 合法 CA 憑證
✔ HTTPS Only
✔ Certificate Pinning（選用）
✘ 禁止略過 SSL 驗證
```

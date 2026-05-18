# bjtu-health

## 1. 伪装微信
建议用 Playwright 模拟手机，并把 `user_agent` 强行改成微信的标识。（运行中似乎有屏幕尺寸、设备的记录，待确认）

```python
# Playwright 伪装配置
iphone = self.playwright.devices["iPhone 12 Pro"].copy()
iphone["user_agent"] = (
    "Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) "
    "AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 "
    "MicroMessenger/8.0.42(0x18002a2c) NetType/WIFI Language/zh_CN"
)
self.context = await self.browser.new_context(**iphone)
```

## 2. 账号鉴权 (获取 Cookie)

加密数据里没有身份信息，**鉴权全靠 Cookie**。

* **做法**：用抓包工具（Charles/Fiddler）手动登录抓取 Cookie，塞进代码里直接用。

<img width="3484" height="2042" alt="Image" src="https://github.com/user-attachments/assets/c30fb0f8-74d9-4647-986b-ba01c0ae4631" />

## 3. 核心加解密 (AES-ECB)

所有请求都走 `/gateway` 这个网关，请求和响应都是 AES 加密。
* **网关地址**: https://weixin.ngarihealth.com/weixin/wx/mp/wx22de84315ab03575/gateway
* **加密算法**: AES-128-ECB
* **固定密钥 (Key)**: `vss7db748e839799` （不确定是不是固定的，搜索entry.js中 `Y.aesKey = ee`打上断点，看`ee`的值）

<img width="2274" height="1194" alt="Image" src="https://github.com/user-attachments/assets/65aab605-0c12-4d46-bc2b-d415343245ec" />

**Python 后端解密代码：**

```python
import base64
from Crypto.Cipher import AES

aes_key = "vss7db748e839799"

# 1. 如果密文里有被 URL 编码搞错的空格，先还原成 '+'
clean_text = ciphertext.strip().replace(" ", "+")
raw_bytes = base64.b64decode(clean_text)

# 2. ECB 模式解密
cipher = AES.new(aes_key.encode("utf-8"), AES.MODE_ECB)
decrypted_bytes = cipher.decrypt(raw_bytes)

# 3. 去除 padding 并转字符串 (通常用到 unpad)
# print(decrypted_bytes.decode("utf-8"))

```

## 4. 前端直接看明文 (Hook 脚本)

如果想在浏览器边点边看明文，把这段代码贴到控制台。它会自动拦截所有请求并解密打印：

```javascript
(async () => {
  const TARGET_URL = "[https://weixin.ngarihealth.com/weixin/wx/mp/wx22de84315ab03575/gateway](https://weixin.ngarihealth.com/weixin/wx/mp/wx22de84315ab03575/gateway)";
  const AES_KEY = "vss7db748e839799";
  const logs = [];

  if (!window.CryptoJS) {
    await new Promise((res, rej) => {
      const s = document.createElement("script");
      s.src = "[https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.2.0/crypto-js.min.js](https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.2.0/crypto-js.min.js)";
      s.onload = res; s.onerror = rej;
      document.head.appendChild(s);
    });
  }

  function decryptRaw(text) {
    if (!text) return null;
    try {
      const s = text.trim().replace(/\s+/g, "+");
      const key = CryptoJS.enc.Utf8.parse(AES_KEY);
      const decrypted = CryptoJS.AES.decrypt(s, key, { mode: CryptoJS.mode.ECB, padding: CryptoJS.pad.Pkcs7 });
      return CryptoJS.enc.Utf8.stringify(decrypted) || null;
    } catch { return null; }
  }

  function attemptDecrypt(payload) {
    if (!payload) return null;
    const text = String(payload);
    let res = decryptRaw(text);
    if (res) return res;
    if (text.includes("=")) {
      const match = text.match(/=([^&]+)/);
      if (match) return decryptRaw(decodeURIComponent(match[1]));
    }
    return null; 
  }

  function outputLog(label, raw) {
    const plain = attemptDecrypt(raw);
    if (!plain) return;
    logs.push({ time: new Date().toLocaleTimeString(), label, data: plain });
    console.groupCollapsed(`%c${label}`, "color:#22c55e;font-weight:bold;");
    console.log(plain);
    console.groupEnd();
  }

  const originFetch = window.fetch;
  window.fetch = async function (input, init = {}) {
    const url = typeof input === "string" ? input : (input?.url || "");
    const method = (init.method || input?.method || "GET").toUpperCase();
    if (url.includes(TARGET_URL) && method === "POST") outputLog("[Fetch 请求]", init.body);
    const res = await originFetch.apply(this, arguments);
    if (url.includes(TARGET_URL) && method === "POST") {
      res.clone().text().then(t => outputLog("[Fetch 响应]", t)).catch(()=>{});
    }
    return res;
  };

  const originOpen = XMLHttpRequest.prototype.open;
  const originSend = XMLHttpRequest.prototype.send;
  XMLHttpRequest.prototype.open = function (m, u) { this._m = m; this._u = u; return originOpen.apply(this, arguments); };
  XMLHttpRequest.prototype.send = function (body) {
    if ((this._u || "").includes(TARGET_URL) && (this._m || "GET").toUpperCase() === "POST") {
      outputLog("[XHR 请求]", body);
      this.addEventListener("load", () => outputLog("[XHR 响应]", this.responseText));
    }
    return originSend.apply(this, arguments);
  };

  window.printAllGatewayLogs = () => logs;
  console.log("%c[拦截就绪] 输入 window.printAllGatewayLogs() 查看全部", "color:#38bdf8;font-weight:bold;");
})();

```

## 5. 后续的工作
* 考虑继续使用 Playwright 自动化 还是 纯逆向。
* 实测发现存在风控，会记录用户完整选号路径，不知是否存在时序校验。考虑是否需要完整模拟。

<img width="1724" height="1792" alt="Image" src="https://github.com/user-attachments/assets/d43218b3-efa2-4515-bd08-b75935785a76" />

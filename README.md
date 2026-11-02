<div align="center">

<a href="https://www.tiam.codes/?utm_source=github&utm_medium=readme&utm_campaign=logo">
  <img src="https://www.tiam.codes/favicon.svg" alt="SolveHub" width="88" height="88">
</a>

# SolveHub — CAPTCHA Solver Api

**One REST endpoint for Cloudflare Turnstile, DataDome, Akamai, PerimeterX, Imperva, AWS WAF, FunCaptcha, Alibaba Cloud CAPTCHA and image OCR.**
Pay only for solved tasks.

[![Docs](https://img.shields.io/badge/docs-tiam.codes-2563eb?style=flat-square)][docs]
[![Pricing](https://img.shields.io/badge/from-%240.20%20%2F%201k%20solves-16a34a?style=flat-square)][prices]
[![API](https://img.shields.io/badge/API-REST%20%2B%20JSON-0f172a?style=flat-square)][api]
[![Languages](https://img.shields.io/badge/console-EN%20%7C%20中文%20%7C%20日本語-7c3aed?style=flat-square)][home]

[**Website**][home] · [**Quick start**][start] · [**API reference**][api] · [**Live prices**][prices] · [**Create account**][register]

English · [简体中文](#简体中文) · [日本語](#日本語)

</div>

---

## Why SolveHub

- **One API, every challenge** — the same `POST /v1/tasks` call works for tokens (Turnstile, FunCaptcha), cookie-based anti-bot (DataDome, Akamai, PerimeterX, Imperva, AWS WAF) and image text.
- **Pay per success** — each task's price is frozen at creation and charged only when it completes. Failed or cancelled tasks are refunded automatically.
- **Safe retries** — every create request takes an `Idempotency-Key`, so a network timeout never turns into a duplicate charge.
- **Transparent pricing** — `GET /v1/prices` is public and always current. No login needed to check what a solve costs.
- **Webhooks or polling** — get signed (HMAC-SHA256) callbacks when a task finishes, or poll every few seconds.
- **Real console** — API keys with scopes and expiry, usage & spend reports, ledger, support tickets, TOTP 2FA. Available in English, 中文 and 日本語.
- **No SDK required** — plain HTTPS + JSON. Works from Python, Node.js, Go, Java, PHP, cURL — anything with an HTTP client.

## Supported CAPTCHA types

| Type | `type` value | You get back | Price / 1,000* | Guide |
| --- | --- | --- | ---: | --- |
| Cloudflare Turnstile | `turnstile` | token | $0.20 | [docs][g-turnstile] |
| Image text CAPTCHA (OCR) | `ocr` | recognized text | $0.20 | [docs][g-ocr] |
| Cloudflare Interstitial (challenge page) | `interstitial` | `cf_clearance` cookies + UA | $0.40 | [docs][g-interstitial] |
| Alibaba Cloud CAPTCHA 2.0 (阿里云验证码) | `aliyun` | verify params | $0.40 | [docs][g-aliyun] |
| Akamai Bot Manager | `akamai` | `_abck` / `bm_sz` cookies + UA | $0.80 | [docs][g-akamai] |
| PerimeterX / HUMAN | `perimeterx` | `_px3` / `_pxvid` cookies + UA | $1.00 | [docs][g-perimeterx] |
| AWS WAF CAPTCHA | `awswaf` | `aws-waf-token` | $1.00 | [docs][g-awswaf] |
| DataDome | `datadome` | `datadome` cookie + UA | $1.20 | [docs][g-datadome] |
| Imperva / Incapsula | `imperva` | Reese84 / UT cookies + UA | $1.60 | [docs][g-imperva] |
| Arkose Labs FunCaptcha | `funcaptcha` | Arkose token | $2.00 | [docs][g-funcaptcha] |

<sub>* Public list prices as of October 2026, in USD per 1,000 successful tasks. Live prices and availability: [`GET /v1/prices`][prices]. Volume pricing is available on request.</sub>

## Quick start

**1. Get an API key** — [create an account][register], then bind the API key you receive from us and top up your balance.

**2. Create a task**

```bash
curl -X POST https://www.tiam.codes/v1/tasks \
  -H "Authorization: Bearer $SOLVEHUB_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: my-order-1234-turnstile" \
  -d '{
        "type": "turnstile",
        "task": {
          "websiteURL": "https://example.com/login",
          "websiteKey": "0x4AAAAAAA_SITE_KEY"
        }
      }'
```

```json
{"taskId":"4f6c…","type":"turnstile","status":"queued","priceMicros":200,"traceId":"…","createdAt":"…"}
```

**3. Poll for the result** (every 2–3 s)

```bash
curl https://www.tiam.codes/v1/tasks/$TASK_ID \
  -H "Authorization: Bearer $SOLVEHUB_API_KEY"
```

```json
{"taskId":"4f6c…","status":"completed","solution":{"value":"0.Ab1Cd2…"},"priceMicros":200,"finishedAt":"…"}
```

That's it. Each type's guide lists its exact request fields and `solution` format.

<details>
<summary><b>Python</b></summary>

```python
import os, time, uuid, requests

BASE = "https://www.tiam.codes"
HEADERS = {"Authorization": f"Bearer {os.environ['SOLVEHUB_API_KEY']}"}

def solve(captcha_type: str, task: dict, timeout: int = 300) -> dict:
    # Reuse the same Idempotency-Key if you retry this exact request.
    idem = {"Idempotency-Key": f"{captcha_type}-{uuid.uuid4()}"}
    r = requests.post(f"{BASE}/v1/tasks", headers=HEADERS | idem,
                      json={"type": captcha_type, "task": task}, timeout=20)
    r.raise_for_status()
    task_id = r.json()["taskId"]

    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        time.sleep(3)
        t = requests.get(f"{BASE}/v1/tasks/{task_id}", headers=HEADERS, timeout=20).json()
        if t["status"] == "completed":
            return t["solution"]
        if t["status"] in ("failed", "cancelled"):
            raise RuntimeError(t.get("errorCode", t["status"]))
        # queued / dispatching / processing / uncertain -> keep polling the same task
    raise TimeoutError(task_id)

token = solve("turnstile", {
    "websiteURL": "https://example.com/login",
    "websiteKey": "0x4AAAAAAA_SITE_KEY",
})["value"]
print(token)
```

</details>

<details>
<summary><b>Node.js (18+)</b></summary>

```js
import { randomUUID } from "node:crypto";

const BASE = "https://www.tiam.codes";
const auth = { Authorization: `Bearer ${process.env.SOLVEHUB_API_KEY}` };
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function solve(type, task, timeoutMs = 300_000) {
  const res = await fetch(`${BASE}/v1/tasks`, {
    method: "POST",
    headers: { ...auth, "Content-Type": "application/json", "Idempotency-Key": `${type}-${randomUUID()}` },
    body: JSON.stringify({ type, task }),
  });
  if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);
  const { taskId } = await res.json();

  for (const end = Date.now() + timeoutMs; Date.now() < end; ) {
    await sleep(3000);
    const t = await (await fetch(`${BASE}/v1/tasks/${taskId}`, { headers: auth })).json();
    if (t.status === "completed") return t.solution;
    if (t.status === "failed" || t.status === "cancelled") throw new Error(t.errorCode ?? t.status);
  }
  throw new Error(`timeout: ${taskId}`);
}

const { cookies, useragent } = await solve("datadome", {
  websiteURL: "https://example.com/product",
  captchaUrl: "https://geo.captcha-delivery.com/captcha/?initialCid=…",
  proxyType: "http", proxyAddress: "203.0.113.10", proxyPort: "8080",
});
console.log(cookies, useragent);
```

</details>

> **Tip:** for cookie-based protections (DataDome, Akamai, PerimeterX, Imperva) pass your own proxy and reuse the returned cookies **with the same IP and `useragent`**. Turnstile and OCR don't need a proxy.

## How it works

```
 your code ──POST /v1/tasks──▶ SolveHub ──▶ solver
     ▲                            │
     └──GET /v1/tasks/{id} ◀──────┘   (or a signed webhook when it's done)
```

| Status | Meaning |
| --- | --- |
| `queued` → `dispatching` → `processing` | in progress — keep polling |
| `completed` | read `solution`; the frozen price is charged |
| `failed` / `cancelled` | reserved amount refunded |
| `uncertain` | not final — keep polling the **same** `taskId`, don't create a new task |

## FAQ

<details><summary><b>Is there an SDK?</b></summary>
Not needed — it's two HTTP calls. Copy the Python or Node.js helper above. The machine-readable OpenAPI contract is linked from the <a href="https://www.tiam.codes/docs/api?utm_source=github&utm_medium=readme">API reference</a>.
</details>

<details><summary><b>How fast is a solve?</b></summary>
Most tasks finish in seconds. Each type has a soft limit of 90–180 s; past that a task may become <code>uncertain</code> — keep polling the same task ID.
</details>

<details><summary><b>How do I pay?</b></summary>
Balances are in USD and topped up with recharge keys. Contact us (below) to buy a recharge key or discuss volume pricing.
</details>

<details><summary><b>Can I get a callback instead of polling?</b></summary>
Yes. Register an HTTPS endpoint with <code>PUT /v1/webhook</code>; every event is signed with <code>HMAC-SHA256</code> and can be retried from the console.
</details>

<details><summary><b>Is there a concurrency limit?</b></summary>
Yes, per API key. Contact us if you need a higher limit.
</details>

## Contact

- 🌐 Website: [www.tiam.codes][home]
- 📧 Email: `529828069@qq.com`
- 💬 Telegram: `seencx`

---

## 简体中文

**SolveHub 是一个统一的验证码识别 API**：一个 REST 接口处理 Cloudflare Turnstile / 5 秒盾、DataDome、Akamai、PerimeterX、Imperva、AWS WAF、阿里云验证码 2.0、FunCaptcha 和图片验证码。

- 按**成功任务**计费，USD 计价；失败、取消自动退还预占金额
- `Idempotency-Key` 幂等，网络超时重试不会重复扣费
- 公开实时价格接口 `GET /v1/prices`，Turnstile 低至 **$0.20 / 千次**
- 支持 Webhook 回调（HMAC 签名）或轮询
- 中文控制台：API Key 管理、用量统计、账单流水、工单、两步验证

👉 [中文文档](https://www.tiam.codes/zh/docs/?utm_source=github&utm_medium=readme&utm_campaign=zh) · [注册账户][register] · [实时价格](https://www.tiam.codes/zh/docs/prices?utm_source=github&utm_medium=readme&utm_campaign=zh)

## 日本語

**SolveHub は CAPTCHA 解決 API です。** Cloudflare Turnstile、DataDome、Akamai、PerimeterX、Imperva、AWS WAF、FunCaptcha、画像 CAPTCHA に 1 つの REST エンドポイントで対応。成功したタスクのみ課金、コンソールは日本語対応。

👉 [日本語ドキュメント](https://www.tiam.codes/ja/docs/?utm_source=github&utm_medium=readme&utm_campaign=ja) · [アカウント作成][register]

---

<sub>Use SolveHub only on websites and flows you are authorized to access. All site keys, URLs, proxies and tokens above are placeholders.</sub>

<!-- Links (UTM-tagged so GitHub traffic shows up in GA4) -->
[home]: https://www.tiam.codes/?utm_source=github&utm_medium=readme&utm_campaign=readme
[docs]: https://www.tiam.codes/docs/?utm_source=github&utm_medium=readme&utm_campaign=readme
[start]: https://www.tiam.codes/docs/start?utm_source=github&utm_medium=readme&utm_campaign=readme
[api]: https://www.tiam.codes/docs/api?utm_source=github&utm_medium=readme&utm_campaign=readme
[prices]: https://www.tiam.codes/docs/prices?utm_source=github&utm_medium=readme&utm_campaign=readme
[register]: https://www.tiam.codes/register?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-turnstile]: https://www.tiam.codes/docs/turnstile?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-ocr]: https://www.tiam.codes/docs/ocr?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-interstitial]: https://www.tiam.codes/docs/interstitial?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-aliyun]: https://www.tiam.codes/docs/aliyun?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-akamai]: https://www.tiam.codes/docs/akamai?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-perimeterx]: https://www.tiam.codes/docs/perimeterx?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-awswaf]: https://www.tiam.codes/docs/awswaf?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-datadome]: https://www.tiam.codes/docs/datadome?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-imperva]: https://www.tiam.codes/docs/imperva?utm_source=github&utm_medium=readme&utm_campaign=readme
[g-funcaptcha]: https://www.tiam.codes/docs/funcaptcha?utm_source=github&utm_medium=readme&utm_campaign=readme

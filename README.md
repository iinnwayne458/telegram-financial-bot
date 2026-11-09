# Telegram Expense Bot – Hướng dẫn / Guide

Bot ghi chi tiêu cá nhân qua Telegram, lưu vào Google Sheet. / A personal expense bot on Telegram that stores data in Google Sheets.

**Cách hoạt động / How it works:** Telegram → Cloudflare Worker → Google Apps Script → Google Sheet (tab `Năm nay`)

- Worker trả `200 OK` ngay cho Telegram (tránh lỗi 302 của Apps Script). / The Worker answers Telegram with `200 OK` immediately (avoids Apps Script’s 302 error).
- Apps Script xử lý tin nhắn và ghi vào Sheet. / Apps Script parses the message and writes to the Sheet.

## Cú pháp nhắn tin / Message syntax

| Gõ / Type | Ý nghĩa / Meaning |
| --- | --- |
| `Cà phê:food 35k` | Chi 35.000đ, danh mục food / Expense 35,000, category food |
| `Cà phê 35k` | Chi, danh mục “khác” / Expense, category “khác” |
| `+Lương:thu 15tr` | Khoản thu 15.000.000đ / Income 15,000,000 |
| `/thang` | Tổng kết tháng này / This month’s summary |
| `/xoa` | Xóa dòng vừa nhập / Delete the last entry |

Đơn vị / Units: `k` = nghìn / thousand, `tr` hoặc `m` = triệu / million.

## Mã nguồn Apps Script / Apps Script source code

Dùng chung cho cả hai bản hướng dẫn bên dưới. Sửa 3 dòng đầu. / Shared by both guides below. Edit the first 3 constants.

```jsx
const TOKEN = 'YOUR_BOT_TOKEN';
const ALLOWED_ID = 123456789;          // your Telegram chat id
const SECRET = 'YOUR_RANDOM_SECRET';   // long random string, e.g. k8Xp2mQ9vL4zR7tY
const SHEET_NAME = 'Năm nay';
const TZ = 'Asia/Ho_Chi_Minh';

function doPost(e) {
  if (e.parameter.key !== SECRET) return ContentService.createTextOutput('ok');
  const lock = LockService.getScriptLock();
  lock.waitLock(10000);
  try {
    const update = JSON.parse(e.postData.contents);
    const cache = CacheService.getScriptCache();
    const key = 'u' + update.update_id;
    if (!cache.get(key)) {          // ignore duplicates
      cache.put(key, '1', 3600);
      const msg = update.message;
      if (msg && msg.text && msg.chat.id === ALLOWED_ID) {
        send(msg.chat.id, handle(msg.text.trim()));
      }
    }
  } finally {
    lock.releaseLock();
  }
  return ContentService.createTextOutput('ok');
}

function handle(text) {
  const sh = SpreadsheetApp.getActive().getSheetByName(SHEET_NAME);

  if (text === '/start') return 'Xin chào! Gõ ví dụ:\nCà phê:food 35k\n+Lương:thu 15tr\n/thang  /xoa';
  if (text === '/thang') return summary(sh);
  if (text === '/xoa') {
    if (sh.getLastRow() < 2) return 'Chưa có dữ liệu.';
    const r = sh.getRange(sh.getLastRow(), 1, 1, 5).getValues()[0];
    sh.deleteRow(sh.getLastRow());
    return `🗑 Đã xóa:${r[1]}${fmt(r[3])}`;
  }

  // "Cà phê:food 35k" | "Cà phê 35k" | "+Lương:thu 15tr"
  const m = text.match(/^(\+)?\s*(.+?)(?:\s*:\s*(\S.*?))?\s+([\d.,]+)\s*(k|tr|m)?$/i);
  if (!m) return 'Sai cú pháp. Ví dụ:\nCà phê:food 35k\n+Lương:thu 15tr\n/thang  /xoa';

  const isIncome = !!m[1];
  const name = m[2];
  const cat = m[3] || 'khác';
  const amount = toNumber(m[4], m[5]);

  sh.appendRow([new Date(), name, cat, amount, isIncome ? 'thu' : 'chi']);
  return `${isIncome ? '💰 Thu' : '✅ Chi'}:${name} [${cat}]${fmt(amount)}`;
}

function toNumber(s, unit) {
  unit = (unit || '').toLowerCase();
  if (unit === 'k') return Math.round(parseFloat(s.replace(',', '.')) * 1000);
  if (unit === 'tr' || unit === 'm') return Math.round(parseFloat(s.replace(',', '.')) * 1000000);
  return parseInt(s.replace(/[.,]/g, ''), 10);
}

function fmt(n) { return Number(n).toLocaleString('vi-VN') + 'đ'; }

function summary(sh) {
  const ym = Utilities.formatDate(new Date(), TZ, 'yyyy-MM');
  const rows = sh.getDataRange().getValues().slice(1);
  let thu = 0, chi = 0, byCat = {};
  rows.forEach(r => {
    if (Utilities.formatDate(new Date(r[0]), TZ, 'yyyy-MM') !== ym) return;
    if (r[4] === 'thu') thu += r[3];
    else { chi += r[3]; byCat[r[2]] = (byCat[r[2]] || 0) + r[3]; }
  });
  const lines = Object.entries(byCat).sort((a, b) => b[1] - a[1])
    .map(([c, v]) => `•${c}:${fmt(v)}`).join('\n');
  return `📊 Tháng${ym}\nThu:${fmt(thu)}\nChi:${fmt(chi)}\nCòn:${fmt(thu - chi)}\n\n${lines}`;
}

function send(chatId, text) {
  UrlFetchApp.fetch(`https://api.telegram.org/bot${TOKEN}/sendMessage`, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify({ chat_id: chatId, text })
  });
}
```

## Cloudflare Worker code

```jsx
export default {
  async fetch(request, env, ctx) {
    if (request.method !== 'POST') return new Response('ok');
    const body = await request.text();
    ctx.waitUntil(fetch(env.GAS_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body,
      redirect: 'follow'
    }));
    return new Response('ok');
  }
}
```

---

## 🇻🇳 Hướng dẫn tiếng Việt

### Bước 1: Tạo bot Telegram

1. Chat với **@BotFather**, gõ `/newbot`, đặt tên và username cho bot.
2. Copy **token** BotFather gửi (dạng `123456:ABC...`).
3. Chat với **@userinfobot** để lấy **chat id** của bạn (một dãy số).

### Bước 2: Tạo Google Sheet

1. Tạo Sheet mới, đổi tên tab đầu tiên thành **Năm nay**.
2. Dòng 1 điền tiêu đề: `Thời gian | Nội dung | Danh mục | Số tiền | Loại` (cột A đến E).
3. File → Settings → đặt Time zone là **Asia/Ho_Chi_Minh**.
4. Định dạng cột (làm một lần): cột A dùng Custom date and time `dd/MM/yyyy HH:mm`, cột D dùng Custom number format `#,##0`.

### Bước 3: Dán code Apps Script

1. Trong Sheet: Extensions → Apps Script, xóa code mẫu, dán **mã nguồn Apps Script** ở trên.
2. Sửa `TOKEN`, `ALLOWED_ID`, `SECRET` (chuỗi ngẫu nhiên dài bạn tự chọn). Lưu lại (Ctrl+S).

### Bước 4: Deploy Web app

1. Deploy → New deployment → chọn loại **Web app**.
2. Execute as: **Me**. Who has access: **Anyone**.
3. Deploy, cấp quyền khi Google hỏi (nếu hiện cảnh báo “chưa xác minh” thì Advanced → Go to … → Allow).
4. Copy **Web app URL** (kết thúc bằng `/exec`) và thêm đuôi `?key=SECRET_CUA_BAN`, ví dụ:

```
https://script.google.com/macros/s/XXXX/exec?key=k8Xp2mQ9vL4zR7tY
```

### Bước 5: Tạo Cloudflare Worker

1. Vào dash.cloudflare.com, đăng ký miễn phí.
2. Workers & Pages → Create → Create Worker, đặt tên (ví dụ `tele-bot`), Deploy.
3. Edit code, dán **Cloudflare Worker code** ở trên, Deploy.
4. Settings → Variables and Secrets → Add: tên `GAS_URL`, giá trị là link ở bước 4 (đã có `?key=...`). Lưu và Deploy lại.
5. Copy link Worker (dạng `https://tele-bot.<ten-ban>.workers.dev`).

### Bước 6: Gắn webhook Telegram

Mở trên trình duyệt (thay token và link Worker):

```
https://api.telegram.org/bot<TOKEN>/setWebhook?url=<LINK_WORKER>&drop_pending_updates=true
```

Thấy `"ok": true` là xong. Kiểm tra bằng `https://api.telegram.org/bot<TOKEN>/getWebhookInfo`: `pending_update_count` phải là 0 và không có `last_error_message` mới.

### Bước 7: Dùng thử

Nhắn `Cà phê:food 35k`, bot trả lời `✅ Chi: Cà phê [food] 35.000đ` và tab `Năm nay` có thêm dòng mới.

### Khi sang năm mới

1. Chuột phải tab `Năm nay` → **Duplicate**, đổi tên bản sao thành tên năm cũ (ví dụ `2026`).
2. Ở tab `Năm nay`, chọn từ dòng 2 trở xuống → **Delete rows**. Giữ dòng tiêu đề.

Không xóa hay đổi tên tab `Năm nay`, và không chèn cột vào giữa A đến E. Muốn thêm cột thì thêm từ cột F trở đi.

### Lưu ý và xử lý sự cố

- Mỗi lần sửa code Apps Script phải Deploy → Manage deployments → bút chì → **New version** → Deploy. Link `/exec` giữ nguyên.
- Bot im lặng: xem `getWebhookInfo` (dòng `last_error_message`), rồi Apps Script → **Executions**. Kiểm tra `ALLOWED_ID` đúng chat id của bạn và `?key=` trong `GAS_URL` khớp `SECRET`.
- Lỗi `302 Found`: webhook đang trỏ thẳng vào Apps Script, phải trỏ vào link Worker.
- Tin đầu tiên sau một lúc không dùng có thể trả lời chậm vài giây (Apps Script khởi động nguội).
- Thông báo “Update your Wrangler configuration” của Cloudflare có thể bỏ qua nếu bạn quản lý Worker trên dashboard.

### An toàn dữ liệu

- Không chia sẻ file Sheet, file Apps Script, token bot hay link có `?key=`. Nếu lộ token, nhắn @BotFather `/revoke`.
- Chat với bot không mã hóa đầu cuối, đừng nhắn số tài khoản hay mật khẩu.
- Thỉnh thoảng tải file về (File → Download → .xlsx) để sao lưu. Sheets có sẵn Version history.

### Giới hạn (gói miễn phí, số liệu tham khảo, có thể thay đổi)

Worker khoảng 100.000 request/ngày, Apps Script khoảng 20.000 lệnh `UrlFetch`/ngày và 6 phút mỗi lần chạy. Dùng cá nhân gần như không bao giờ chạm giới hạn.

---

## 🇬🇧 English guide

### Step 1: Create the Telegram bot

1. Chat with **@BotFather**, send `/newbot`, choose a name and username.
2. Copy the **token** BotFather gives you (looks like `123456:ABC...`).
3. Chat with **@userinfobot** to get your **chat id** (a number).

### Step 2: Create the Google Sheet

1. Create a new Sheet and rename the first tab to **Năm nay**.
2. Put headers in row 1: `Thời gian | Nội dung | Danh mục | Số tiền | Loại` (columns A to E).
3. File → Settings → set Time zone to **Asia/Ho_Chi_Minh**.
4. Format columns once: column A with Custom date and time `dd/MM/yyyy HH:mm`, column D with Custom number format `#,##0`.

### Step 3: Paste the Apps Script code

1. In the Sheet: Extensions → Apps Script, delete the sample code, paste the **Apps Script source code** above.
2. Set `TOKEN`, `ALLOWED_ID`, and `SECRET` (a long random string of your choice). Save (Ctrl+S).

### Step 4: Deploy as a Web app

1. Deploy → New deployment → type **Web app**.
2. Execute as: **Me**. Who has access: **Anyone**.
3. Deploy and grant permissions (if you see an “unverified app” warning: Advanced → Go to … → Allow).
4. Copy the **Web app URL** (ends with `/exec`) and append `?key=YOUR_SECRET`, for example:

```
https://script.google.com/macros/s/XXXX/exec?key=k8Xp2mQ9vL4zR7tY
```

### Step 5: Create the Cloudflare Worker

1. Go to dash.cloudflare.com and sign up (free).
2. Workers & Pages → Create → Create Worker, name it (e.g. `tele-bot`), Deploy.
3. Edit code, paste the **Cloudflare Worker code** above, Deploy.
4. Settings → Variables and Secrets → Add: name `GAS_URL`, value is the link from Step 4 (including `?key=...`). Save and Deploy again.
5. Copy the Worker URL (like `https://tele-bot.<your-name>.workers.dev`).

### Step 6: Set the Telegram webhook

Open in your browser (replace the token and Worker URL):

```
https://api.telegram.org/bot<TOKEN>/setWebhook?url=<WORKER_URL>&drop_pending_updates=true
```

You should see `"ok": true`. Verify with `https://api.telegram.org/bot<TOKEN>/getWebhookInfo`: `pending_update_count` should be 0 with no new `last_error_message`.

### Step 7: Try it

Send `Cà phê:food 35k`. The bot replies `✅ Chi: Cà phê [food] 35.000đ` and a new row appears in the `Năm nay` tab.

### At the start of a new year

1. Right-click the `Năm nay` tab → **Duplicate**, rename the copy to the old year (e.g. `2026`).
2. In `Năm nay`, select row 2 downward → **Delete rows**. Keep the header row.

Do not delete or rename the `Năm nay` tab, and do not insert columns between A and E. Add extra columns from column F onward.

### Notes and troubleshooting

- After every Apps Script code change: Deploy → Manage deployments → pencil → **New version** → Deploy. The `/exec` URL stays the same.
- Bot is silent: check `getWebhookInfo` (`last_error_message`), then Apps Script → **Executions**. Make sure `ALLOWED_ID` is your chat id and the `?key=` in `GAS_URL` matches `SECRET`.
- `302 Found` error: the webhook points straight at Apps Script; it must point at the Worker URL.
- The first message after a quiet period may take a few seconds (Apps Script cold start).
- Cloudflare’s “Update your Wrangler configuration” notice can be ignored if you manage the Worker on the dashboard.

### Data safety

- Do not share the Sheet, the Apps Script file, the bot token, or the URL containing `?key=`. If the token leaks, message @BotFather `/revoke`.
- Bot chats are not end-to-end encrypted; do not send account numbers or passwords.
- Download a backup now and then (File → Download → .xlsx). Sheets also keeps Version history.

### Limits (free tier, approximate and subject to change)

Worker: about 100,000 requests/day. Apps Script: about 20,000 `UrlFetch` calls/day and 6 minutes per run. Personal use will almost never reach these.

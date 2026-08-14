/**
 * POST /api/subscribe
 * يستقبل الاسم والبريد → يرسل الفصل الأول عبر Resend → يحفظ المشترك في قائمة البريد.
 *
 * متغيّرات البيئة المطلوبة:
 *   RESEND_API_KEY      مفتاح Resend                       (إلزامي)
 *   FROM_EMAIL          "د. نانيس الشيمي <hello@yourdomain.com>"  (إلزامي — نطاق موثّق في Resend)
 * اختيارية:
 *   CHAPTER_URL         رابط الفصل — يُستنتج تلقائياً كـ /chapter-1.pdf إن تُرك فارغاً
 *   RESEND_AUDIENCE_ID  معرّف قائمة المشتركين في Resend
 *   REPLY_TO            بريد للرد المباشر
 *   SHEET_WEBHOOK       رابط Google Apps Script لحفظ نسخة في Google Sheets
 *
 * يعمل كما هو على Vercel (Node runtime). لـ Netlify: انقلي الملف إلى
 * netlify/functions/subscribe.js وغلّفي الدالة بـ exports.handler.
 */

const RATE = new Map();                       // حد بسيط: 5 طلبات لكل IP في الساعة
const WINDOW_MS = 60 * 60 * 1000;
const MAX_HITS = 5;

function rateLimited(ip) {
  const now = Date.now();
  const hits = (RATE.get(ip) || []).filter(t => now - t < WINDOW_MS);
  hits.push(now);
  RATE.set(ip, hits);
  if (RATE.size > 5000) RATE.clear();          // تنظيف بدائي للذاكرة
  return hits.length > MAX_HITS;
}

const isEmail = v => /^[^@\s]+@[^@\s]+\.[^@\s]{2,}$/.test(v);
const esc = s => String(s).replace(/[&<>"]/g, c => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;" }[c]));

/* ------------------------- قالب الرسالة ------------------------- */
function chapterEmail(name, chapterUrl) {
  const safeName = esc(name);
  return `<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>الفصل الأول من كتاب حياة… بتفاصيل تانية</title></head>
<body style="margin:0;padding:0;background:#FAF7F0;">
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background:#FAF7F0;padding:28px 12px;">
 <tr><td align="center">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="max-width:560px;background:#FFFFFF;border:1px solid #E5DFD1;border-radius:22px;overflow:hidden;font-family:'Cairo','Segoe UI',Tahoma,sans-serif;color:#2C2A26;">

   <tr><td style="background:#33503F;padding:26px 30px;text-align:center;">
     <div style="color:#C6A868;font-size:12px;letter-spacing:2px;">دار نانلش</div>
     <div style="color:#FFFFFF;font-size:23px;font-weight:700;padding-top:6px;">حياة… بتفاصيل تانية</div>
     <div style="color:rgba(255,255,255,.72);font-size:13px;padding-top:4px;">د. نانيس الشيمي</div>
   </td></tr>

   <tr><td style="padding:32px 30px 8px;">
     <p style="margin:0 0 18px;font-size:17px;line-height:1.9;">أهلاً ${safeName},</p>
     <p style="margin:0 0 14px;font-size:16.5px;line-height:2;">شكراً لك على اهتمامك بالكتاب.</p>
     <p style="margin:0 0 14px;font-size:16.5px;line-height:2;">يسعدني أن أشاركك الفصل الأول.</p>
     <p style="margin:0 0 24px;font-size:16.5px;line-height:2;">أتمنى أن تجد فيه ما يلامس قلبك.</p>
   </td></tr>

   <tr><td align="center" style="padding:4px 30px 30px;">
     <a href="${chapterUrl}"
        style="display:inline-block;background:#C6A868;color:#33290F;text-decoration:none;font-weight:700;font-size:16px;padding:15px 40px;border-radius:100px;">
        تحميل الفصل الأول (PDF)
     </a>
     <p style="margin:16px 0 0;font-size:12.5px;color:#6E6A61;line-height:1.8;">
       إن لم يعمل الزر، انسخ هذا الرابط في المتصفح:<br>
       <a href="${chapterUrl}" style="color:#33503F;word-break:break-all;">${chapterUrl}</a>
     </p>
   </td></tr>

   <tr><td style="padding:0 30px 30px;">
     <div style="border-top:1px solid #E5DFD1;padding-top:22px;text-align:center;">
       <p style="margin:0;font-size:16px;line-height:2;">مع خالص المحبة</p>
       <p style="margin:2px 0 0;font-size:17px;font-weight:700;color:#33503F;">د. نانيس الشيمي</p>
     </div>
   </td></tr>

   <tr><td style="background:#F3EEE3;padding:18px 30px;text-align:center;color:#6E6A61;font-size:12px;line-height:1.8;">
     وصلتك هذه الرسالة لأنك طلبت الفصل الأول من موقع الكتاب.<br>
     © ٢٠٢٦ دار نانلش
   </td></tr>

  </table>
 </td></tr>
</table>
</body></html>`;
}

/* ------------------------- المعالج ------------------------- */
export default async function handler(req, res) {
  if (req.method === "OPTIONS") return res.status(204).end();
  if (req.method !== "POST") return res.status(405).json({ error: "Method not allowed" });

  const { RESEND_API_KEY, FROM_EMAIL, CHAPTER_URL, RESEND_AUDIENCE_ID, REPLY_TO, SHEET_WEBHOOK } = process.env;
  if (!RESEND_API_KEY || !FROM_EMAIL) {
    console.error("Missing env vars: RESEND_API_KEY / FROM_EMAIL");
    return res.status(500).json({ error: "الخدمة غير مهيّأة بعد" });
  }

  // رابط الفصل: يُستنتج تلقائياً من نطاق الموقع ما لم يُضبط CHAPTER_URL
  const proto = req.headers["x-forwarded-proto"] || "https";
  const chapterLink = CHAPTER_URL || `${proto}://${req.headers.host}/chapter-1.pdf`;

  const ip = (req.headers["x-forwarded-for"] || "").split(",")[0].trim() || "unknown";
  if (rateLimited(ip)) return res.status(429).json({ error: "محاولات كثيرة. حاولي بعد قليل." });

  let body = req.body;
  if (typeof body === "string") { try { body = JSON.parse(body); } catch { body = {}; } }
  const name = String(body?.name || "").trim().slice(0, 60);
  const email = String(body?.email || "").trim().toLowerCase().slice(0, 120);

  if (name.length < 2) return res.status(400).json({ error: "الاسم غير صالح" });
  if (!isEmail(email)) return res.status(400).json({ error: "البريد الإلكتروني غير صالح" });

  try {
    // 1) إرسال الفصل الأول
    const send = await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: { Authorization: `Bearer ${RESEND_API_KEY}`, "Content-Type": "application/json" },
      body: JSON.stringify({
        from: FROM_EMAIL,
        to: [email],
        subject: "الفصل الأول من كتاب حياة… بتفاصيل تانية",
        html: chapterEmail(name, chapterLink),
        text: `أهلاً ${name},

شكراً لك على اهتمامك بالكتاب.
يسعدني أن أشاركك الفصل الأول.
أتمنى أن تجد فيه ما يلامس قلبك.

رابط التحميل: ${chapterLink}

مع خالص المحبة
د. نانيس الشيمي`,
        ...(REPLY_TO ? { reply_to: REPLY_TO } : {}),
        tags: [{ name: "campaign", value: "free_chapter" }]
      })
    });

    if (!send.ok) {
      const detail = await send.text();
      console.error("Resend send failed:", send.status, detail);
      return res.status(502).json({ error: "تعذّر إرسال الرسالة" });
    }

    // 2) حفظ المشترك في قائمة Resend (لا يُفشل الطلب إن تعثّر)
    if (RESEND_AUDIENCE_ID) {
      fetch(`https://api.resend.com/audiences/${RESEND_AUDIENCE_ID}/contacts`, {
        method: "POST",
        headers: { Authorization: `Bearer ${RESEND_API_KEY}`, "Content-Type": "application/json" },
        body: JSON.stringify({ email, first_name: name, unsubscribed: false })
      }).catch(e => console.error("audience add failed:", e));
    }

    // 3) نسخة احتياطية في Google Sheets (اختياري)
    if (SHEET_WEBHOOK) {
      fetch(SHEET_WEBHOOK, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ name, email, source: body?.source || "landing", date: new Date().toISOString() })
      }).catch(e => console.error("sheet webhook failed:", e));
    }

    return res.status(200).json({ ok: true });
  } catch (err) {
    console.error("subscribe error:", err);
    return res.status(500).json({ error: "خطأ غير متوقع" });
  }
}

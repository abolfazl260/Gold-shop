# مدل داده و API پیشنهادی (سطح مفهومی)

> این سند **طرح اولیه** است نه migration، schema نهایی یا قرارداد API تصویب‌شده. انتخاب stack و سرویس پرداخت هنوز باز است.

## موجودیت‌های اصلی
| موجودیت | فیلدهای نمونه | قید کلیدی |
|---|---|---|
| User | id, phone, status, created_at | شماره با ذخیره‌سازی/نمایش امن |
| Role/Permission | id, name, scope | deny by default |
| Seller | id, legal_status, public_name, verified_at, state | تایید تنها با شواهد |
| SellerDocument | id, seller_id, doc_type, secure_ref, expires_at | دسترسی محدود، بدون فایل عمومی |
| SellerMember | seller_id, user_id, role_id | جداسازی tenant |
| Category | id, parent_id, attributes_version | ساختار درختی پایدار |
| Product | id, category_id, title, media | متن عمومی |
| Offer/SKU | id, seller_id, product_id, sku, metal_weight, purity, status | unique(seller_id, sku) |
| Inventory | offer_id, available, reserved, version | موجودی غیرمنفی |
| Reservation | id, offer_id, quantity, expires_at, state | جلوگیری از oversell |
| GoldRate | id, source, rate, purity, observed_at, status | دارای زمان اعتبار |
| PricingRule | id, version, seller_scope, valid_from, valid_to | عدم تغییر نسخه قبلی |
| Quote | id, user_id, offer_snapshot, breakdown, total_irr, expires_at | immutable پس از صدور |
| Order | id, user_id, seller_id, quote_id, status, snapshots | یک فروشنده در MVP |
| OrderItem | id, order_id, offer_snapshot, quantity, total | فقط داده زمان خرید |
| PaymentAttempt | id, order_id, provider_id, amount_irr, state, idempotency_key | کلید یکتا |
| PaymentEvent | id, attempt_id, provider_event_id, payload_digest | unique provider_event_id |
| Shipment | id, order_id, carrier_id, tracking_ref, state | یکپارچگی با Order |
| LedgerEntry | id, order_id, seller_id, type, amount_irr, event_ref | append-only |
| Refund | id, payment_attempt_id, amount_irr, state, provider_ref | مقدار تجمیعی ≤ پرداخت |
| Settlement | id, seller_id, amount_irr, state, provider_ref | جلوگیری از تسویه دوباره |
| SupportTicket | id, user_id, order_id, status, assigned_to | دسترسی سطح رکورد |
| AuditEvent | actor, action, entity, before_hash, after_hash, at, trace_id | append-only و قابل کنترل |
| Notification | id, recipient, template, channel, state | جلوگیری از ارسال تکراری |

## روابط
Seller 1—N Offers؛ Product 1—N Offers (در صورت کاتالوگ مشترک)؛ Offer 1—N Reservations؛ Quote متعلق به snapshot مشخص Offer + نرخ؛ Order 1—N OrderItems و **تمام آیتم‌های آن متعلق به یک seller_id**؛ Order 1—N PaymentAttempts؛ Order 1—N Shipments (برای استثناها)؛ LedgerEntry به رخداد مالی پیوند دارد.

## قیود مهم دیتابیس و سرویس
- currency برای تمام پرداخت‌ها ریال صحیح و تبدیل واحد فقط در boundary.
- check available ≥ 0، reserved ≥ 0 و جلوگیری از فروش بیشتر از ظرفیت واقعی.
- optimistic concurrency یا lock تراکنشی برای رزرو آخرین واحد.
- FK و unique constraints، transaction boundaries و outbox pattern برای پیام‌های پرداخت/اعلان حساس.
- در callback، یک provider_event_id فقط یک‌بار پردازش شود؛ audit و گزارش خطا حتی برای duplicate.
- مبلغ Order از Quote snapshot پذیرفته‌شده؛ هرگز از payload موبایل.
- موجودی و Order soft-delete/archival معنادار؛ عدم حذف رویدادهای مالی گذشته.

## API نمونه (نام مسیر قطعی نیست)
| Method | Route | شرح | دسترسی |
|---|---|---|---|
| POST | /api/v1/auth/otp/request | درخواست کد | عمومی با rate limit |
| POST | /api/v1/auth/otp/verify | تأیید ورود | عمومی با محدودیت |
| GET | /api/v1/categories | دسته‌بندی | عمومی |
| GET | /api/v1/offers | جست‌وجو و فیلتر | عمومی |
| GET | /api/v1/offers/{id} | جزئیات عرضه | عمومی |
| POST | /api/v1/quotes | محاسبه قیمت و شرایط | احراز‌شده/سیاست مهمان |
| POST | /api/v1/orders | ایجاد سفارش از Quote | خریدار |
| POST | /api/v1/payments/intents | آغاز پرداخت | مالک سفارش |
| POST | /api/v1/payments/webhook | رویداد PSP معتبر | فقط امضای معتبر + استعلام |
| GET | /api/v1/orders/{id} | وضعیت و جزئیات | مالک سفارش/نقش مجاز |
| POST | /api/v1/tickets | تیکت | کاربر احراز شده |
| GET/POST | /api/v1/seller/offers | کاتالوگ فروشنده | مالک فروشگاه/مجوزدار |
| GET | /api/v1/seller/settlements | صورت‌حساب فروشنده | عضو مالی مجاز |
| POST | /api/v1/admin/sellers/{id}/approve | تأیید فروشنده | compliance |
| POST | /api/v1/admin/rates/freeze | توقف نرخ | admin مجاز |

## استاندارد پاسخ و امنیت
- پاسخ خطای ساخت‌یافته: code، message فارسی/قابل بومی‌سازی، trace_id، retryable؛ عدم افشای stack trace.
- status code درست، pagination و validation سرور، نسخه API و schema رسمی OpenAPI در زمان پیاده‌سازی.
- idempotency_key در POSTهای پرداخت/ایجاد سفارش، request_id در لاگ؛ timestamps UTC با نمایش شمسی/محلی فقط در UI.
- object-level authorization برای **هر** seller/order/ticket؛ نه فقط نقش کلی.
- secrets، KYC و اطلاعات بانکی صرفاً در ذخیره‌سازی امن رمزگذاری‌شده با policy نگهداری.

## تصمیم‌های باز
مالکیت Product میان فروشندگان (shared vs seller-owned)، مدل ERP، روش PSP، scope دسترسی کارکنان، حفظ داده و الزامات صدور صورتحساب. تصمیم‌های باز نباید از نمونه API به تصمیم قطعی تعبیر شوند.

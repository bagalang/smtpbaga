# smtpbaga

**SMTP client** for Baga — изпращане на писма върху `std/net` (TCP + TLS 1.3).
EHLO, STARTTLS (RFC 3207), AUTH LOGIN, multipart/alternative, RFC 5322 дата,
RFC 2047 UTF-8 заглавия, dot-stuffing. Направен, за да го ползват приложенията
за едно нещо: welcome, покани и **reset на парола** — без всеки да пише свой
SMTP клиент.

Rust аналог: [`lettre`](https://crates.io/crates/lettre).

Това репо е пакетът. Компилаторът, `std` и `chronobaga` остават в
монорепото на езика Baga. Checkout-ва се като `app-product/smtpbaga` там
(git submodule), за да работят path зависимостите и `-I app-product`.

| | |
|--|--|
| **sandak** | `smtpbaga` **0.1.0** |
| **Зависимости** | `std`, `chronobaga` |
| **Тест** | `tests/smtp_test.baga` (в baga) |

## Checkout

В клонинг на baga:

```bash
git submodule update --init app-product/smtpbaga
# или първия път, ако submodule-ът не е записан:
git clone git@github.com:bagalang/smtpbaga.git app-product/smtpbaga
```

`sandak.toml` държи path deps (`../../std`, `../chronobaga`).
Тестът `tests/smtp_test.baga` остава в baga.

## Ползване

```baga
import "smtpbaga/smtp.baga"

fn main() -> i64 !Net !IO !Random !Time {
    let cfg = smtp_cfg("smtp.example.com", 587, "no-reply@example.com")
    cfg.tls = SMTP_STARTTLS()
    cfg.user = "no-reply@example.com"
    cfg.pass = env("SMTP_PASS")?

    let rcpts: Vec<SmtpAddr> = vec_new()
    vec_push(rcpts, smtp_to("ivan@example.com"))

    let msg = smtp_msg("Добре дошли", "Текст на писмото")
    let r = smtp_send(cfg, rcpts, msg)?
    if r.ok == 0 {
        eprintln(concat(r.err.stage, concat(": ", r.err.msg)))
        return 1
    }
    return 0
}
```

За най-честия случай има и едноредово:

```baga
let r = smtp_send_text(cfg, "ivan@example.com", "Тема", "Тяло")?
```

HTML + текстов вариант се прави, като се напълни `msg.html`:

```baga
let mut msg = smtp_msg("Тема", "Текстовата версия")
msg.html = "<h1>HTML версия</h1>"
```

## API накратко

| Функция | Какво прави |
|--|--|
| `smtp_cfg(host, port, from)` | конфигурация; `tls` е STARTTLS по подразбиране |
| `smtp_to/cc/bcc(email)` | получател; BCC не влиза в хедърите |
| `smtp_msg(subject, text)` | писмо; `msg.html` включва multipart |
| `smtp_send(cfg, rcpts, msg)` | пълният разговор; връща `SmtpOk` |
| `smtp_send_text(cfg, to, subj, text)` | обвивка за един получател |
| `smtp_verify(cfg)` | EHLO+STARTTLS+AUTH без писмо — за health check при старт |
| `smtp_err_transient(e)` | `true` за 4xx/локална → има смисъл retry |
| `smtp_date_rfc(d, t)` | RFC 5322 дата (напр. за `msg.headers`) |
| `smtp_dot_stuff(body)` | за тяло със собствена точка-край |

## Транспорт

`cfg.tls` е изричен, не „по подразбиране TLS“:

| Стойност | Порт | Поведение |
|--|--|--|
| `SMTP_PLAIN()` | 25 (dev) | без TLS — само за mock/MailHog |
| `SMTP_STARTTLS()` | 587 | plain → `STARTTLS` → TLS → ново EHLO |
| `SMTP_TLS()` | 465 | TLS от първия байт (implicit) |

`cfg.anchor_der` е root CA в DER. Празно `bytes` = приемай самоподписан
сертификат — **само** за dev/mock (същата конвенция като `std/net/tls_conn`).

## Сигурност

- AUTH се прави **само** след TLS. Ако `cfg.tls = STARTTLS`, а сървърът не
  обяви `STARTTLS` в EHLO, клиентът отказва, вместо да прати паролата в
  открит вид.
- Паролата не се логва никъде. `SmtpErr` пази само кода и текста от сървъра.
- 5xx е окончателно, 4xx е транзиентно — `smtp_err_transient` решава дали
  да се опита пак.

## Строеж

| Файл | Роля |
|--|--|
| `smtp.baga` | вход: `smtp_send`, `smtp_send_text`, `smtp_verify` |
| `src/types.baga` | `SmtpCfg`, `SmtpMsg`, `SmtpAddr`, `SmtpErr`, `SmtpOk` |
| `src/message.baga` | RFC 5322/2045/2047: дата, base64 редове, multipart, dot-stuffing |
| `src/transport.baga` | сокет + четене на редове; единственото място, което знае plain/TLS |
| `src/client.baga` | разговорът: EHLO → STARTTLS → AUTH → MAIL → RCPT → DATA → QUIT |

## Run

```bash
cd app-product/smtpbaga
BAGA=../../baga ../../sandak build
```

Тестът върви в baga (mock сървър, без реален доставчик):

```bash
BAGA=./baga ./baga -I . -I app-product tests/smtp_test.baga
# със STARTTLS покритие:
TLSKEYPATH=/tmp/baga_tls_rsa_key.pem TLSCERTPATH=/tmp/baga_tls_rsa_cert.pem \
  ./baga -I . -I app-product tests/smtp_test.baga
```

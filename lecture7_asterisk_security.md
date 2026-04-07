---
marp: true
theme: default
paginate: true
backgroundColor: '#000814'
color: '#e0e0e0'
style: |
  section {
    font-family: 'Segoe UI', Arial, sans-serif;
    background: linear-gradient(180deg, #0a0f1e 0%, #0d1527 100%);
    color: #e0e0e0;
    padding: 2.5em 3em;
  }
  h1 { color: #00b4d8; font-size: 2em; line-height: 1.15; }
  h2 { color: #48cae4; border-bottom: 2px solid #00b4d8; padding-bottom: 0.2em; }
  h3 { color: #ffffff; }
  strong { color: #90e0ef; }
  code { color: #90e0ef; background: #0d1117; padding: 0.1em 0.3em; border-radius: 3px; }
  pre { background: #0d1117; border: 1px solid #00b4d8; border-radius: 5px; padding: 0.8em; }
  table { font-size: 0.82em; width: 100%; border-collapse: collapse; }
  th { background: rgba(0,180,216,0.22); color: #48cae4; padding: 0.35em 0.6em; }
  td { padding: 0.28em 0.6em; border-bottom: 1px solid rgba(255,255,255,0.09); }
  tr:nth-child(even) td { background: rgba(255,255,255,0.04); }
  .ok { color: #52b788; }
  .er { color: #e63946; }
  .warn { color: #f77f00; }
  section.title { background: radial-gradient(circle at 60% 40%, #003566 0%, #001d3d 55%, #000814 100%); text-align: center; display: flex; flex-direction: column; align-items: center; justify-content: center; }
  section.red-bg { background: linear-gradient(135deg, #1a0000 0%, #3d0000 100%); }
  section.mid-bg { background: linear-gradient(135deg, #001d3d 0%, #003566 100%); }
---

<!-- _class: title -->

# Лекція 7

## Безпека IP-АТС на прикладі Asterisk

**Курс «Технології VoIP» · Кібербезпека (магістратура)**

Кафедра інформаційних технологій та кібербезпеки

---

## 📋 План лекції

| Блок | Тема |
|------|------|
| 1 | IP-АТС та Asterisk: архітектура і поверхня атаки |
| **2** | **Теоретичні принципи безпеки (CIA, AAA, Saltzer & Schroeder…)** |
| 3 | Мінімізація привілеїв (PoLP): ОС, ФС, мережева ізоляція |
| 4 | ACL: контроль доступу на рівні IP |
| 5 | Контексти SIP-драйверів: ізоляція груп |
| 6 | Безпека диалплану: Toll Fraud, DISA, валідація |
| 7 | Автентифікація, TLS/SRTP, AMI, моніторинг |
| 8 | Типові помилки, чек-ліст, Q&A |

---

## 🔺 CIA Тріада в контексті IP-АТС

> **CIA Triad** — фундаментальна модель інформаційної безпеки: **C**onfidentiality · **I**ntegrity · **A**vailability.

| Властивість | Загроза для IP-АТС | Захист в Asterisk |
|-------------|-------------------|-------------------|
| **Confidentiality** (Конфіденційність) | Перехоплення SIP/RTP, eavesdropping | TLS для сигналізації, SRTP для медіа |
| **Integrity** (Цілісність) | Підміна SIP-запитів, Caller ID spoofing | Digest Auth, TLS з верифікацією сертифіката |
| **Availability** (Доступність) | DoS/DDoS на SIP-порт, flood реєстрацій | fail2ban, rate limiting, VLAN-ізоляція |

**Практичне застосування:**
- Порушення **C** → витік записів переговорів, комерційної таємниці
- Порушення **I** → шахрайські дзвінки від імені легітимного номера
- Порушення **A** → АТС недоступна, бізнес-процеси зупинено

> Захист IP-АТС = забезпечення **всіх трьох** властивостей одночасно.

---

## 🔑 AAA Framework: Автентифікація · Авторизація · Аудит

> **AAA** — класична тріада контролю доступу, фундамент для управління доступом у будь-якій системі.

```
┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  Authentication │   │  Authorization   │   │   Accounting    │
│  «Хто ти?»      │   │  «Що можна?»     │   │  «Що робив?»    │
├─────────────────┤   ├──────────────────┤   ├─────────────────┤
│ SIP Digest Auth │   │ Контекст у       │   │ CDR / CEL       │
│ ACL (IP-адреса) │   │ extensions.conf  │   │ /var/log/       │
│ TLS-сертифікат  │   │ Extension pattern│   │ asterisk/       │
│ (chan_pjsip)     │   │ ACL permit/deny  │   │ SIEM-інтеграція │
└─────────────────┘   └──────────────────┘   └─────────────────┘
```

**Типова вразливість:** слабка **Authentication** відкриває всі три рівні.
**Типова помилка:** налаштована **Authentication** але відсутній **Accounting** → атаку не виявлено.

---

## 📐 8 Принципів Saltzer & Schroeder (1975)

> Класична робота «The Protection of Information in Computer Systems» — основа сучасної security engineering.

| # | Принцип | Реалізація в Asterisk |
|---|---------|----------------------|
| 1 | **Economy of Mechanism** — простота | Мінімальний набір модулів; прості контексти |
| 2 | **Fail-Safe Defaults** — безпечний дефолт | `allowguest=no`; `deny=0.0.0.0/0` першим |
| 3 | **Complete Mediation** — перевірка кожного запиту | ACL на кожному endpoint; кожен виклик через контекст |
| 4 | **Open Design** — безпека ≠ таємниця | Asterisk open-source; безпека через коректну конфігурацію |
| 5 | **Separation of Privilege** — розподіл прав | Окремі контексти; окремі AMI-акаунти |
| 6 | **Least Privilege** — мінімум прав | Запуск від `asterisk`; обмежені права AMI |
| 7 | **Least Common Mechanism** — ізоляція | VLAN; окремі процеси; не shared контексти |
| 8 | **Psychological Acceptability** — зрозумілість | Іменовані контексти; коментарі у конфігах |

---

## 🔒 Fail-Safe Defaults: Безпечна відмова

> **Принцип:** система за замовчуванням **забороняє** все, а дозволяє лише явно налаштоване.
> Якщо перевірка неможлива або виникла помилка — відмовити у доступі.

**Теорія vs Практика:**

```
Небезпечний підхід (Fail-Open):        Безпечний підхід (Fail-Secure):
─────────────────────────────          ──────────────────────────────────
дозволити все →                        заборонити все →
  заблокувати відоме шкідливе            дозволити явно перевірене
```

**Fail-Safe Defaults в Asterisk:**

```ini
; ❌ Fail-Open (небезпечний дефолт)    ; ✅ Fail-Secure (правильно)
allowguest=yes                          allowguest=no
; дозволяє анонімні дзвінки            ; відхиляє будь-який неавтентифікований

; ❌ Спочатку дозволити:               ; ✅ Спочатку заборонити:
permit=0.0.0.0/0                        deny=0.0.0.0/0
deny=10.0.0.0/8                         permit=192.168.10.0/24

; ❌ context=default (відкритий)       ; ✅ context=restricted (мінімальний)
```

> **Наслідок порушення:** `allowguest=yes` + відсутність ACL = Toll Fraud без злому паролів.

---

## 🎯 Мінімізація поверхні атаки

> **Attack Surface** — сукупність усіх точок входу, через які зловмисник може взаємодіяти з системою.
> **Принцип:** зменшити кількість та розмір цих точок до мінімально необхідного.

**Компоненти поверхні атаки Asterisk:**

```
Повна поверхня (за замовчуванням):
  UDP/5060 (SIP) · UDP/5061 (TLS-SIP) · UDP/10000-20000 (RTP)
  TCP/5038 (AMI) · TCP/8088 (HTTP/ARI) · AGI scripts · modules/

Мінімізована поверхня:
  UDP/5060 — тільки для внутрішньої підмережі
  UDP/5061 — тільки для довірених trunk-ів
  UDP/RTP — в обмеженому діапазоні (rtp.conf: rtpstart/rtpend)
  TCP/5038 — тільки loopback
  TCP/8088 — вимкнений або тільки localhost + HTTPS
```

**Вимкнення невикористаних модулів (`modules.conf`):**

```ini
[modules]
autoload=no            ; НЕ завантажувати все підряд
load => chan_pjsip.so  ; тільки те, що потрібно
load => pbx_config.so
load => app_dial.so
; chan_sip.so — НЕ завантажуємо, якщо не потрібен
; res_http_websocket.so — НЕ завантажуємо без WebRTC
```

---

## 🔐 Zero Trust для IP-АТС

> **Zero Trust:** «Ніколи не довіряй, завжди перевіряй» — жоден компонент не є автоматично довіреним, навіть всередині мережі.

**Традиційна модель (периметр) vs Zero Trust:**

```
Традиційна (довіра по зоні):        Zero Trust:
──────────────────────────          ─────────────────────────
Internet    │  LAN                  Internet    │  LAN
  ❌        │  ✅ (всі довірені)      ❌        │  перевіряється!
            │                                   │
         Asterisk                            Asterisk
    ← SIP без auth від                  ← SIP Digest Auth
      внутрішніх IP                       від КОЖНОГО IP
```

**Zero Trust в Asterisk:**

```ini
; Навіть внутрішні IP-телефони — автентифікація обов'язкова
[6001]
type=endpoint
auth=auth6001          ; пароль для КОЖНОГО endpoint
acl=internal_acl       ; + перевірка IP
context=internal       ; + обмежений контекст

; Мікросегментація: навіть VLAN-сусіди не довіряємо без auth
[voicemail_server]
type=endpoint
auth=auth_voicemail
acl=voicemail_acl
```

> Zero Trust ≠ параноя: це **архітектурний принцип**, що знижує ризик горизонтального руху зловмисника.

---

## 🏢 IP-АТС у сучасній інфраструктурі

**IP-АТС (IP PBX)** — програмна або апаратна телефонна станція, що обробляє дзвінки через IP-мережу.

**Чому безпека критична?**

- Toll Fraud завдає індустрії **~40 млрд USD** збитків щорічно (CFCA 2023)
- IP-АТС — точка концентрації корпоративних комунікацій
- Злом АТС → несанкціоновані дзвінки, прослуховування, DoS

**Популярні IP-АТС системи:**

| Open Source | Комерційні |
|-------------|-----------|
| **Asterisk** | Cisco CUCM |
| FreeSWITCH | Avaya Aura |
| Kamailio | 3CX |

---

## 🔭 Архітектура Asterisk

```
┌──────────────────────────────────────────────────────────────┐
│                        Asterisk Core                         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ Channel      │  │  Dialplan    │  │  Applications     │  │
│  │ Drivers      │  │  (extensions │  │  (Dial, Voicemail,│  │
│  │ chan_pjsip   │  │  .conf)      │  │   AGI, Queue…)    │  │
│  │ chan_dahdi   │  └──────────────┘  └───────────────────┘  │
│  └──────────────┘                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ AMI          │  │ ARI          │  │ AGI / FastAGI     │  │
│  │ (Manager)    │  │ (REST API)   │  │ (External scripts)│  │
│  └──────────────┘  └──────────────┘  └───────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Поверхня атаки:** SIP-стек · AMI · ARI · AGI-скрипти · HTTP · ФС · ОС

---

## 🔐 Принцип мінімізації привілеїв (PoLP) → Asterisk

> **Principle of Least Privilege (Saltzer & Schroeder, принцип №6):**
> кожен суб'єкт отримує рівно ті права, які необхідні для виконання своєї функції, — і не більше.

**Чому PoLP критичний для IP-АТС:**
- Компрометація одного компонента **не поширюється** на всю систему
- Зменшується збиток від помилок конфігурації та помилок у коді

**Реалізація PoLP у чотирьох рівнях Asterisk:**

```
┌──────────────────────────────────────────────────────────┐
│  Рівень 4 — Диалплан                                     │
│  Принцип: кожен контекст дозволяє тільки потрібні        │
│  напрямки (whitelist замість blacklist)                   │
├──────────────────────────────────────────────────────────┤
│  Рівень 3 — Asterisk ACL (permit/deny по IP)             │
│  Принцип: кожен endpoint бачить тільки свій ACL          │
├──────────────────────────────────────────────────────────┤
│  Рівень 2 — Мережа (VLAN, iptables, fail2ban)            │
│  Принцип: сегментація — кожна зона має мінімальний доступ│
├──────────────────────────────────────────────────────────┤
│  Рівень 1 — ОС (непривілейований користувач asterisk)    │
│  Принцип: процес не може робити більше, ніж його user    │
└──────────────────────────────────────────────────────────┘
```

**Зв'язок з Defense-in-Depth:** кожен рівень PoLP = окремий рубіж.

---

<!-- _class: mid-bg -->

## 🐧 Захист на рівні операційної системи

**Запуск від непривілейованого користувача:**

```bash
# Створення користувача та групи
useradd -r -s /sbin/nologin asterisk
groupadd asterisk

# Запуск з обмеженими правами
asterisk -U asterisk -G asterisk
```

**systemd unit-файл — захисні директиви:**

```ini
[Service]
User=asterisk
Group=asterisk
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```

> **Навіщо?** Якщо зловмисник виконає код через AGI, він не отримає root-доступу.

---

## 📁 Захист конфігураційних файлів

**Права доступу:**

```bash
chown -R asterisk:asterisk /etc/asterisk/
chmod 750 /etc/asterisk/
chmod 640 /etc/asterisk/*.conf

# Особливо чутливі файли
chmod 600 /etc/asterisk/sip.conf
chmod 600 /etc/asterisk/manager.conf
```

**Розділення секретів через `#include`:**

```ini
; sip.conf — публічна частина
[general]
bindport=5060
#include "sip_secrets.conf"   ; права 600, у .gitignore

; sip_secrets.conf
[mypeer]
secret=V3ryStr0ngP@ssw0rd!
```

> ⚠️ **Ніколи** не зберігайте паролі у файлах, доступних для читання іншим користувачам.

---

## 🌐 Мережева ізоляція АТС

**Архітектура VLAN для IP-АТС:**

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│  VLAN 10    │     │    VLAN 20       │     │   VLAN 30    │
│  IP-phones  │◄───►│  Asterisk PBX   │◄───►│  SIP Trunk   │
│  Softphones │     │  (ізольована    │     │  (провайдер) │
│  192.168.10 │     │   підмережа)    │     │  10.0.30.x   │
└─────────────┘     │  10.0.20.x      │     └──────────────┘
                    └──────────────────┘
                            │
                    ┌───────▼──────┐
                    │   VLAN 40    │
                    │  Моніторинг  │
                    │  (тільки TCP)│
                    └──────────────┘
```

Кожен VLAN має **окремий ACL** на рівні мережевого обладнання.

---

## 🛡️ ACL: Рівні захисту

**Defense-in-Depth для IP-фільтрації:**

```
Incoming SIP INVITE
        │
        ▼
┌───────────────────┐
│  1. iptables/nft  │ → блокує цілі підмережі/країни
└────────┬──────────┘
         ▼
┌───────────────────┐
│  2. fail2ban      │ → динамічно блокує brute-force IP
└────────┬──────────┘
         ▼
┌───────────────────┐
│  3. Asterisk ACL  │ → permit/deny на рівні конфігурації
└────────┬──────────┘
         ▼
┌───────────────────┐
│  4. Контекст      │ → обмежує доступні extension
└───────────────────┘
```

---

## ⚙️ ACL у chan\_sip (sip.conf)

```ini
[general]
; Глобальна заборона — дозволяємо тільки явно
deny=0.0.0.0/0.0.0.0
permit=192.168.10.0/255.255.255.0  ; внутрішня мережа

; Заборона гостьових дзвінків (КРИТИЧНО!)
allowguest=no

; Відхиляти невідомі джерела — не розкривати чи існує peer
alwaysauthreject=yes

[mytrunk]
type=peer
host=sip.provider.com
deny=0.0.0.0/0.0.0.0
permit=203.0.113.10/255.255.255.255  ; тільки IP провайдера

; Обмеження Contact-header при REGISTER
contactpermit=192.168.10.0/255.255.255.0
contactdeny=0.0.0.0/0.0.0.0
```

> ⚠️ **`allowguest=yes`** (default у старих версіях) — дозволяє дзвінки без автентифікації!

---

## ⚙️ ACL у chan\_pjsip (pjsip.conf)

```ini
; Визначення об'єкта ACL
[internal_acl]
type=acl
deny=0.0.0.0/0.0.0.0
permit=192.168.10.0/24
permit=192.168.11.0/24

[contact_acl]
type=acl
deny=0.0.0.0/0.0.0.0
permit=192.168.10.0/24

; Прив'язка ACL до endpoint
[6001]
type=endpoint
aors=6001
auth=auth6001
context=internal          ; контекст диалплану
acl=internal_acl          ; перевірка IP джерела запиту
contact_acl=contact_acl   ; перевірка Contact-header

[trunk_provider]
type=endpoint
acl=trunk_acl             ; тільки IP провайдера
context=from-trunk
```

---

## 🔥 fail2ban + iptables/nftables

**Конфігурація fail2ban для Asterisk:**

```ini
; /etc/fail2ban/filter.d/asterisk.conf
[Definition]
failregex = NOTICE.* <HOST>.*: Registration from .* failed
            NOTICE.* <HOST>.*: No matching peer found
            WARNING.* <HOST>.*: Digest auth failed
            NOTICE.* <HOST>.*: Sending fake auth rejection

; /etc/fail2ban/jail.local
[asterisk]
enabled  = true
port     = 5060,5061
filter   = asterisk
logpath  = /var/log/asterisk/messages
maxretry = 5
bantime  = 3600      ; блокування на 1 годину
findtime = 600       ; 5 спроб за 10 хвилин → бан
```

**nftables rate-limiting для SIP:**

```bash
nft add rule inet filter input udp dport 5060 \
  limit rate 30/second burst 50 packets accept
nft add rule inet filter input udp dport 5060 drop
```

---

## 🗂️ Контексти Asterisk: концепція

> **Контекст** — іменований простір у `extensions.conf`, що містить набір правил обробки дзвінків (extension patterns).

**Зв'язок endpoint → context → extension:**

```
SIP INVITE від 6001
        │
        ▼ (chan_pjsip визначає endpoint)
 context = "internal"
        │
        ▼ (Asterisk шукає в [internal])
 exten => 6002,1,Dial(PJSIP/6002)   ✓ знайдено
 exten => 9.,1,Dial(PJSIP/trunk)    ✓ дозволено

SIP INVITE від зовнішнього trunk
        │
        ▼
 context = "from-trunk"
        │
        ▼ (в [from-trunk] немає внутрішніх extension!)
 exten => s,1,Answer()              ✓ тільки IVR
 exten => 6001,1,Dial(PJSIP/6001)  ✓ маршрут до внутрішнього
```

---

## 🔒 Ізоляція груп у chan\_sip (sip.conf)

```ini
[general]
context=default            ; fallback — має бути обмеженим!

[6001]                     ; внутрішній телефон
type=friend
context=internal           ; доступ до внутрішніх + зовнішніх ліній
secret=str0ngP@ss

[6099]                     ; обмежений користувач (лобі, гість)
type=friend
context=restricted         ; тільки внутрішні номери
secret=anotherP@ss

[mytrunk]                  ; SIP-провайдер
type=peer
context=from-trunk         ; incoming дзвінки → IVR/черга
host=sip.provider.com
```

> ⚠️ **Ніколи** не ставте `context=from-trunk` для внутрішніх телефонів — trunk зазвичай не має автентифікації!

---

## 🔒 Ізоляція груп у chan\_pjsip (pjsip.conf)

```ini
[6001]
type=endpoint
context=internal           ; ← визначає доступні можливості
aors=6001
auth=auth6001
disallow=all
allow=opus,g722,ulaw

[6099]
type=endpoint
context=restricted         ; ← обмежений контекст
aors=6099
auth=auth6099

[trunk_inbound]
type=endpoint
context=from-trunk         ; ← зовнішні виклики
identify_by=ip
acl=trunk_ip_acl
; Без секрету — автентифікація тільки по IP
```

---

## 🌲 Ієрархія контекстів: include

```ini
; extensions.conf

[restricted]               ; мінімальний доступ (гості, лобі)
include => internal_phones ; тільки внутрішні

[internal]                 ; співробітники
include => internal_phones
include => outbound_local  ; місцеві дзвінки

[internal_premium]         ; керівники
include => internal
include => outbound_international

[from-trunk]               ; вхідні від провайдера
exten => s,1,Goto(ivr,s,1)
exten => +380XXXXXXXXX,1,Dial(PJSIP/6001)

; !!! КРИТИЧНА ПОМИЛКА:
; include => internal   ← в [from-trunk] → Toll Fraud!
```

**Правило:** `include` передає права вниз. Ніколи не включайте привілейований контекст у менш захищений.

---

<!-- _class: red-bg -->

## 💥 Загрози на рівні диалплану

### Toll Fraud
Зловмисник реєструється (або обходить автентифікацію) і дзвонить на дорогі номери за рахунок жертви.

### DISA (Direct Inward System Access)
Вхід у АТС ззовні з відкритим виходом у зовнішні лінії через введення PIN.

### Voicemail → Outbound
VM-меню дозволяє передзвонити на довільний номер з зовнішньої лінії АТС.

### AGI/System() injection
Некоректна валідація вводу в AGI-скриптах або `System()` — виконання системних команд.

### IVR DTMF Abuse
Ін'єкція через DTMF-введення в IVR для переходу на заборонені extension.

---

## 💸 Toll Fraud: Механізм атаки

```
1. Розвідка:
   svmap -r 5060 target_ip      # сканування SIP-сервісу
   svwar -e100-200 target_ip    # перебір extension

2. Brute-force паролів:
   svcrack -u 6001 target_ip    # підбір секрету

3. Реєстрація скомпрометованого акаунта:
   REGISTER sip:6001@target ... Authorization: ...

4. Генерація дорогих дзвінків:
   INVITE sip:+1900XXXXXXX@target ...
   INVITE sip:+441XXXXXXXXX@target ...  (міжнародні)

5. Збиток: сотні дзвінків за кілька годин
   → рахунок від провайдера = тисячі доларів
```

**Типові цілі:** +1900 (преміум США), +44 (Великобританія), +371 (Латвія), міжнародні premium-rate.

---

## 🛡️ Захист від Toll Fraud у диалплані

```ini
; extensions.conf — [outbound_rules]

; Дозволяємо тільки явний whitelist:
; Місцеві номери (10 цифр)
exten => _NXXXXXXXXX,1,Dial(PJSIP/trunk/${EXTEN})

; Служби екстреної допомоги
exten => 112,1,Dial(PJSIP/trunk/112)
exten => 101,1,Dial(PJSIP/trunk/101)

; Заборона преміум та міжнародних (для [internal]):
; (не включаємо outbound_international)

; [outbound_international] — тільки для авторизованих:
exten => _+.,1,Dial(PJSIP/trunk/${EXTEN})
exten => _011.,1,Dial(PJSIP/trunk/${EXTEN})

; Явна заборона преміум (у будь-якому контексті):
exten => _1900NXXXXXX,1,Playback(ss-noservice)
exten => _1900NXXXXXX,2,Hangup()
```

---

## ✅ Валідація введення в диалплані

**Небезпечний патерн — довільне введення:**

```ini
; ❌ НЕБЕЗПЕЧНО: будь-який введений номер
exten => s,1,Read(USERNUM)
exten => s,2,Dial(PJSIP/trunk/${USERNUM})
```

**Безпечна валідація через `FILTER()`:**

```ini
; ✅ БЕЗПЕЧНО: дозволяємо тільки цифри
exten => s,1,Read(USERNUM)
exten => s,2,Set(CLEAN=${FILTER(0-9,${USERNUM})})
exten => s,3,GotoIf($[${LEN(${CLEAN})} < 4]?invalid)
exten => s,4,GotoIf($[${LEN(${CLEAN})} > 15]?invalid)
exten => s,5,Goto(outbound_rules,${CLEAN},1)
exten => s,n(invalid),Playback(invalid)
exten => s,n,Hangup()
```

**`System()` і `AGI()` — лише з контрольованими аргументами:**

```ini
; ❌ НЕБЕЗПЕЧНО
exten => s,1,System(echo ${CALLERID(num)} >> /tmp/log)

; ✅ БЕЗПЕЧНО — фіксований аргумент
exten => s,1,System(/usr/local/bin/log_call.sh)
```

---

## 🚫 Захист від DISA

**Що таке DISA:**

```
Зовнішній дзвінок → АТС → DISA() → введення PIN
     → отримання dial tone → дзвінок на будь-який номер
```

**Якщо DISA необхідна — мінімальний захист:**

```ini
exten => s,1,DISA(no-password,restricted)
;               ↑ файл з PIN-кодами  ↑ обмежений контекст

; /etc/asterisk/disa.conf:
; 7491823654|restricted   ; складний PIN + обмежений контекст
```

**Кращий підхід — замінити DISA на VPN:**

```
Замість: Зовнішній дзвінок → DISA → зовнішній trunk
Краще:   VPN-підключення → internal context → SIP-дзвінок
```

> ⚠️ DISA без PIN або зі слабким PIN = відкритий trunk для будь-кого.

---

## 📞 Обмеження одночасних дзвінків

**Захист від масового toll fraud після компрометації:**

```ini
; Обмеження одночасних каналів на акаунт
exten => _X.,1,Set(GROUP()=outbound_${CALLERID(num)})
exten => _X.,2,Set(COUNT=${GROUP_COUNT(outbound_${CALLERID(num)})})
exten => _X.,3,GotoIf($[${COUNT} > 3]?toolimit)
exten => _X.,4,Dial(PJSIP/trunk/${EXTEN})
exten => _X.,n(toolimit),Playback(conf-locked)
exten => _X.,n,Hangup(17)  ; User Busy

; Глобальне обмеження вихідних на trunk
[from-trunk-outbound]
exten => _X.,1,Set(GROUP()=trunk_out)
exten => _X.,2,GotoIf($[${GROUP_COUNT(trunk_out)} > 30]?overlimit)
```

**Також у `sip.conf` / `pjsip.conf`:**

```ini
; chan_pjsip endpoint:
max_contacts=2            ; не більше 2 реєстрацій одночасно
```

---

## 🔑 SIP Digest Authentication

**Вимоги до паролів у Asterisk:**

```ini
; sip.conf / pjsip auth object
[6001]
type=auth
auth_type=userpass
username=6001
password=P@$$w0rd!2024Str0ng   ; мінімум 16 символів, складний

; Захист від username enumeration:
; sip.conf:
alwaysauthreject=yes    ; завжди відхиляти з "fake" challenge

; Обмеження реєстрацій:
maxexpirey=3600         ; максимальний термін реєстрації
minexpirey=60           ; мінімальний
```

**Перевірка паролів через CLI:**

```bash
# Перевірка поточних реєстрацій
asterisk -rx "pjsip show registrations"

# Активні endpoints
asterisk -rx "pjsip show endpoints"
```

---

## 🔐 TLS та SRTP в Asterisk

**Налаштування TLS для SIP (pjsip.conf):**

```ini
[transport-tls]
type=transport
protocol=tls
bind=0.0.0.0:5061
cert_file=/etc/asterisk/certs/server.crt
priv_key_file=/etc/asterisk/certs/server.key
ca_list_file=/etc/asterisk/certs/ca.crt
method=tlsv1_2            ; мінімум TLS 1.2

[6001]
type=endpoint
transport=transport-tls
media_encryption=srtp     ; обов'язковий SRTP
media_encryption_optimistic=no  ; не допускати незашифроване
dtls_cert_file=/etc/asterisk/certs/server.crt
```

**Генерація сертифікатів (вбудований скрипт):**

```bash
cd /usr/share/asterisk/scripts
./ast_tls_cert -C pbx.company.com -O "Company" \
               -d /etc/asterisk/certs
```

---

<!-- _class: mid-bg -->

## 🔧 Захист AMI (Asterisk Manager Interface)

**AMI надає повний контроль над Asterisk — потребує максимального захисту.**

```ini
; /etc/asterisk/manager.conf
[general]
enabled=yes
port=5038
bindaddr=127.0.0.1        ; тільки loopback!
; bindaddr=0.0.0.0       ← НІКОЛИ!

[monitoring_user]
secret=Str0ngAMISecret!
deny=0.0.0.0/0.0.0.0
permit=127.0.0.1/255.255.255.255
permit=10.0.40.5/255.255.255.255  ; тільки моніторинг-хост
read=system,call,log,cdr
write=command             ; мінімум необхідних прав

[readonly_user]
secret=AnotherSecret!
permit=10.0.40.5/255.255.255.255
read=call,cdr             ; тільки читання CDR
write=                    ; без запису
```

---

## 📊 Логування та моніторинг

**Конфігурація `logger.conf`:**

```ini
[logfiles]
; Повне логування в файл
messages => notice,warning,error,verbose,dtmf,fax
; Security-події окремо (для SIEM)
security => security
; Console для дебагу
console => notice,warning,error
```

**Ключові SECURITY-події Asterisk:**

```
SECURITY[1234] InvalidPassword: "SIP/192.168.1.100"
SECURITY[1234] InvalidAccountID: "SIP/6099"
SECURITY[1234] FailedACL: "SIP/10.0.0.50" acl_name="internal_acl"
SECURITY[1234] RequestNotAllowed: "AMI/unauthorized"
```

**Команди для моніторингу:**

```bash
asterisk -rx "core show channels verbose"
asterisk -rx "pjsip show channels"
tail -f /var/log/asterisk/security | grep -E "Fail|Invalid"
sngrep -I capture.pcap                 # аналіз SIP
```

---

<!-- _class: red-bg -->

## ⚠️ Типові помилки конфігурації

| Помилка | Наслідок | Виправлення |
|---------|----------|-------------|
| `allowguest=yes` | Дзвінки без автентифікації | `allowguest=no` |
| Слабкий пароль AMI | Повний контроль над АТС | 20+ символів, складний |
| `context=from-trunk` для IP-phones | Обхід білого списку | Окремі контексти |
| AMI на `0.0.0.0:5038` | Доступ з будь-якого IP | `bindaddr=127.0.0.1` |
| `include => internal` в `from-trunk` | Toll Fraud через trunk | Ізоляція контекстів |
| Asterisk від root | Компрометація ОС | `-U asterisk -G asterisk` |
| `System(${USERDATA})` | Command injection | `FILTER()` + фіксовані аргументи |
| Відсутність fail2ban | Brute-force без обмежень | fail2ban + rate limiting |
| Відкритий DISA без PIN | Безкоштовний доступ до trunk | VPN або складний PIN |

---

## ✅ Чек-ліст безпеки Asterisk

**ОС та файлова система:**
- [ ] Asterisk запускається від користувача `asterisk` (не root)
- [ ] `systemd`: `NoNewPrivileges`, `ProtectSystem=strict`
- [ ] `/etc/asterisk/` — права `750`, файли `640` або `600`

**Мережевий рівень:**
- [ ] SIP-порти відкриті тільки для необхідних підмереж
- [ ] fail2ban активний з правилами для Asterisk
- [ ] AMI (`5038`) прив'язаний до `127.0.0.1`
- [ ] `nftables`/`iptables` rate-limiting для SIP

**Конфігурація Asterisk:**
- [ ] `allowguest=no` (chan_sip)
- [ ] `alwaysauthreject=yes` (chan_sip)
- [ ] ACL `permit`/`deny` для кожного endpoint і trunk
- [ ] Окремі контексти: `internal`, `from-trunk`, `restricted`

---

## ✅ Чек-ліст безпеки Asterisk (продовження)

**Диалплан:**
- [ ] Whitelist вихідних напрямків
- [ ] Заборона преміум-номерів (1900, international)
- [ ] `GROUP_COUNT()` — обмеження одночасних дзвінків
- [ ] `FILTER()` для будь-якого зовнішнього введення
- [ ] `System()` і `AGI()` — без змінних від користувача
- [ ] DISA видалена або замінена VPN

**Автентифікація та шифрування:**
- [ ] Паролі SIP: мінімум 16 символів
- [ ] TLS для SIP (`tlsv1_2` або `tlsv1_3`)
- [ ] SRTP увімкнений (`media_encryption=srtp`)
- [ ] AMI: окремі акаунти з мінімальними правами

**Моніторинг:**
- [ ] `security` лог увімкнений
- [ ] CDR/CEL зберігаються та аналізуються
- [ ] Інтеграція з SIEM або централізованим логуванням
- [ ] Алерти при аномальних патернах дзвінків

---

## 🔍 Інструменти для аудиту

| Інструмент | Призначення | Команда |
|------------|------------|---------|
| **sipvicious** | Тест на проникнення SIP | `svmap target`, `svcrack -u user target` |
| **sngrep** | Аналіз SIP у терміналі | `sngrep -d eth0 port 5060` |
| **Wireshark** | Глибокий аналіз пакетів | Фільтр: `sip or rtp` |
| **fail2ban-client** | Перевірка блокувань | `fail2ban-client status asterisk` |
| **nmap** | Виявлення відкритих портів | `nmap -sU -p 5060 target` |
| **asterisk CLI** | Внутрішній стан | `asterisk -rx "pjsip show endpoints"` |
| **grep/awk** | Аналіз security-логів | `grep SECURITY /var/log/asterisk/security` |

**Рекомендований workflow аудиту:**
1. Сканування портів (`nmap`) → виявлення поверхні атаки
2. SIP-сканування (`svmap`) → перевірка відкритості
3. Аналіз конфігурацій → пошук слабких місць
4. Перевірка ACL та контекстів
5. Аналіз логів → виявлення попередніх інцидентів

---

## 📌 Підсумок: теорія → практика Asterisk

| Принцип безпеки | Реалізація в Asterisk |
|----------------|----------------------|
| **CIA Тріада** | TLS+SRTP (C), Digest Auth (I), fail2ban+rate-limit (A) |
| **AAA Framework** | Digest/ACL (Authn), контексти (Authz), CDR/CEL/SIEM (Accounting) |
| **Fail-Safe Defaults** | `allowguest=no`, `deny=0.0.0.0/0` першим, `alwaysauthreject=yes` |
| **Least Privilege** | user `asterisk`, мінімальні AMI-права, вузькі контексти |
| **Attack Surface Min.** | `autoload=no`, закриті порти, VLAN-ізоляція |
| **Complete Mediation** | ACL на кожному endpoint, кожен виклик через контекст |
| **Zero Trust** | Auth для ВСІХ endpoint, навіть внутрішніх |
| **Defense-in-Depth** | ОС → мережа → Asterisk ACL → контекст → диалплан |

**Ключовий висновок:** безпека IP-АТС — це **послідовне** застосування загальних принципів ІБ на специфічних рівнях телефонної системи, а не набір окремих «трюків».

---

## 📚 Ресурси та RFC

**Офіційна документація:**
- [wiki.asterisk.org — Security](https://wiki.asterisk.org/wiki/display/AST/Asterisk+Security+Best+Practices)
- [wiki.asterisk.org — PJSIP](https://wiki.asterisk.org/wiki/display/AST/PJSIP+Configuration+Sections+and+Relationships)

**Стандарти:**
- **RFC 3261** — SIP
- **RFC 3711** — SRTP
- **RFC 4568** — SDES (SDP Security Descriptions)
- **RFC 5764** — DTLS-SRTP

**Інструменти безпеки:**
- SIPVicious Suite — `github.com/enablesecurity/sipvicious`
- fail2ban — `fail2ban.org`
- OWASP VoIP Security Project

**Книги:**
- *"Hacking VoIP"* — Himanshu Dwivedi
- *"Asterisk: The Definitive Guide"* — Madsen, Van Meggelen, Smith

---

<!-- _class: title -->

## 🙋 Дякую за увагу!

### Запитання?

---

**Контрольні запитання:**

1. Чому `allowguest=yes` є критичною вразливістю?
2. Яка різниця між ACL у `sip.conf` та правилами `iptables`?
3. Чому не можна включати `[internal]` у контекст `[from-trunk]`?
4. Опишіть сценарій Toll Fraud крок за кроком.
5. Як `FILTER()` захищає від ін'єкцій у диалплані?

---

*RFC: 3261 · 3711 · 4568 · 5764*
*Навігація: ← → (стрілки) | F — повний екран (у Marp)*

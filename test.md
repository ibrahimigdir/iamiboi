# iBoi SOC Bot — Specification

> Hetzner VPS ve M70q Home Server için AI destekli, Telegram tabanlı güvenlik izleme ve otomasyon botu.

## 1. Overview

### 1.1 What Is iBoi SOC Bot?

iBoi SOC Bot, Hetzner VPS ve M70q Home Server altyapılarını izleyen, yapay zeka destekli bir Telegram tabanlı Security Operations Center (SOC) aracıdır. aiogram 3.x ve FastAPI kullanılarak inşa edilmiş olup, Groq API üzerinden LLM destekli analiz, otomatik alerting ve servis durumu izleme yetenekleri sunar.

**Temel Yetenekler:**
- **Servis İzleme:** nginx, SSH, firewall (ufw/iptables) servislerinin gerçek zamanlı durum takibi
- **AI Analiz:** Groq LLM destekli log analizi ve güvenlik olayı değerlendirmesi
- **Otomatik Alerting:** Kritik olaylarda Telegram anlık bildirimleri
- **Interactive UI:** Custom keyboard (replyKeyboardMarkup) tabanlı kontrol paneli
- **Otomatik Yanıt:** AI destekli doğal dil sorguları ile sistem durumu sorgulama

### 1.2 Target Audience

- **Birincil:** Kendi altyapısını yöneten bireysel geliştiriciler ve küçük ekipler
- **İkincil:** Ev sunucusu (homelab) kullanan IT meraklıları
- **Üçüncül:** Küçük işletmeler için uygun maliyetli SOC çözümü arayanlar

### 1.3 Key Differentiators

- **Groq Destekli AI Analiz:** Geleneksel rule-based sistemlerin ötesinde, LLM ile bağlamsal güvenlik değerlendirmesi
- **Minimal Ayak İzi:** Docker + systemd ile basit deployment, SQLite ile sıfır external DB bağımlılığı
- **Dual Infrastructure Monitor:** Hem cloud (Hetzner) hem on-premise (M70q) tek panelden izleme
- **Interactive Telegram Native:** Inline olmayan, direct reply keyboard ile hızlı kontrol
- **Self-contained Automation:** Dış bağımlılık yok - log analizi, ban/unban, raporlama bot içinde

### 1.4 Competitive Landscape

| Feature | iBoi SOC Bot | Traditional SOC Tools | Cloud-native Monitoring |
|---------|-------------|---------------------|------------------------|
| Telegram Native | ✅ | ❌ | ❌ |
| Groq AI Analysis | ✅ | ❌ | ❌ |
| Self-hosted | ✅ | ✅ | ❌ |
| SQLite Backend | ✅ | ❌ | ❌ |
| Auto Ban/Unban | ✅ | ⚠️ | ⚠️ |
| Custom Keyboard UI | ✅ | ❌ | ❌ |
| Zero External DB | ✅ | ❌ | ❌ |

---

## 2. Core Concepts

| Concept | Definition |
|---------|-----------|
| **SOC (Security Operations Center)** | Güvenlik olaylarının izlenmesi, tespiti ve analiz edildiği merkezi birim |
| **Hetzner VPS** | Hetzner Cloud üzerinde barındırılan sanal sunucu |
| **M70q Home Server** | Lenovo M70q mini PC üzerinde çalışan on-premise sunucu |
| **Groq LLM** | GroqCloud üzerinde çalışan büyük dil modeli (Llama3, Mixtral vb.) |
| **aiogram** | Python için asynchronous Telegram Bot framework |
| **FastAPI** | Modern, hızlı Python web framework |
| **replyKeyboardMarkup** | Telegram native custom klavye (buton tabanlı menü) |
| **ufw/iptables** | Linux firewall yönetim araçları |
| **Auto-ban** | SSH brute-force veya şüpheli aktivite tespitinde otomatik IP ban |
| **Log Anomalisi** | Normal dışı log pattern'i tespiti |

---

## 3. Functional Requirements

### 3.1 Servis İzleme (Service Monitoring)

#### 3.1.1 Servis Durumu Kontrolü

**User Story:** Sistem yöneticisi olarak, sunuculardaki kritik servislerin (nginx, ssh, firewall) durumunu anlık olarak görmek istiyorum.

**Description:** Bot, her iki sunucuda (Hetzner VPS, M70q) çalışan servislerin canlı durumunu sorgular ve kullanıcıya reply keyboard üzerinden sunar.

**Acceptance Criteria:**
- [ ] `/status` komutu ile her iki sunucunun servis durumu görüntülenir
- [ ] nginx, ssh, ufw servislerinin "active/inactive/failed" durumu gösterilir
- [ ] Servis durumu 30 saniyeden eski ise "stale" olarak işaretlenir
- [ ] Her sunucu için ayrı status kartı formatlanır

**Edge Cases:**
- [SSH servisi down ise uzaktan müdahale opsiyonları sunulur]
- [Network bağlantısı kaybolursa "Connection Timeout" mesajı]
- [Paralel sorgulama yapılamazsa kuyruk mekanizması]

#### 3.1.2 Periodic Health Check

**User Story:** Otomatik olarak periyodik sağlık kontrolü yapılmasını istiyorum ki sürekli komut vermek zorunda kalmayayım.

**Description:** Bot, arka planda periyodik olarak (varsayılan: 5 dakika) servis durumlarını kontrol eder ve değişiklik tespit ederse alert gönderir.

**Acceptance Criteria:**
- [ ] Konfigürasyondan ayarlanabilir check interval (1-60 dakika)
- [ ] Durum değişikliğinde otomatik Telegram bildirimi
- [ ] Değişiklik logları SQLite'e kaydedilir
- [ ] Background scheduler bot başlatıldığında aktif olur

### 3.2 AI Analiz (Groq Integration)

#### 3.2.1 Log Analizi

**User Story:** SSH auth log'larını AI'a analiz ettirerek şüpheli aktiviteleri tespit etmek istiyorum.

**Description:** Kullanıcı `/analyze <log_icerigi>` komutu verdiğinde, içerik Groq LLM'e gönderilir ve güvenlik değerlendirmesi döndürülür.

**Acceptance Criteria:**
- [ ] `/analyze` komutu ile log içeriği gönderilebilir
- [ ] Analiz sonucu: risk seviyesi (low/medium/high/critical) + açıklama
- [ ] Şüpheli IP tespit edilirse otomatik ban önerisi
- [ ] Analiz geçmişi veritabanında saklanır

**Edge Cases:**
- [Boş log içeriği gönderilirse hata mesajı]
- [Groq API hatasında fallback mesajı + retry opsiyonu]
- [Çok uzun log (>4000 token) için otomatik truncate]

#### 3.2.2 Natural Language Query

**User Story:** Doğal dil ile sistem durumunu sorgulayabilmek istiyorum.

**Description:** `/ask <soru>` komutu ile kullanıcı doğal dilde soru sorabilir, bot Groq LLM üzerinden yanıt döndürür.

**Acceptance Criteria:**
- [ ] `/ask` komutu ile serbest metin sorgusu
- [ ] Yanıt sistem durumu, loglar veya genel güvenlik tavsiyesi içerebilir
- [ ] Konfigürasyon bilgisi ile bağlam sağlanır
- [ ] Rate limit: dakikada 5 sorgu

### 3.3 Otomatik Alerting

#### 3.3.1 Brute-force Detection & Auto-ban

**User Story:** SSH brute-force saldırılarını otomatik olarak tespit edip IP banlamak istiyorum.

**Description:** Bot, auth log'larını izler ve başarısız SSH deneme sayısı kritik eşiği aştığında otomatik olarak IP'yi banlar.

**Acceptance Criteria:**
- [ ] `/analyze` veya periodic check ile brute-force tespiti
- [ ] Kritik eşik: 10 başarısız deneme (konfigüre edilebilir)
- [ ] Otomatik ban komutu oluşturulur (ufw deny from <ip>)
- [ ] Ban işlemi loglanır ve kullanıcıya bildirilir

**Edge Cases:**
- [False positive: aynı IP farklı portlardan deneme]
- [True positive: bot banladıktan sonra kullanıcı onayı istenir]
- [Zaten banlı IP tekrar banlanmaya çalışılırsa ignore]

#### 3.3.2 Alert Thresholds

**User Story:** Alert eşiklerini kendi ihtiyaıma göre ayarlamak istiyorum.

**Description:** Konfigürasyon dosyasından alert parametreleri ayarlanabilir.

**Acceptance Criteria:**
- [ ] `ALERT_THRESHOLD_SSH_FAILURES`: varsayılan 10
- [ ] `ALERT_THRESHOLD_AUTH_RATE`: dakikada X deneme
- [ ] `ALERT_CHECK_INTERVAL`: kontrol sıklığı (saniye)

### 3.4 Interactive UI (Custom Keyboard)

#### 3.4.1 Main Menu Keyboard

**User Story:** Tek bir menüden tüm komutlara hızlı erişim istiyorum.

**Description:** Ana.replyKeyboardMarkup menüsü sunulur.

**Keyboard Layout:**
```
[🌐 Sunucular] [📊 Durum] [🔍 Analiz]
[🛡️ Güvenlik] [📈 Raporlar] [⚙️ Ayarlar]
```

**Acceptance Criteria:**
- [ ] `/start` komutu ile keyboard görüntülenir
- [ ] Her buton对应的 handler mevcuttur
- [ ] Keyboard persistence - yeniden başlatmada korunur
- [ ] `resize_keyboard=True` - ekran boyutuna uygun

#### 3.4.2 Sub-menu Keyboards

**Description:** Alt menüler için inline olmayan reply keyboard'lar.

**Sunucular Menüsü:**
```
[🖥 Hetzner VPS] [🏠 M70q Home]
[📋 Tüm Sunucular] [🔙 Geri]
```

**Güvenlik Menüsü:**
```
[🚫 Ban Listesi] [✅ Unban] [📜 Son Banlar]
[🔙 Geri]
```

### 3.5 Daily Reporting

#### 3.5.1 Otomatik Günlük Rapor

**User Story:** Her gün otomatik olarak güvenlik raporu almak istiyorum.

**Description:** Bot, günlük (varsayılan: 09:00 UTC) özet rapor gönderir.

**Rapor İçeriği:**
- [ ] Günlük aktif/pasif servis sayısı
- [ ] Tespit edilen şüpheli IP'ler
- [ ] Toplam brute-force girişim sayısı
- [ ] Sistem uptime bilgisi
- [ ] AI analiz özeti (varsa)

**Acceptance Criteria:**
- [ ] `/report` komutu ile anlık rapor
- [ ] Otomatik rapor zamanı konfigüre edilebilir
- [ ] Rapor formatı: markdown table + emoji
- [ ] Rapor geçmişi SQLite'de saklanır

### 3.6 Veritabanı Yönetimi

#### 3.6.1 SQLite Schema

**Description:** Tüm veri SQLite üzerinde saklanır.

**Tables:**
- `alerts`: Alert kayıtları (id, timestamp, type, server, message, severity)
- `ban_history`: Ban kayıtları (id, timestamp, ip, reason, banned_by, active)
- `analysis_logs`: AI analiz geçmişi (id, timestamp, input, output, risk_level)
- `server_status`: Son bilinen sunucu durumları (id, server_name, service, status, last_check)
- `config`: Bot konfigürasyonu (key, value)

**Acceptance Criteria:**
- [ ] Otomatik schema migration
- [ ] Veritabanı dosyası: `./data/socbot.db`
- [ ] CRUD operasyonları için repository pattern

---

## 4. Architecture Overview

### 4.1 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                      Telegram User                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Telegram Bot (aiogram 3.x)                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Handlers   │  │  Keyboard   │  │  State Management    │  │
│  │  (commands) │  │  (reply_    │  │  (FSM if needed)     │  │
│  │             │  │   markup)   │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
┌─────────────────┐ ┌───────────┐ ┌──────────────────────────┐
│  FastAPI Server  │ │  Groq     │ │   Monitoring Agent       │
│  (Health Check)  │ │  Client   │ │   (Service Checker)      │
│                  │ │           │ │                          │
│  /health         │ │ /analyze  │ │  periodic_checks()       │
│  /metrics        │ │ /ask      │ │  check_services()        │
└─────────────────┘ └───────────┘ └──────────────────────────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      SQLite Database                          │
│           ./data/socbot.db                                     │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Component Interactions

| Component | Responsibility | Communicates With |
|-----------|----------------|-------------------|
| `telegram_bot` | User input handling, UI | `keyboard_menus`, `services` |
| `keyboard_menus` | Reply keyboard definitions | `telegram_bot` |
| `services` | Business logic orchestration | `db`, `groq_client`, `monitor` |
| `groq_client` | Groq API interaction | `services` |
| `monitor` | Service health checks | `services`, `servers` |
| `db` | SQLite operations | `services` |
| `fastapi_server` | HTTP endpoints, health | `services`, `db` |
| `servers` | Remote server SSH/command execution | `monitor` |

### 4.3 External Integrations

| Service | Purpose | Fallback |
|---------|---------|----------|
| **Telegram API** | Bot mesajlaşma | Retry 3x, then log error |
| **Groq API** | LLM analysis | Return "Analiz şu an mevcut değil" |
| **Hetzner VPS** | Servis izleme (SSH) | Timeout 10s, mark offline |
| **M70q Home Server** | Servis izleme (SSH) | Timeout 10s, mark offline |

---

## 5. Data Model

### 5.1 Core Entities

#### Alert

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | INTEGER | Yes | Primary key, auto-increment |
| timestamp | DATETIME | Yes | UTC timestamp |
| type | TEXT | Yes | ssh_bruteforce, service_down, firewall_block |
| server | TEXT | Yes | hetzner, m70q |
| message | TEXT | Yes | Alert message |
| severity | TEXT | Yes | low, medium, high, critical |
| acknowledged | BOOLEAN | No | User acknowledged |

#### BanRecord

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | INTEGER | Yes | Primary key |
| timestamp | DATETIME | Yes | UTC timestamp |
| ip_address | TEXT | Yes | Banned IP |
| reason | TEXT | Yes | Ban reason |
| banned_by | TEXT | Yes | auto, manual |
| is_active | BOOLEAN | Yes | Currently active ban |

#### ServerStatus

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | INTEGER | Yes | Primary key |
| server_name | TEXT | Yes | hetzner, m70q |
| service_name | TEXT | Yes | nginx, ssh, ufw |
| status | TEXT | Yes | active, inactive, failed, unknown |
| last_check | DATETIME | Yes | Last check timestamp |

### 5.2 Data Lifecycle

- **Alert:** Soft delete after 30 days, hard delete after 90 days
- **BanRecord:** Active bans kept indefinitely, inactive after 30 days archived
- **Analysis:** Hard delete after 7 days
- **ServerStatus:** Latest record per server/service kept, history trimmed to 7 days

---

## 6. Non-Goals (What This Project Does NOT Do)

- [ ] Web dashboard veya admin panel (Telegram UI only)
- [ ] Multi-user / team desteği (tek kullanıcı)
- [ ] Cloud log storage entegrasyonu (yalnızca yerel SQLite)
- [ ] Otomatik yazılım güncelleme
- [ ] Container orchestrator (Kubernetes vb.)
- [ ] SSL sertifika yenileme otomasyonu
- [ ] Database cluster veya replication

---

## 7. Configuration

### 7.1 Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TELEGRAM_BOT_TOKEN` | Yes | - | Telegram bot token |
| `GROQ_API_KEY` | Yes | - | Groq API key |
| `HETZNER_HOST` | Yes | - | Hetzner VPS IP/hostname |
| `HETZNER_USER` | Yes | - | SSH user for Hetzner |
| `HETZNER_KEY_PATH` | Yes | - | Path to SSH private key |
| `M70Q_HOST` | Yes | - | M70q IP/hostname |
| `M70Q_USER` | Yes | - | SSH user for M70q |
| `M70Q_KEY_PATH` | Yes | - | Path to SSH private key |
| `DATABASE_PATH` | No | ./data/socbot.db | SQLite path |
| `ADMIN_USER_ID` | Yes | - | Telegram user ID for admin |
| `CHECK_INTERVAL` | No | 300 | Health check interval (seconds) |
| `BRUTEFORCE_THRESHOLD` | No | 10 | Failed login attempts before ban |

---

## 8. Deployment

### 8.1 Docker

- **Base Image:** `python:3.11-slim`
- **Container:** Single container, non-privileged
- **Volumes:** `./data` for SQLite, `./keys` for SSH keys
- **Health Check:** `curl localhost:8000/health`

### 8.2 Systemd

- **Service Name:** `iboi-soc-bot.service`
- **Restart Policy:** `always`
- **User:** Non-root service user
- **Socket Activation:** Not used

---

## 9. Security Model

### 9.1 Bot Access Control

- **Admin-only:** Sadece `ADMIN_USER_ID` ile eşleşen Telegram user ID'leri bot kullanabilir
- **Unauthorized Access:** Diğer kullanıcılar "Yetkisiz erişim" mesajı alır
- **Command Validation:** Tüm komutlar admin ID kontrolü sonrası çalışır

### 9.2 SSH Key Management

- SSH anahtarları container dışında tutulur (`./keys/`)
- Key dosyaları: `0600` permissions
- Container read-only mount ile key'lere erişir

### 9.3 Secrets Management

- Environment variable'lar ile secrets (`docker-compose.yml` veya `.env`)
- `GROQ_API_KEY` ve `TELEGRAM_BOT_TOKEN` loglarda maskelenir
- SQLite veritabanı encryption yok (yerel güvenlik yeterli)

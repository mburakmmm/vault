# VAULT — Proje Yol Haritası ve Görev Takip Listesi (`todo.md`)

> **Proje:** `vault` — NOX 2.x için Bağımsız, Güvenli, Tip Korumalı Konfigürasyon, Ortam ve Sır (Secret) Yönetim Kütüphanesi  
> **Temel Altyapı:** NOX v2.0.123 (Doğrulanmış ve Sabitlenmiş)  
> **Kaynak Doküman:** [`AGENTS.md`](./AGENTS.md)  
> **Durum:** v0.1.0 Hazırlık / Faz 0 Başlangıcı  

---

## 🎯 Proje Hedefleri ve Temel İlkeler

1. **NOX Native & Standalone:** Dış ekosistem bağımlılığı olmaksızın sadece NOX 2.x ve standart kütüphane (`nox.os`, `nox.fs`, `nox.path`, `nox.strings`, `nox.json`, `nox.toml`, `nox.yaml`, `nox.crypto`, `nox.log`, `nox.time`) kullanılarak yazılır.
2. **Never Invent APIs:** NOX standart kütüphane ve dil özellikleri varsayılmayacak; önce yetenek probları (capability probes) derlenip çalıştırılarak doğrulanacaktır.
3. **Safe by Default & Fail-Closed:** `.env` dosyası okunurken işletim sistemi ortamı (`os.environ`) otomatik kirletilmez. Zorunlu değerler eksik veya geçersizse sessiz fallback yapılmaz, derhal hata verilir.
4. **Sıfır Sır Sızıntısı (Zero Secret Leakage):** Sır (secret) işaretli hiçbir değer istisnalarda, hata mesajlarında, `to_json` çıktılarında, loglarda veya raporlarda yer alamaz. Otomatik maskeleme `[REDACTED]` zorunludur.
5. **Deterministik Öncelik & Provenance:** Hangi ayarın hangi kaynaktan kazandığı tekil, net ve sır içermeyen bir izleme kaydıyla (`explain`) doğrulanabilir olmalıdır.
6. **Sınırlandırılmış Girdi (Bounded Limits):** Dosya boyutu, anahtar sayısı, değer uzunluğu ve interpolasyon derinliği katı sınırlarla korunacaktır.

---

## 📅 Sürüm ve Faz Dağılımı

| Sürüm | Faz | Kapsam | Aşama Durumu |
| :--- | :--- | :--- | :--- |
| **v0.1.0** | **Faz 0** | Yetenek Denetimi (Capability Audit), NOX Probları ve İskelet Paket | [x] Tamamlandı |
| **v0.2.0** | **Faz 1** | Saf (Pure) & Bounded `.env` Ayrıştırıcı (Parser & Tokenizer) | [x] Tamamlandı |
| **v0.2.5** | **Faz 2** | Çekirdek Anlık Görüntü (Snapshot), Kaynaklar & Deterministik Öncelik | [x] Tamamlandı |
| **v0.3.0** | **Faz 3** | Tipli Şema (Typed Schema) & Toplu Güvenli Doğrulama (**MVP Sürümü**) | [x] Tamamlandı |
| **v0.4.0** | **Faz 4** | Yapılandırılmış Kaynaklar (JSON, TOML, YAML Köprüleri) & Kök Korumalı Dosyalar | [x] Tamamlandı |
| **v0.5.0** | **Faz 5** | Profiller (Profiles), Değişken İnterpolasyonu & Sır Bulaşması (Secret Taint) | [ ] Beklemede |
| **v0.6.0** | **Faz 6** | Sır Sınıflandırması, Maskeleme (Redaction) & Güvenli İnceleme (**Pratik Paket**) | [ ] Beklemede |
| **v0.7.0** | **Faz 7** | Yerel Sır Sağlayıcıları (Env, Dosya ve Bellek Sağlayıcıları) | [ ] Beklemede |
| **v0.8.0** | **Faz 8** | Doğrulanmış Yeniden Yükleme (Atomic Reload) & Son Bilinen İyi Durum | [ ] Beklemede |
| **v0.9.0** | **Faz 9** | Opsiyonel Uzak Sır Sağlayıcı Çerçevesi (HTTPS, İzolasyon, SSRF Savunması) | [ ] Beklemede |
| **v0.9.5** | **Faz 10** | Opsiyonel CLI Aracı (`vault validate`, `explain`, `check-env`) | [ ] Beklemede |
| **v0.9.8** | **Faz 11** | Ekosistem Adaptörleri (Aether, Nyx, Feather vb. için ince köprüler) | [ ] Beklemede |
| **v1.0.0** | **Faz 12** | v1.0 Sertleştirme, Tam Regresyon Matrisi & Kararlılık Sürümü | [ ] Beklemede |

---

## 🛠️ Detaylı Faz Planları ve Görev Kontrol Listeleri

### [x] Sürüm v0.1.0 — Faz 0: Yetenek Denetimi (Capability Audit) ve Paket İskeleti
> **Hedef:** NOX v2.0.123 derleyici yeteneklerini doğrulamak, stdlib imzalarını teyit etmek ve temel paket iskeletini oluşturmak.

- [x] **0.1 Paket İskeleti ve Yapılandırma**
  - [x] `nox.json` oluştur (paket adı: `vault`, sürüm: `0.1.0`, NOX sürümü: `2.0.123`, lisans: MIT).
  - [x] `.gitignore` oluştur (geçici dosyalar, derleme çıktıları, test artifact'leri).
  - [x] `vault.nox` genel cephe (public facade) modülünü oluştur.
  - [x] `README.md` başlangıç dokümanını oluştur.
  - [x] `CHANGELOG.md` oluştur.
  - [x] `.env.example` (yalnızca sahte/dummy değerlerle) hazırla.
- [x] **0.2 NOX Yetenek Probları (`tests/probes/`)**
  - [x] Özel hata sınıfları ve mesajsız hata kodları probu (`probe_errors.nox`).
  - [x] `dict[str, T]`, `list[T]` ve veri kopyalama probu (`probe_collections.nox`).
  - [x] `str | None`, eksik (missing), boş (empty) ve null ayrımı probu (`probe_nullability.nox`).
  - [x] Fonksiyon tipleri ve callback mekanizmaları probu (`probe_callbacks.nox`).
  - [x] `nox.os.getenv` yokluk davranışı (`OsError`) ve `set_var` test probu (`probe_os.nox`).
  - [x] `nox.fs.metadata`, boyut denetimi, `canonicalize` ve symlink probu (`probe_fs.nox`).
  - [x] `nox.json`, `nox.toml` ve `nox.yaml` ayrıştırma ve hata davranış probu (`probe_formats.nox`).
  - [x] String dilimleme (slicing), UTF-8 BOM ve CRLF satır sonu probu (`probe_strings.nox`).
- [x] **0.3 Dokümantasyon & Kabul Kapısı**
  - [x] Probların çıktılarını ve stdlib imzalarını `docs/compatibility.md` dosyasına kaydet.
  - [x] Mimari Karar Kaydı (ADR): `docs/adr/0001-value-model.md` oluştur.
  - [x] İlk test dosyasını oluştur: `tests/package_test.nox` (gerçek bir `assert` içeren).
  - [x] **Kabul Kapısı (Gate 0):** `noxc check vault.nox` ve `noxc test` sıfır hata ile geçecek.

---

### [x] Sürüm v0.2.0 — Faz 1: Saf (Pure) & Bounded `.env` Ayrıştırıcı
> **Hedef:** Hiçbir I/O yapmayan, doğrudan metin alan, güvenlik sınırlarına ve katı kurallara uyan `.env` parser'ı tamamlamak.

- [x] **1.1 Parser Çekirdeği (`env/parser.nox`)**
  - [x] Token ve AST/Satır modellerini tanımla (`EnvEntry`, `EnvParseError`).
  - [x] Değişken adı dilbilgisi denetimi (`^[A-Za-z_][A-Za-z0-9_]*$`).
  - [x] `KEY=VALUE`, eşittir etrafındaki opsiyonel boşluklar, boş değerler (`KEY=`).
  - [x] `export KEY=VALUE` sözdizimini tanı (işletim sistemine aktarmadan güvenle ayrıştır).
  - [x] Tek tırnak (`'...'`) ve çift tırnak (`"..."`) desteği.
  - [x] Kaçış dizileri (Escape sequences): `\n`, `\r`, `\t`, `\\`, `\"` (katı politika ile).
  - [x] Yorum satırları (`#`) ve tırnak içi/satır içi yorum ayrımı.
  - [x] UTF-8 BOM ve CRLF/LF normalizasyonu.
- [x] **1.2 Güvenlik Sınırları & Korumalar**
  - [x] Maksimum dosya boyutu sınırı (1 MiB).
  - [x] Maksimum anahtar sayısı (2.000 anahtar).
  - [x] Maksimum tekil değer boyutu (64 KiB).
  - [x] Maksimum anahtar uzunluğu (128 bayt).
  - [x] Çift anahtar (Duplicate key) katı hata politikası (`CONFIG_DUPLICATE_KEY`).
  - [x] Hata mesajlarında asla sır değeri içermeme, sadece satır/sütun/anahtar adı bildirme.
  - [x] `$(...)`, backtick veya kabuk komutlarını kesinlikle çalıştırmama (hata ver veya literal al).
- [x] **1.3 Testler ve Fikstürler (`tests/env_parser_test.nox`)**
  - [x] Pozitif sözdizimi testleri (`fixtures/dotenv/valid.env`).
  - [x] Tırnak, escape ve yorum testleri (`fixtures/dotenv/quotes.env`).
  - [x] Geçersiz anahtar, bozuk tırnak, NUL baytı testleri (`fixtures/dotenv/malformed.env`).
  - [x] Çift anahtar çakışma testleri.
  - [x] Aşırı boyutlu (size bomb) girdi testleri.
  - [x] Regresyon senaryoları: `ENV-03`, `ENV-04`, `ENV-05`, `ENV-06`, `ENV-07`, `ENV-08`.
  - [x] **Kabul Kapısı (Gate 1):** Tüm parser testleri `noxc test` ile yeşil olmalı, hiçbir ortam değişkeni mutasyona uğramamalıdır.

---

### [x] Sürüm v0.2.5 — Faz 2: Çekirdek Anlık Görüntü (Snapshot), Kaynaklar & Deterministik Öncelik
> **Hedef:** Çoklu kaynakları birleştiren, deterministik öncelik kurallarını işleten ve izlenebilirlik sağlayan çekirdeği kurmak.

- [x] **2.1 Çekirdek Tipler ve Hatalar (`core/types.nox`, `core/errors.nox`)**
  - [x] `ConfigValue`: `String`, `Integer`, `Float`, `Boolean`, `Array`, `Object`, `Null`, `Missing`.
  - [x] `PresenceState`: `Missing`, `NullValue`, `EmptyString`, `SuppliedValue`.
  - [x] `SourceOrigin`: Kaynak türü (`Defaults`, `Memory`, `File`, `Dotenv`, `Env`, `Override`), kaynak ID'si, profil.
  - [x] `ConfigEntry`: Değer, varlık durumu, kaynak bilgisi, sır bayrağı (`is_secret`), öncelik sırası.
  - [x] Güvenli hata tipleri ve standart kodlar (`CONFIG_MISSING`, `CONFIG_DUPLICATE_KEY`, vb.).
- [x] **2.2 Kaynak Soyutlamaları (`sources/interface.nox`, `sources/memory.nox`)**
  - [x] `ConfigSource` arayüzü/sözleşmesi.
  - [x] `MemorySource`: Bellek içi test geçersiz kılmaları ve varsayılanlar.
  - [x] `DotenvSource (`env/source.nox`): Dosya boyutu denetimli `.env` okuyucu.
  - [x] `ProcessEnvSource` (`env/process.nox`): Yalnızca izin verilen/şemada kayıtlı anahtarları `nox.os.getenv` ile okur.
- [x] **2.3 Öncelik ve Birleştirme Motoru (`core/merge.nox`, `core/snapshot.nox`)**
  - [x] Standart öncelik sırası: Defaults -> Base File -> Profile File -> `.env` -> `.env.<profile>` -> Process Env -> Runtime Overrides.
  - [x] Nesneler için derin/özyinelemeli (recursive) birleştirme, diziler için yer değiştirme (replace).
  - [x] Tipler arası çakışmada açık hata verme (`CONFIG_TYPE_MISMATCH`).
  - [x] `ConfigSnapshot`: Dışarıya değişmez (immutable) ve yalnızca okunabilir nesne sunumu.
  - [x] Güvenli kaynak izleme: `snapshot.explain(key)` (asla sır değeri ifşa etmez).
- [x] **2.4 Testler (`tests/merge_test.nox`, `tests/env_process_test.nox`)**
  - [x] Çok kaynaklı öncelik doğrulama testleri (`MERGE-01`).
  - [x] Üst kaynak geçersiz olduğunda alt kaynağa sessizce düşmeme testi (`MERGE-02`).
  - [x] Nesne ve dizi birleştirme testleri (`MERGE-03`).
  - [x] Missing/Null/Empty ayrım testleri (`MERGE-04`, `ENV-01`, `ENV-02`).
  - [x] İzin verilmeyen ortam değişkenlerinin taranmaması testi.
  - [x] **Kabul Kapısı (Gate 2):** Kaynak sıralaması deterministik olmalı, yayınlanan snapshot dondurulmuş (immutable) olmalıdır.

---

### [x] Sürüm v0.3.0 — Faz 3: Tipli Şema (Typed Schema) & Toplu Doğrulama (**MVP DÖNÜM NOKTASI**)
> **Hedef:** Kullanıcıların tipli alanlar tanımlamasını, kısıtları denetlemesini ve toplu, güvenli doğrulama raporları almasını sağlamak.

- [x] **3.1 Şema Alan Tanımları (`schema/fields.nox`)**
  - [x] `FieldType`: String, Int, Float, Bool, StringList, NestedObject.
  - [x] Alan özellikleri: `required`, `nullable`, `non_blank`, `default_value`, `secret`.
  - [x] Sayısal ve metinsel kısıtlar: `min`, `max`, `min_length`, `max_length`, `one_of`.
  - [x] `ConfigSchema` yapısı ve akıcı/net alan ekleme API'si.
- [x] **3.2 Katı Tip Dönüşümü ve Zorlama (`schema/coerce.nox`)**
  - [x] Metinden (`.env`) tam sayıya katı çevrim ve taşma (overflow) denetimi.
  - [x] Mantıksal (boolean) katı çevrim: `true`/`false` (büyük/küçük harf duyarsız).
  - [x] Ondalıklı (float) katı çevrim (NaN ve Infinity reddi).
  - [x] Liste çevrimi (dizi tipleri ve virgülle ayrılmış dizgeler).
- [x] **3.3 Toplu Doğrulama Motoru (`schema/validate.nox`)**
  - [x] İlk hatada durmayan, keşfedilebilir tüm hataları toplayan doğrulayıcı (`ValidationReport`).
  - [x] Hata sıralamasının deterministik olması (kaynak sırası, anahtar, hata kodu).
  - [x] Hatalarda değer içeriğinin gizlenmesi (`SafeConfigDiagnostic`).
- [x] **3.4 Tipli Snapshot Getters (`core/snapshot.nox`)**
  - [x] `get_string(key) -> str`
  - [x] `get_int(key) -> int`
  - [x] `get_float(key) -> float`
  - [x] `get_bool(key) -> bool`
  - [x] `get_list(key) -> list[str]`
  - [x] `has(key) -> bool`, `is_null(key) -> bool`
- [x] **3.5 Testler (`tests/schema_test.nox`, `tests/integration_test.nox`)**
  - [x] Zorunlu alan ve varsayılan değer testleri.
  - [x] Tip uyumsuzluğu ve aralık dışı testleri (`SCHEMA-02`, `SCHEMA-03`, `SCHEMA-04`).
  - [x] Birden fazla hatanın toplu raporlanması testi (`SCHEMA-05`).
  - [x] Üretim modunda eksik sır ayarının fail-closed kapanması (`SCHEMA-01`).
  - [x] **Kabul Kapısı (Gate 3 - MVP Gate):** MVP tamamlanmalı, `tests/integration_test.nox` uçtan uca çalışmalıdır.

---

### [x] Sürüm v0.4.0 — Faz 4: Yapılandırılmış Kaynaklar (JSON, TOML, YAML) & Güvenli Dosya Okuma
> **Hedef:** NOX standart kütüphanesini kullanarak JSON, TOML ve YAML dosyalarını tip kaybı yaşamadan yüklemek ve kök dizin güvenliğini sağlamak.

- [x] **4.1 Kök Dizin ve Dosya Güvenliği (`sources/files.nox`)**
  - [x] `canonicalize` ile yol çözümleme.
  - [x] Yol segmenti bazlı (path-segment aware) kök dizin (`allowed_root`) sınır denetimi.
  - [x] Symlink kaçışı ve `../` dizin geçişi (directory traversal) engelleme (`FILE-01`, `FILE-02`, `FILE-03`).
  - [x] Okuma öncesi dosya boyutu kontrolü (1 MiB sınırı) ve dosya varlık kontrolü (`FILE-04`, `FILE-05`).
- [x] **4.2 Format Köprüleri (`sources/json.nox`, `sources/toml.nox`, `sources/yaml.nox`)**
  - [x] `nox.json.parse` ile JSON -> `ConfigValue` dönüşümü (`json_source_test.nox`).
  - [x] `nox.toml.parse` ile TOML 1.0 -> `ConfigValue` dönüşümü (`toml_source_test.nox`).
  - [x] `nox.yaml.parse` ile YAML alt kümesi -> `ConfigValue` dönüşümü (`yaml_source_test.nox`).
  - [x] Desteklenmeyen veya bozuk formatların güvenli raporlanması (`FORMAT-01`, `FORMAT-02`).
- [x] **4.3 Testler (`tests/security_test.nox`, format testleri)**
  - [x] Dizin dışına sızma girişimleri testi (`/app/config-evil` vs `/app/config`).
  - [x] İçiçe geçmiş (nested) harita ve dizilerin tip korumalı aktarım testleri.
  - [x] **Kabul Kapısı (Gate 4):** Biçim köprüleri NOX stdlib üzerinden çalışmalı ve kök dizin sınırları asla aşılamamalıdır.

---

### [x] Sürüm v0.5.0 — Faz 5: Profiller, İnterpolasyon ve Sır Bulaşması (Taint)
> **Hedef:** Çoklu ortam profillerini (`dev`, `test`, `prod`) katmanlamak, güvenli `${KEY}` referans çözümü ve sır yayılımını gerçekleştirmek.

- [x] **5.1 Profil Yönetimi (`profiles/overlay.nox`)**
  - [x] Açık profil seçimi (`development`, `test`, `staging`, `production`, özel adlar).
  - [x] Profil tabanlı dosya ve `.env` katmanlama (`vault.<profile>.toml`, `.env.<profile>`).
  - [x] Bilinmeyen profil için hata verme (`PROFILE-02`).
  - [x] `production` modunda sıkı güvenlik kontrolleri (`PROFILE-01`).
- [x] **5.2 Güvenli Değişken İnterpolasyonu (`interpolate/resolve.nox`)**
  - [x] Birleştirme sonrasında `${KEY}` sözdizimi çözümü.
  - [x] Döngü tespiti (Cycle detection) ve bağımlılık grafiği (`A -> B -> A`) (`INTERP-02`).
  - [x] Çok adımlı çözüm (`A -> B -> C`) (`INTERP-01`).
  - [x] Maksimum derinlik sınırı (16) ve maksimum genişleme boyutu (128 KiB) (`INTERP-03`).
  - [x] Kabuk komutu, harici kod çalıştırma veya keyfi fonksiyon çağrısı yasağının korunması.
- [x] **5.3 Sır Yayılımı (Secret Taint Propagation)**
  - [x] İnterpolasyonda sır içeren bir anahtar kullanıldığında hedef anahtarın da otomatik `secret=true` olması (`INTERP-04`).
- [x] **5.4 Testler (`tests/profiles_test.nox`, `tests/interpolation_test.nox`)**
  - [x] Zincirleme referans ve döngü testleri.
  - [x] Genişleme bombası (expansion bomb) koruma testleri.
  - [x] Sır lekeleme (taint) testleri.
  - [x] **Kabul Kapısı (Gate 5):** İnterpolasyon deterministik olmalı, döngüler güvenle yakalanmalı ve sırlar lekelenerek korunmalıdır. (LLVM ve QBE onaylı)

---

### [x] Sürüm v0.6.0 — Faz 6: Sır Sınıflandırması, Maskeleme & Güvenli İnceleme (**PRATİK PAKET DÖNÜM NOKTASI**)
> **Hedef:** Sırların hafızada, loglarda, snapshot'larda ve hata çıktılarında asla açığa çıkmamasını garanti etmek.

- [x] **6.1 Sır Modeli ve API (`secrets/mark.nox`, `secrets/reveal.nox`)**
  - [x] `SecretValue` sarmalayıcısı (varsayılan metinsel temsili `[REDACTED]`).
  - [x] Bilinçli ve yetkili sır okuma metodu: `snapshot.get_secret(key) -> str` (`SECRET-03`).
  - [x] Güvenli açıklama metodu: `snapshot.explain(key)` ve `snapshot.describe(key)` (yalnızca kaynak türü, profil ve durum).
- [x] **6.2 Yapılandırılmış Maskeleme (`secrets/redact.nox`)**
  - [x] `snapshot.redacted_json() -> str` üretimi (`SECRET-02`).
  - [x] `redact_string(text, sentinels)` savunma derinliği metin maskelemesi (`SECRET-01`).
  - [x] Sır önbelleği (`secrets/cache.nox`) (TTL destekli, opt-in).
- [x] **6.3 Tehdit Modeli ve Redaksiyon Testleri (`tests/redaction_test.nox`)**
  - [x] `VAULT_TEST_SECRET_DO_NOT_PRINT` sentineliyle tüm çıktıların (stdout, stderr, loglar, hatalar, explain, json) taranması (`SECRET-01`, `CI-03`).
  - [x] Üretimde varsayılan parola kullanımının engellenmesi.
  - [x] **Kabul Kapısı (Gate 6 - Practical Package Gate):** Sentinel hiçbir raporda yer almamalı, yetkili get_secret doğru değeri vermelidir. (LLVM ve QBE onaylı)

---

### [x] Sürüm v0.7.0 — Faz 7: Yerel Sır Sağlayıcıları (Secret Providers)
> **Hedef:** Harici sır depoları için eklenebilir arayüz ve yerel sağlayıcı adaptörlerini sunmak.

- [x] **7.1 Sağlayıcı Sözleşmesi (`providers/interface.nox`)**
  - [x] `SecretProvider` arayüzü (`id`, `get(key)`, `health()`, `close()`).
  - [x] `SecretFetchResult` yapısı.
- [x] **7.2 Yerel Sağlayıcılar (`providers/env.nox`, `providers/file.nox`, `providers/memory.nox`, `providers/source.nox`)**
  - [x] `EnvSecretProvider`: Açıkça izin verilen ortam anahtarları.
  - [x] `FileSecretProvider`: Sınırlandırılmış, kök dizin korumalı sır dosyaları okuyucusu.
  - [x] `MemorySecretProvider`: Test amaçlı sahte sağlayıcı.
  - [x] `SecretProviderSource`: Sağlayıcıların konfigürasyon öncelik sırasına (rank 7) bağlanması.
- [x] **7.3 Hata Davranışları (`PROVIDER-01`, `PROVIDER-03`)**
  - [x] Sağlayıcıya erişilemediğinde fail-closed davranışı (`CONFIG_PROVIDER_UNAVAILABLE`).
  - [x] İzin verilmeyen ağ çağrısı yapılmaması (`PROVIDER-01`).
- [x] **7.4 Testler (`tests/secret_provider_test.nox`)**
  - [x] Yerel dosya, env ve memory sır sağlayıcı testleri.
  - [x] **Kabul Kapısı (Gate 7):** Sır sağlayıcıları izole ve yapılandırılabilir olmalıdır. (LLVM ve QBE onaylı)

---

### [ ] Sürüm v0.8.0 — Faz 8: Doğrulanmış Yeniden Yükleme (Atomic Reload) & Son Bilinen İyi Durum
> **Hedef:** Çalışma anında konfigürasyon yenilenirken sistemin kesintiye uğramamasını ve hatalı konfigürasyonda eski duruma dönülmesini sağlamak.

- [ ] **8.1 Reload Yöneticisi (`reload/manager.nox`)**
  - [ ] Aday (candidate) snapshot oluşturma ve tam doğrulama adımı.
  - [ ] Doğrulama başarısız olursa: Eski snapshot'ı aynen koru, hata raporu dön (`RELOAD-01`).
  - [ ] Doğrulama başarılı olursa: Yeni versiyonlanmış snapshot'ı yayınla (`RELOAD-02`).
  - [ ] Eşzamanlı okuyucular için tutarlılık ve yarım kalmış durum (partial state) engeli.
- [ ] **8.2 Testler (`tests/reload_test.nox`)**
  - [ ] Geçersiz güncelleme sonrası geri alma (rollback) testi.
  - [ ] Başarılı versiyon artışı testi.
  - [ ] **Kabul Kapısı (Gate 8):** Snapshot takası atomik olmalı, yarım kalmış veya geçersiz konfigürasyon asla yayınlanmamalıdır.

---

### [ ] Sürüm v0.9.0 — Faz 9: Opsiyonel Uzak Sır Sağlayıcı Çerçevesi (Remote Providers)
> **Hedef:** Ağ üzerinden çalışan uzak sır sağlayıcıları için güvenli, HTTPS zorunlu, SSRF korumalı altyapı sağlamak.

- [ ] **9.1 Uzak Sağlayıcı Modeli (`providers/remote_http.nox`)**
  - [ ] Yalnızca HTTPS ve açıkça izin verilen alan adı/port (allowlist).
  - [ ] Yeniden yönlendirme (redirect) engelleme ve SSRF savunması (`PROVIDER-02`).
  - [ ] Bounded timeout, sınırlı tekrar deneme (retry) ve arka plana çekilme (backoff).
  - [ ] Ağ çağrılarında veya hatalarda token sızdırmazlığı.
- [ ] **9.2 Testler (Mock Taşıyıcı ile)**
  - [ ] Ağ istekleri yapılmayan hermetik test koşusu (`PROVIDER-01`).
  - [ ] Timeout ve yetkilendirme hatası testleri.
  - [ ] **Kabul Kapısı (Gate 9):** Uzak sağlayıcılar varsayılan suite'te ağ çağrısı yapmamalı, tüm güvenlik sınırlarına uymalıdır.

---

### [ ] Sürüm v0.9.5 — Faz 10: Opsiyonel CLI Aracı (`cli/main.nox`)
> **Hedef:** Geliştiriciler ve CI ortamları için komut satırı arayüzü sunmak.

- [ ] **10.1 CLI Komutları**
  - [ ] `vault validate --config <yol> --profile <ad>` (`CLI-01`).
  - [ ] `vault explain <anahtar> --config <yol>`.
  - [ ] `vault check-env --schema <yol>`.
  - [ ] `vault version` ve `vault --help` (`CLI-02`).
- [ ] **10.2 CLI Güvenliği**
  - [ ] Yardım komutu (`--help`) sır yüklememeli veya ağ açmamalıdır.
  - [ ] Başarısız doğrulamada sıfır olmayan (nonzero) çıkış kodu dönmelidir.
  - [ ] Standart çıktı/hata akışına sır değerleri asla basılmamalıdır.
- [ ] **10.3 Testler**
  - [ ] Komut satırı argümanları ve çıkış kodları testi.
  - [ ] **Kabul Kapısı (Gate 10):** CLI komutları sır sızdırmadan çalışmalı ve CI otomasyonuna uygun olmalıdır.

---

### [ ] Sürüm v0.9.8 — Faz 11: Ekosistem Adaptörleri (Ecosystem Adapters)
> **Hedef:** NOX ekosistemindeki kütüphaneler (Aether, Nyx vb.) için çekirdeği kirletmeyen ince adaptörler sağlamak.

- [ ] **11.1 Adaptör Modülleri (`adapters/`)**
  - [ ] `adapters/aether.nox` (HTTP sunucu ayarları).
  - [ ] `adapters/nyx.nox` (Web servis ayarları).
  - [ ] `adapters/feather.nox` (Veritabanı bağlantı ayarları - parola ifşasız).
  - [ ] `adapters/quartz.nox` (Test fikstür yardımcıları).
- [ ] **11.2 Bağımsızlık İlkesi**
  - [ ] Adaptörlerin varlığı veya yokluğu `import vault` işlemini bozmamalıdır.
- [ ] **11.3 Kabul Kapısı (Gate 11):** Adaptörler opsiyonel olmalı, çekirdeğe bağımlılık taşımamalıdır.

---

### [ ] Sürüm v1.0.0 — Faz 12: v1.0 Sertleştirme, Tam Regresyon ve Yayınlama
> **Hedef:** Tüm güvenlik, platform ve regresyon testlerini tamamlayarak prodüksiyon kalitesinde v1.0.0 sürümünü mühürlemek.

- [ ] **12.1 Tam Regresyon Senaryo Matrisi Doğrulaması**
  - [ ] `ENV-01` .. `ENV-08` (Ortam ve .env senaryoları)
  - [ ] `MERGE-01` .. `MERGE-05` (Öncelik ve birleştirme)
  - [ ] `SCHEMA-01` .. `SCHEMA-05` (Tipli şema ve sınırlar)
  - [ ] `INTERP-01` .. `INTERP-04` (İnterpolasyon ve döngü tespiti)
  - [ ] `SECRET-01` .. `SECRET-04` (Sır maskeleme ve sentineller)
  - [ ] `FILE-01` .. `FILE-05` (Kök dizin ve dosya sınırları)
  - [ ] `FORMAT-01` .. `FORMAT-02` (JSON, TOML, YAML doğruluk)
  - [ ] `PROFILE-01` .. `PROFILE-02` (Profil kontrolleri)
  - [ ] `PROVIDER-01` .. `PROVIDER-04` (Sağlayıcı sınırları)
  - [ ] `RELOAD-01` .. `RELOAD-02` (Yeniden yükleme atomikliği)
  - [ ] `CLI-01` .. `CLI-02` (CLI hata kodları ve güvenlik)
  - [ ] `CI-01` .. `CI-04` (Platform matrisi, backend uyumu ve sır taraması)
- [ ] **12.2 Belgeler ve Örnekler**
  - [ ] `docs/api.md`, `docs/recipes.md`, `docs/security.md`.
  - [ ] `examples/basic_env.nox`, `examples/typed_config.nox`, `examples/secrets.nox`.
- [ ] **12.3 Kod İnceleme ve Güvenlik Denetimi**
  - [ ] Bölüm 20'deki kod inceleme kontrol listesinin (Code Review Checklist) tamamına "EVET".
  - [ ] `noxc check vault.nox` ve `noxc test` tam yeşil raporu.
- [ ] **Kabul Kapısı (Gate 12 - v1.0 Release Gate):** Tüm aşamalar eksiksiz, sızıntısız ve kararlı olmalıdır.

---

## 🔒 Güvenlik & Ajan Uygulama Döngüsü Kuralları

Her görev tamamlandığında aşağıdaki standart rapor formatı kullanılacaktır:

```text
IMPLEMENTED:
- Yapılan değişiklikler, eklenen dosyalar ve fonksiyonlar.

VERIFIED:
- Çalıştırılan derleyici komutu (örn: noxc check vault.nox, noxc test) ve çıktıları.
- Pozitif ve negatif test sonuçları.

SECURITY:
- Sır maskeleme, dizin sınırları, ağ kısıtlamaları denetimleri.

NOT VERIFIED / KNOWN RISKS:
- Henüz test edilmemiş veya desteklenmeyen alanlar.

NEXT STEP:
- Sıradaki görev ve kabul kriteri.
```

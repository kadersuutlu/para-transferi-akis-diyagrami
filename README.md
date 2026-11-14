# Para Transferi Akış Diyagramı

Bir para transferi akışının beş parçaya bölünmüş Mermaid diyagramı ve bu akıştan türetilen 52 test senaryosu. Akış genel bir modeldir; belirli bir bankanın sistemini yansıtmaz. Limit, süre ve deneme sayıları örnek varsayımdır.

Bu çalışmanın anlatıldığı yazı: [Para Transferi Akışını Test Uzmanı Gözüyle Okumak](MEDIUM_LINKI_BURAYA)

## İçindekiler

- [1. Alıcı Seçimi ve Doğrulama](#s)
- [2. Hesap Seçimi ve Transfer Yöntemi Belirleme](#sss)
- [3. Masraf, Bakiye, Limit, Açıklama ve Özet](#s)
- [4. Güvenlik: Risk Kontrolü ve Dijital İmza](#s)
- [5. Rezervasyon, Gönderim, Sonuç ve İade Yönetimi](#ss)
- [Test senaryoları](#test-senaryoları)

Her parçanın çıkış düğümü bir sonrakinin giriş düğümüne bağlanır (Giriş/Çıkış düğümleri yeşil ve mavi).

## 1. Alıcı Seçimi ve Doğrulama

```mermaid
flowchart TB
    A(["Giriş: Ana sayfa"]) -- "Kısayollar" --> C["Para transferi ekranı açılır"]
    A -- "Menü" --> C
    C --> D{"Alıcı kayıtlı mı, yeni mi?"}

    D -- "Kayıtlı alıcı" --> E0{"Kayıtlı alıcı var mı?"}
    E0 -- "Hayır" --> H{"Alıcı nasıl eklenecek?"}
    E0 -- "Evet" --> E["Kayıtlı alıcılar listelenir"]
    E --> F["Kullanıcı alıcıyı seçer"]
    F --> F2{"Kayıtlı hesap hâlâ geçerli mi?"}
    F2 -- "Hayır" --> X_SAVED["Kayıtlı hesap artık geçerli değil"]
    X_SAVED --> H
    F2 -- "Evet" --> G["Alıcı doğrulanır"]

    D -- "Yeni alıcı" --> H

    H -- "IBAN ile" --> I1["IBAN ve alıcı adı girilir"]
    I1 --> I2{"IBAN geçerli mi?"}
    I2 -- "Hayır" --> X_IBAN["IBAN geçersiz"]
    X_IBAN --> I1
    X_IBAN -. "Vazgeçerse" .-> CIKIS_CANCEL
    I2 -- "Evet" --> I3{"Girilen isim IBAN sahibiyle uyuşuyor mu?"}
    I3 -- "Evet" --> G
    I3 -- "Hayır" --> I4["Alıcı adı uyuşmuyor"]
    I4 --> I5{"Yine de devam edilsin mi?"}
    I5 -- "Evet" --> G
    I5 -- "Hayır" --> I1

    H -- "Hesap no ile" --> J1["Banka, şube ve hesap numarası girilir"]
    J1 --> J2{"Hesap bilgileri doğru mu?"}
    J2 -- "Hayır" --> X_ACC["Hesap bilgileri hatalı"]
    X_ACC --> J1
    X_ACC -. "Vazgeçerse" .-> CIKIS_CANCEL
    J2 -- "Evet" --> G

    H -- "QR ile" --> KP{"Kamera izni verilmiş mi?"}
    KP -- "Hayır" --> X_CAM["Kamera izni verilmedi, ayarlardan izin verilmeli ya da başka yöntem seçilmeli"]
    X_CAM --> H
    KP -- "Evet" --> K1["Kamera açılır, QR kod okutulur"]
    K1 --> K2{"QR kod geçerli mi?"}
    K2 -- "Hayır" --> X_QR["QR kod geçersiz"]
    X_QR --> K1
    X_QR -. "Vazgeçerse" .-> CIKIS_CANCEL
    K2 -- "Evet" --> G

    H -- "Kolay Adres ile (KOLAS)" --> L1["Telefon, TCKN veya e-posta girilir"]
    L1 --> L2{"Kolay Adres sistemi sorgulanır"}
    L2 -- "Bulunamadı" --> X_KOLAS["Bu bilgiyle kayıtlı biri bulunamadı"]
    X_KOLAS --> L1
    X_KOLAS -. "Vazgeçerse" .-> CIKIS_CANCEL
    L2 -- "Zaman aşımı" --> X_TIMEOUT["Servis zamanında yanıt vermedi"]
    X_TIMEOUT --> L1
    X_TIMEOUT -. "Vazgeçerse" .-> CIKIS_CANCEL
    L2 -- "Başarılı" --> G

    G --> CIKIS_OK(["Çıkış: Alıcı doğrulandı, Bölüm 2'ye geçilir"])
    CIKIS_CANCEL(["Çıkış: İşlem iptal edildi"])

    classDef giris fill:#e6f4ea,stroke:#188038,color:#202124
    classDef cikis fill:#e8f0fe,stroke:#1a73e8,color:#202124
    class A giris
    class CIKIS_OK,CIKIS_CANCEL cikis

```

## 2. Hesap Seçimi ve Transfer Yöntemi Belirleme

```mermaid
flowchart TB
    GIRIS_A(["Giriş A: Alıcı doğrulandı (Bölüm 1'den)"]) --> M["Gönderen hesap seçilir"]
    GIRIS_B(["Giriş B: Tutar yeniden girilecek (Bölüm 3 ve 5'ten)"]) --> P

    M --> N{"Hesap uygun mu?"}
    N -- "Gönderen ve alıcı hesap aynı" --> X_SELF["Gönderen ve alıcı hesap aynı olamaz"]
    X_SELF --> M
    N -- "Hesap bloke ya da kısıtlı" --> X_FROZEN["Hesap kısıtlı olduğu için işlem yapılamıyor"]
    X_FROZEN --> CIKIS_FAIL
    N -- "Uygun" --> P["Tutar girilir"]

    P --> PV{"Tutar geçerli mi?"}
    PV -- "Hayır" --> X_AMT["Tutar geçersiz: sıfır, negatif, hatalı biçim ya da minimum tutarın altında"]
    X_AMT --> P
    PV -- "Evet" --> OWN{"Alıcı hesap müşterinin kendisine mi ait?"}
    OWN -- "Evet" --> OWN_SUPPORT{"Hesaplar arası transfer yapılabiliyor mu?"}
    OWN_SUPPORT -- "Hayır" --> X_OWN["Bu transfer türü şu an desteklenmiyor"]
    X_OWN --> M
    OWN_SUPPORT -- "Evet" --> INTERNAL["Banka içi hesaplar arası transfer olarak belirlenir"]
    OWN -- "Hayır" --> BANK{"Alıcı aynı bankada mı?"}
    BANK -- "Evet" --> HAVALE["Havale ile gönderilir"]
    BANK -- "Hayır" --> FAST_CHECK{"FAST ile gönderilebilir mi?"}
    FAST_CHECK -- "Evet: TL ve karşı banka uygun" --> FAST["FAST ile gönderilir (7/24)"]
    FAST_CHECK -- "Hayır" --> EFT_CHECK{"EFT ile gönderilebilir mi?"}
    EFT_CHECK -- "Hayır" --> X_CHANNEL["Bu akışta kullanılabilecek bir transfer yöntemi yok"]
    X_CHANNEL --> CIKIS_FAIL
    EFT_CHECK -- "Evet" --> EFT_TIME{"Şu an EFT saatleri içinde mi?"}
    EFT_TIME -- "Evet" --> EFT["EFT ile gönderilir"]
    EFT_TIME -- "Hayır" --> X_EFT_TIME["EFT saatleri dışında, uygun bir zamanda tekrar denenmeli"]
    X_EFT_TIME --> CIKIS_FAIL

    INTERNAL --> CIKIS_OK
    HAVALE --> CIKIS_OK
    FAST --> CIKIS_OK
    EFT --> CIKIS_OK

    CIKIS_OK(["Çıkış: Yöntem belirlendi, Bölüm 3'e geçilir"])
    CIKIS_FAIL(["Çıkış: İşlem tamamlanamadı, kullanıcıya bildirilir"])

    classDef giris fill:#e6f4ea,stroke:#188038,color:#202124
    classDef cikis fill:#e8f0fe,stroke:#1a73e8,color:#202124
    class GIRIS_A,GIRIS_B giris
    class CIKIS_OK,CIKIS_FAIL cikis

```

## 3. Masraf, Bakiye, Limit, Açıklama ve Özet

```mermaid
flowchart TB
    GIRIS_A(["Giriş A: Yöntem belirlendi (Bölüm 2'den)"]) --> FEE["Yönteme göre masraf hesaplanır"]
    GIRIS_B(["Giriş B: Özet yeniden gösterilecek (Bölüm 4'ten)"]) --> T

    FEE --> Q{"Bakiye ve limit yeterli mi?"}

    Q -- "Tutar ve masraf bakiyeyi aşıyor" --> X_BAL["Bakiye yetersiz, tutar değiştirilmeli"]
    X_BAL --> CIKIS_AMOUNT
    X_BAL -. "Vazgeçerse" .-> CIKIS_CANCEL

    Q -- "Yöntem ya da tek işlem limiti aşılıyor" --> X_LIMIT["Tek işlem limiti aşılıyor, tutar değiştirilmeli"]
    X_LIMIT --> CIKIS_AMOUNT
    X_LIMIT -. "Vazgeçerse" .-> CIKIS_CANCEL

    Q -- "Günlük ya da aylık toplam limit aşılıyor" --> X_LIMIT_CUM["Toplam limit aşılıyor, kalan limit gösterilir"]
    X_LIMIT_CUM -- "Tutarı değiştirir" --> CIKIS_AMOUNT
    X_LIMIT_CUM -- "Limit artırmak ister" --> LIM_UP["Limit artırma akışına yönlendirilir, ek doğrulama gerekir; dönüşte bakiye ve limit yeniden kontrol edilir"]
    X_LIMIT_CUM -. "Vazgeçerse" .-> CIKIS_CANCEL

    Q -- "Kullanıcının kendi belirlediği limit aşılıyor" --> X_LIMIT_USER["Kendi belirlediğiniz transfer limiti aşılıyor"]
    X_LIMIT_USER -- "Tutarı değiştirir" --> CIKIS_AMOUNT
    X_LIMIT_USER -- "Limiti güncellemek ister" --> LIM_UP
    X_LIMIT_USER -. "Vazgeçerse" .-> CIKIS_CANCEL

    LIM_UP --> Q
    Q -- "Uygun" --> R["Açıklama girilir"]

    R --> S{"Açıklama uygun mu?"}
    S -- "Hayır" --> X_DESC["Açıklamanın formatı ya da uzunluğu uygun değil"]
    X_DESC --> R
    S -- "Evet" --> T["Özet ekranında yöntem, tutar ve masraf gösterilir"]
    T --> U{"Kullanıcı ne yapıyor?"}
    U -- "Vazgeçti" --> CIKIS_CANCEL
    U -- "Onayladı" --> CIKIS_OK

    CIKIS_OK(["Çıkış: Kullanıcı özeti onayladı, Bölüm 4'e geçilir"])
    CIKIS_AMOUNT(["Çıkış: Tutarı yeniden gir, Bölüm 2'ye (Giriş B) dönülür"])
    CIKIS_CANCEL(["Çıkış: İşlem iptal edildi"])

    classDef giris fill:#e6f4ea,stroke:#188038,color:#202124
    classDef cikis fill:#e8f0fe,stroke:#1a73e8,color:#202124
    class GIRIS_A,GIRIS_B giris
    class CIKIS_OK,CIKIS_AMOUNT,CIKIS_CANCEL cikis

```

## 4. Güvenlik: Risk Kontrolü ve Dijital İmza

```mermaid
flowchart TB
    GIRIS(["Giriş: Kullanıcı özeti onayladı (Bölüm 3'ten)"]) --> OWNT{"Transfer müşterinin kendi hesapları arasında mı?"}
    OWNT -- "Evet" --> SIGN
    OWNT -- "Hayır" --> RISK["Dolandırıcılık ve risk kontrolü yapılır: tutar, alıcı hesap, cihaz, kullanıcı davranışı, konum"]

    RISK --> RISK_R{"Risk sonucu ne?"}
    RISK_R -- "Düşük risk" --> SIGN
    RISK_R -- "Orta risk" --> STEP["Dolandırıcılık uyarısı gösterilir, ek doğrulama istenir (örn. OTP ya da müşteri hizmetleri araması)"]
    STEP --> STEP_R{"Ek doğrulama başarılı mı?"}
    STEP_R -- "Evet" --> SIGN
    STEP_R -- "Hatalı, deneme hakkı var" --> STEP
    STEP_R -- "Hatalı ve deneme hakkı doldu ya da süre doldu" --> X_STEP["Ek doğrulama başarısız, işlem gönderilmedi"]
    X_STEP --> CIKIS_FAIL
    STEP_R -. "Vazgeçerse" .-> CIKIS_CANCEL

    RISK_R -- "Manuel inceleme gerekli" --> HOLD["İşlem incelemeye alınır, para hareketi yapılmaz, kullanıcıya bilgi verilir"]
    HOLD --> REVIEW{"İnceleme sonucu ne?"}
    REVIEW -- "Onaylandı" --> REAUTH{"Kullanıcı güncel özeti yeniden onaylıyor mu? (risk kontrolü tekrar çalıştırılmaz)"}
    REAUTH -- "Evet" --> SIGN
    REAUTH -- "Hayır" --> CIKIS_CANCEL
    REVIEW -- "Reddedildi" --> BLOCK
    REVIEW -. "Sonuç bekleniyor" .-> HOLD
    REVIEW -- "İnceleme süresi doldu" --> X_REV_TO["İnceleme süresi doldu, işlem gönderilmedi, kullanıcıya bilgi verilir"]
    X_REV_TO --> CIKIS_FAIL
    HOLD -. "Kullanıcı vazgeçerse" .-> CIKIS_CANCEL

    RISK_R -- "Yüksek risk" --> BLOCK["İşlem bloke edilir, para hareketi yapılmaz; kural detayı kullanıcıyla paylaşılmaz"]
    BLOCK --> FRAUD_CASE["Fraud vakası açılır, gerekirse hesap ya da kanal geçici kısıtlanır; kullanıcı müşteri hizmetleriyle doğrulama yapabilir"]
    FRAUD_CASE --> CIKIS_BLOCK

    RISK_R -- "Risk servisi yanıt vermedi" --> X_RISK["Güvenlik kontrolü tamamlanamadı, işlem gönderilmez, tekrar denenmesi istenir"]
    X_RISK --> CIKIS_FAIL

    SIGN["İşlem imzalanır: yöntem, alıcı, tutar, masraf, zaman ve tek kullanımlık nonce içeren özet, cihaz anahtarıyla biyometrik ya da PIN onayıyla imzalanır"]
    SIGN --> SIGN_R{"İmza adımının sonucu ne?"}
    SIGN_R -- "İmza verildi" --> VERIFY{"Sunucu imzayı doğrular: cihaz kayıtlı mı, imza geçerli mi, süre dolmamış mı, tekrar kullanım yok mu?"}
    SIGN_R -- "Biyometrik ya da PIN hatalı" --> X_SIGN["İmza doğrulanamadı, tekrar denenmeli"]
    X_SIGN --> SIGN
    SIGN_R -- "Deneme hakkı doldu" --> X_SIGN_LOCK["İmza yöntemi geçici olarak kilitlendi"]
    X_SIGN_LOCK --> CIKIS_FAIL
    SIGN_R -- "Onay süresi doldu" --> X_SIGN_EXP["Onay süresi doldu, özet yeniden gösterilir"]
    X_SIGN_EXP --> CIKIS_SUMMARY
    SIGN_R -- "Vazgeçti" --> CIKIS_CANCEL
    SIGN_R -- "Oturum süresi doldu" --> X_SESSION["Oturum süresi doldu, işlem gönderilmedi, yeniden giriş gerekir"]
    X_SESSION --> CIKIS_FAIL
    VERIFY -- "Geçersiz" --> X_SIGN_INV["İmza doğrulanamadı, işlem gönderilmez; güvenlik kaydı düşülür ve risk motoruna bildirilir"]
    X_SIGN_INV --> CIKIS_FAIL
    VERIFY -- "Geçerli" --> AUDIT["İmzalı işlem kaydı denetim izine yazılır: zaman damgası, cihaz, imza değeri, işlem referansı"]
    AUDIT --> CIKIS_OK

    CIKIS_OK(["Çıkış: İmzalı ve onaylı işlem, Bölüm 5'e geçilir"])
    CIKIS_SUMMARY(["Çıkış: Özeti yeniden göster, Bölüm 3'e (Giriş B) dönülür"])
    CIKIS_BLOCK(["Çıkış: İşlem güvenlik nedeniyle engellendi, kullanıcıya bildirildi"])
    CIKIS_CANCEL(["Çıkış: İşlem iptal edildi"])
    CIKIS_FAIL(["Çıkış: İşlem tamamlanamadı, kullanıcıya bildirilir"])

    classDef giris fill:#e6f4ea,stroke:#188038,color:#202124
    classDef cikis fill:#e8f0fe,stroke:#1a73e8,color:#202124
    class GIRIS giris
    class CIKIS_OK,CIKIS_SUMMARY,CIKIS_BLOCK,CIKIS_CANCEL,CIKIS_FAIL cikis

```

## 5. Rezervasyon, Gönderim, Sonuç ve İade Yönetimi

```mermaid
flowchart TB
    GIRIS(["Giriş A: İmzalı ve onaylı işlem (Bölüm 4'ten)"]) --> DUP{"Aynı işlem referansıyla daha önce başlatılmış bir işlem var mı?"}
    GIRIS_B(["Giriş B: Kullanıcı uygulamaya geri döndü, bağlantı kopmuştu"]) --> QUERY
    DUP -- "Evet" --> QUERY
    DUP -- "Hayır" --> RESERVE["Tutar ve masraf bloke edilir, limit kullanımı rezerve edilir"]
    RESERVE --> RES_R{"Rezervasyon başarılı mı?"}
    RES_R -- "Hayır: bakiye ya da limit bu arada değişti" --> X_RES["Bakiye ya da limit değişti, tutar yeniden belirlenmeli"]
    X_RES --> CIKIS_AMOUNT
    RES_R -- "Evet" --> Y["Transfer, imzalanan işlem bilgileriyle ve tekil işlem referansıyla başlatılır"]

    Y --> Z{"Banka sistemi ve transfer kanalı ne yanıt verdi?"}
    Z -- "Kesin başarılı" --> DONE["Dekont oluşturulur (imza ve işlem referansıyla), limit kullanımı kesinleşir, kullanıcıya bildirim gönderilir, işlem geçmişine yazılır"]
    Z -- "Kesin hata ya da ret" --> EFFECT{"Başarısız işlemde hesaptan para çıkmış mı?"}
    Z -- "Zaman aşımı ya da sonuç belirsiz" --> PEND["İşlem beklemede; bloke tutar ve limit rezervi korunur; tekrar transfer yapılmaz, otomatik iade de yapılmaz; kullanıcıya bilgi verilir"]

    DONE --> SAVE{"Alıcı yeni ve kayıtlı değil mi?"}
    SAVE -- "Evet" --> SAVE_ASK["Alıcıyı kaydetme seçeneği sunulur"]
    SAVE_ASK --> CIKIS_SUCCESS
    SAVE -- "Hayır" --> CIKIS_SUCCESS

    PEND --> QUERY["Aynı işlem referansıyla durum sorgulanır, kayıtlar karşılaştırılır"]
    QUERY --> STATUS{"İşlemin son durumu ne?"}
    STATUS -- "Başarılı" --> DONE
    STATUS -- "Başarısız" --> EFFECT
    STATUS -- "Hâlâ belirsiz" --> WAIT["İşlem beklemede tutulur, gerekirse operasyon ekibine devredilir"]
    STATUS -- "İşlem bulunamadı" --> X_NF["İşlem hiç başlatılmamış, para hareketi yok, kullanıcıdan işlemi yeniden başlatması istenir"]
    X_NF --> CIKIS_FAIL
    WAIT -. "Planlı sorgu ya da yeni bilgi gelince" .-> QUERY

    EFFECT -- "Hesaptan para çıkmamış ya da bloke kalkmış" --> LIM_FREE_FAIL["Rezerve edilen limit serbest bırakılır"]
    LIM_FREE_FAIL --> CIKIS_FAIL
    EFFECT -- "Para çıkmış, ters kayıt gerekiyor" --> ROLL["Ters kayıt atılır, tutar hesaba iade edilir"]
    EFFECT -- "Kayıt ya da bloke durumu belirsiz" --> RECON["Düzeltme bekleniyor; operasyon ekibi kayıtları takip eder"]
    ROLL --> ROLL_RESULT{"İade tamamlandı mı?"}
    ROLL_RESULT -- "Evet" --> LIM_FREE_REFUND["Kullanılan limit geri yüklenir"]
    LIM_FREE_REFUND --> CIKIS_REFUND
    ROLL_RESULT -- "Hayır ya da belirsiz" --> RECON
    RECON -. "Kayıtlar netleşince" .-> RESOLVE{"Mutabakat sonucu ne?"}
    RESOLVE -- "Para hareketi yok ya da bloke kalktı" --> LIM_FREE_FAIL
    RESOLVE -- "İade tamamlandı" --> LIM_FREE_REFUND
    RESOLVE -- "Düzeltme henüz bitmedi" --> RECON

    CIKIS_SUCCESS(["Çıkış: Transfer gerçekleşti, Ana sayfaya dön ekranı açılır"])
    CIKIS_FAIL(["Çıkış: İşlem tamamlanamadı, kullanıcıya bildirilir"])
    CIKIS_REFUND(["Çıkış: Transfer başarısız oldu, tutar hesaba iade edildi ve kullanıcıya bildirildi"])
    CIKIS_AMOUNT(["Çıkış: Tutarı yeniden gir, Bölüm 2'ye (Giriş B) dönülür"])

    classDef giris fill:#e6f4ea,stroke:#188038,color:#202124
    classDef cikis fill:#e8f0fe,stroke:#1a73e8,color:#202124
    class GIRIS,GIRIS_B giris
    class CIKIS_SUCCESS,CIKIS_FAIL,CIKIS_REFUND,CIKIS_AMOUNT cikis

```

## Test senaryoları

### 1. Alıcı Seçimi ve Doğrulama

| Test | Açıklama | Test adımı | Beklenen sonuç |
| --- | --- | --- | --- |
| T1 | Kullanıcı, ana sayfadan para transferi ekranına gider. | 1. Ana sayfa → Para transferi ekranı açılır | Para transferi ekranı açılır |
| T2 | Kullanıcı, kayıtlı bir alıcı seçer. | 2. Kayıtlı alıcı var mı? → Evet → Alıcı seçimi | Alıcı listesi gösterilir, kullanıcı alıcıyı seçer |
| T33 | Kullanıcı, kayıtlı alıcısı olmadığı halde “Kayıtlı alıcı” seçeneğini seçer. | 33. Kayıtlı alıcı var mı? → Hayır → Alıcı nasıl eklenecek? | Alıcı ekleme yöntemi seçimi (IBAN, hesap no, QR, Kolay Adres) gösterilir |
| T3 | Kullanıcı, geçersiz IBAN girer. | 3. IBAN girilir → IBAN geçersiz → Hata mesajı | "IBAN geçersiz" mesajı gösterilir |
| T4 | Kullanıcı, alıcı adı ile IBAN uyuşmuyorsa devam eder. | 4. Alıcı adı uyuşmuyor → "Yine de devam edilsin mi?" → Evet | Alıcı doğrulanır, işlem devam eder |
| T34 | Kullanıcı, hata mesajını gördükten sonra işlemden vazgeçer. | 34. IBAN geçersiz → Vazgeçerse → İşlem iptal edilir | İşlem iptal edilir, para hareketi olmaz |
| T5 | Kullanıcı, kolay adres ile alıcı ekleme yapar. | 5. Kolay Adres → Telefon, TCKN veya e-posta girilir → Sorgulanır | Sistemde kayıtlı bir kullanıcı bulunuyorsa, kullanıcı seçilebilir |
| T43 | Kullanıcı, QR ile ekler ama kamera izni verilmemiştir. | 43. QR ile → Kamera izni verilmiş mi? → Hayır | Kamera izni gerektiği bildirilir, ayarlardan izin verilmesi ya da başka yöntem seçilmesi istenir, alıcı ekleme yöntemi seçimine dönülür |

### 2. Hesap Seçimi ve Transfer Yöntemi Belirleme

| Test | Açıklama | Test adımı | Beklenen sonuç |
| --- | --- | --- | --- |
| T6 | Kullanıcı, aynı hesaptan transfer yapmaya çalışır. | 6. Gönderen ve alıcı hesap aynı → Hata mesajı | "Gönderen ve alıcı hesap aynı olamaz" mesajı gösterilir |
| T7 | Kullanıcı, bloke bir hesaptan transfer yapmaya çalışır. | 7. Hesap bloke → Hata mesajı | "Hesap kısıtlı olduğu için işlem yapılamıyor" mesajı gösterilir |
| T44 | Kullanıcı, geçersiz tutar girer (sıfır, negatif, hatalı biçim ya da minimum tutarın altı). | 44. Tutar girilir → Tutar geçerli mi? → Hayır | Tutar geçersiz bilgisi verilir, tutar girişine dönülür, sonraki adıma geçilmez |
| T45 | Kullanıcı, kendi hesapları arasında transfer yapar. | 45. Alıcı hesap müşterinin kendisine ait → Hesaplar arası transfer yapılabiliyor mu? → Evet | Banka içi hesaplar arası transfer olarak belirlenir |
| T46 | Kullanıcı, kendi hesapları arasında transfer yapar ama bu transfer türü kapalıdır. | 46. Alıcı hesap müşterinin kendisine ait → Hesaplar arası transfer yapılabiliyor mu? → Hayır | "Bu transfer türü şu an desteklenmiyor" mesajı gösterilir, hesap seçimine dönülür |
| T8 | Kullanıcı, aynı banka içinde transfer yapar. | 8. Alıcı müşterinin kendisi değil → Alıcı aynı bankada → Havale | Havale ile gönderilir |
| T9 | Kullanıcı, farklı banka içinde transfer yapar. | 9. Alıcı farklı bankada → FAST ile gönderilebilir mi? → Evet | FAST ile gönderilir |
| T10 | Kullanıcı, farklı bankaya transfer yapar ve FAST uygun değildir. | 10. Alıcı farklı bankada → FAST ile gönderilebilir mi? → Hayır → EFT ile gönderilebilir mi? → Evet → EFT saatleri içinde → Evet | EFT ile gönderilir |
| T47 | Kullanıcı, FAST ve EFT ile gönderilemeyen bir alıcıya transfer yapar. | 47. FAST ile gönderilebilir mi? → Hayır → EFT ile gönderilebilir mi? → Hayır | "Bu akışta kullanılabilecek bir transfer yöntemi yok" mesajı gösterilir, işlem tamamlanmaz |
| T11 | Kullanıcı, EFT saatleri dışında transfer yapmaya çalışır. | 11. EFT saatleri dışında → Hata mesajı | "EFT saatleri dışında, uygun bir zamanda tekrar denenmeli" mesajı gösterilir |

### 3. Masraf, Bakiye, Limit, Açıklama ve Özet

| Test | Açıklama | Test adımı | Beklenen sonuç |
| --- | --- | --- | --- |
| T12 | Kullanıcı, bakiyesi yetersiz tutar girer. | 12. Bakiye yetersiz → Tutar değiştirilmeli | "Bakiye yetersiz, tutar değiştirilmeli" mesajı gösterilir |
| T35 | Kullanıcı, bakiye yetersiz uyarısından sonra işlemden vazgeçer. | 35. Bakiye yetersiz → Vazgeçerse → İşlem iptal edilir | İşlem iptal edilir, para hareketi olmaz |
| T13 | Kullanıcı, tek işlem limitini aşıyor. | 13. Tek işlem limiti aşıldı → Tutar değiştirilmeli | "Tek işlem limiti aşılıyor, tutar değiştirilmeli" mesajı gösterilir |
| T14 | Kullanıcı, günlük limiti aşıyor. | 14. Günlük limit aşıldı → Kullanıcıya kalan limit gösterilir | "Toplam limit aşılıyor, kalan limit gösterilir" mesajı gösterilir |
| T15 | Kullanıcı, limitini artırma istiyor. | 15. Limit artırma istenir → Limit artırma akışına yönlendirilir | Limit artırma akışı başlatılır |
| T16 | Kullanıcı, kendi belirlediği limiti aşıyor. | 16. Kullanıcının kendi belirlediği limit aşıldı → Tutar değiştirilmeli | "Kendi belirlediğiniz transfer limiti aşılıyor" mesajı gösterilir |
| T17 | Kullanıcı, açıklamayı yanlış girer. | 17. Açıklama formatı uygun değil → Hata mesajı | "Açıklamanın formatı ya da uzunluğu uygun değil" mesajı gösterilir |
| T18 | Kullanıcı, açıklamayı doğru girer. | 18. Açıklama uygun → Özet ekranı gösterilir | Özet ekranı gösterilir, kullanıcı onaylar |
| T19 | Kullanıcı, işlemi iptal eder. | 19. İptal eder → İşlem durdurulur | İşlem iptal edilir |

### 4. Güvenlik: Risk Kontrolü ve Dijital İmza

| Test | Açıklama | Test adımı | Beklenen sonuç |
| --- | --- | --- | --- |
| T20 | Kullanıcı, düşük riskli işlem yapar. | 20. Düşük risk → İmza verilir | İmza verilir, işlem devam eder |
| T49 | Kullanıcı, kendi hesapları arasında transfer yapar. | 49. Transfer kendi hesapları arasında → Risk kontrolü atlanır → İmza | Risk kontrolü ve ek doğrulama istenmeden doğrudan imza adımına geçilir |
| T21 | Kullanıcı, orta riskli işlem yapar. | 21. Orta risk → Ek doğrulama istenir | Ek doğrulama istenir |
| T48 | Kullanıcı, orta riskli işlemde ek doğrulamayı yanlış girer ama deneme hakkı vardır. | 48. Orta risk → Ek doğrulama → Hatalı, deneme hakkı var | Ek doğrulama tekrar istenir, işlem gönderilmez |
| T22 | Kullanıcı, ek doğrulama başarısız olur. | 22. Ek doğrulama başarısız → İşlem gönderilmez | "Ek doğrulama başarısız, işlem gönderilmedi" mesajı gösterilir |
| T23 | Kullanıcı, yüksek riskli işlem yapar. | 23. Yüksek risk → İşlem bloke edilir | İşlem bloke edilir, fraud vakası açılır |
| T36 | İşlem manuel incelemeye düşer. | 36. Risk sonucu → Manuel inceleme gerekli → İşlem incelemeye alınır | İşlem incelemeye alınır, para hareketi yapılmaz, kullanıcıya bilgi verilir |
| T37 | İnceleme onaylanır ve kullanıcı güncel özeti yeniden onaylar. | 37. İnceleme sonucu → Onaylandı → Kullanıcı güncel özeti onaylar → İmza adımı | Risk kontrolü tekrar çalışmaz, doğrudan imza adımına geçilir |
| T38 | İnceleme onaylanır ama kullanıcı güncel özeti onaylamaz. | 38. İnceleme sonucu → Onaylandı → Kullanıcı güncel özeti onaylamaz | İşlem iptal edilir, para hareketi olmaz |
| T39 | İnceleme reddedilir. | 39. İnceleme sonucu → Reddedildi → İşlem bloke edilir | İşlem bloke edilir, fraud vakası açılır, kullanıcıya güvenlik nedeniyle engellendi bilgisi verilir |
| T40 | İnceleme süresi dolar. | 40. İnceleme sonucu → İnceleme süresi doldu | “İnceleme süresi doldu, işlem gönderilmedi” bilgisi verilir, işlem gönderilmez |
| T41 | Kullanıcı, inceleme beklerken işlemden vazgeçer. | 41. İşlem incelemede → Kullanıcı vazgeçerse → İşlem iptal edilir | İşlem iptal edilir, para hareketi olmaz |
| T24 | Kullanıcı, imza doğrulanamaz. | 24. İmza doğrulanamaz → Hata mesajı | "İmza doğrulanamadı" mesajı gösterilir |
| T25 | Kullanıcı, imza süresi doldu. | 25. Onay süresi doldu → Özet yeniden gösterilir | Özet yeniden gösterilir |
| T26 | Kullanıcı, imza verilir. | 26. İmza verilir → Sunucu doğrular | Sunucu imzayı doğrular, işlem devam eder |
| T50 | Kullanıcının oturumu imza adımında sona erer. | 50. İmza adımı → Oturum süresi doldu | "Oturum süresi doldu, işlem gönderilmedi" bilgisi verilir, yeniden giriş istenir, para hareketi olmaz |

### 5. Rezervasyon, Gönderim, Sonuç ve İade Yönetimi

| Test | Açıklama | Test adımı | Beklenen sonuç |
| --- | --- | --- | --- |
| T42 | Rezervasyon sırasında bakiye ya da limit değişmiştir. | 42. Rezervasyon → Başarısız → Tutar yeniden belirlenmeli | “Bakiye ya da limit değişti, tutar yeniden belirlenmeli” bilgisi verilir, tutar girişine dönülür |
| T51 | Aynı işlem referansıyla ikinci istek gelir (çift dokunma ya da yeniden deneme). | 51. Aynı işlem referansıyla daha önce başlatılmış işlem var mı? → Evet → Durum sorgulanır | Yeni transfer oluşmaz, mevcut işlemin durumu gösterilir |
| T52 | Gönderim sırasında bağlantı kopar, kullanıcı uygulamaya geri döner. | 52. Uygulamaya geri dönüş → İşlem referansıyla durum sorgulanır | İşlem varsa gerçek durumu gösterilir, yoksa "işlem başlatılmamış, para hareketi yok" bilgisi verilir |
| T27 | Kullanıcı, işlemi başarıyla tamamlar. | 27. Banka sistemi → Kesin başarılı → Dekont oluşturulur | Dekont oluşturulur, kullanıcıya bildirim gönderilir |
| T28 | Kullanıcı, işlemde hata olur. | 28. Banka sistemi → Hata → Para çıkmış mı? | "Para çıkmış, ters kayıt gerekiyor" → Ters kayıt atılır |
| T29 | Kullanıcı, işlemde sonuç belirsiz. | 29. Banka sistemi → Zaman aşımı → İşlem beklemede | İşlem beklemede tutulur, sorgulanır |
| T30 | Kullanıcı, işlemde hata olur. | 30. Banka sistemi → Hata → Para çıkmamış | "Para çıkmamış, limit serbest bırakılır" |
| T31 | Kullanıcı, iade tamamlanmış. | 31. İade tamamlandı → Limit geri yüklenir | Kullanılan limit geri yüklenir |
| T32 | Kullanıcı, iade tamamlanmamış. | 32. İade tamamlanmamış → Düzeltme bekleniyor | Düzeltme bekleniyor, operasyon ekibi takip eder |

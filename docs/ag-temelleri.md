# Ağ temelleri — hiyerarşik çalışma notu

Kablodan HTTP'ye, alttan üste. Her kavram bir alttakinin üstüne oturur; sırayı bozmadan çalış. Hedef: bu kavramları projede (gömülü cihaz, LwIP, PC aracı) kullanabilecek seviye.

## Hiyerarşinin tamamı

```text
KATMAN 5  Uygulama    HTTP, DHCP, DNS, mDNS, DNS-SD, NTP...
                      "ne konuşuyoruz?"
KATMAN 4  Taşıma      TCP, UDP  (+ port kavramı)
                      "hangi uygulamaya, hangi garantiyle?"
KATMAN 3  Ağ          IP, mask, route, gateway, ARP*, ICMP
                      "hangi cihaza, hangi ağ üzerinden?"
KATMAN 2  Veri bağı   Ethernet çerçevesi, MAC adresi, switch
                      "aynı kablodaki hangi komşuya?"
KATMAN 1  Fiziksel    PHY, RJ45, twisted pair, link
                      "elektrik sinyali var mı?"
```

*ARP, 2 ile 3'ü birbirine yapıştırır.

Bir paket gönderilirken üstten alta **sarılır**, alınırken alttan üste **soyulur**:

```text
[Ethernet başlığı [IP başlığı [TCP başlığı [HTTP verisi]]]]
```

Bu tek şemayı ezberle; gerisi bu şemanın detayıdır.

---

# KATMAN 1 — Fiziksel: kablo ve link

**Soru: elektrik sinyali var mı?**

- **Kablo:** bakır twisted pair (Cat5e/Cat6), uçlarda RJ45 konnektör.
- **PHY:** kablodaki analog sinyali dijital bite çeviren çip. Gömülü kartlarda MCU'nun dışında ayrı bir entegredir (ör. LAN8742), MII/RMII hattıyla MCU'ya bağlanır.
- **Link:** iki PHY karşılıklı anlaşınca "link up" olur. Hız (10/100/1000 Mbps) ve duplex **auto-negotiation** ile seçilir.
- **Link LED yanmıyorsa** üst katmanların hiçbirinin anlamı yok. Arıza aramaya her zaman buradan başlanır.

Projede karşılığı: PHY reset pini, 50 MHz RMII saat, link-up kesmesi/callback.

---

# KATMAN 2 — Veri bağı: Ethernet ve MAC

**Soru: aynı ağdaki hangi komşuya?**

## Ethernet çerçevesi (frame)

Ethernet, aynı yerel ağ (LAN) içinde **çerçeve** taşır. Bir hop'luk teslimattır: kartından switch'e, switch'ten hedef karta. Şehirlerarası yol değildir; o iş IP'nin.

```text
| Hedef MAC | Kaynak MAC | EtherType | Veri (46-1500 B) | CRC |
   6 bayt      6 bayt       2 bayt                        4 bayt
```

- **EtherType:** içerideki verinin türü. `0x0800` = IPv4, `0x0806` = ARP.
- **CRC:** bozuk çerçeve donanımda çöpe atılır.
- **MTU = 1500 bayt:** bir çerçevenin taşıyabileceği azami veri. Üst katmanlar buna sığmak zorunda.

## MAC adresi

- 6 bayt, karta gömülü kimlik: `00:80:E1:12:34:56`.
- İlk 3 bayt üreticiyi söyler (OUI), son 3 bayt seri.
- Yalnızca **kendi LAN'inde** anlamlıdır. Router'dan geçen pakette MAC her hop'ta değişir, IP değişmez.

Üç teslimat biçimi:

| Biçim | Hedef MAC | Kim alır |
|-------|-----------|----------|
| Unicast | Cihazın MAC'i | Tek cihaz |
| Broadcast | `FF:FF:FF:FF:FF:FF` | LAN'deki herkes |
| Multicast | `01:00:5E:...` (IPv4 için) | Gruba abone olanlar |

## Switch

Switch, MAC adreslerini öğrenir ve çerçeveyi yalnızca doğru porta yollar. Broadcast'i her porta kopyalar. IP'den habersizdir; katman 2 cihazıdır.

Projede karşılığı: MAC adresini unique ver (aynı MAC'li iki kart aynı ağda kaos çıkarır). Multicast alacaksan (mDNS!) MAC'in multicast filtresini aç; yoksa çerçeve donanımda düşer ve yazılım hiç görmez.

---

# KATMAN 3 — Ağ: IP, mask, route

**Soru: hangi cihaza, hangi ağ üzerinden?**

## IP adresi

32 bitlik mantıksal adres, 4 onluk yazılır: `192.168.1.40`. MAC donanımın kimliği, IP **ağdaki konumu**dur. Cihaz ağ değiştirince IP değişir, MAC değişmez.

Ezber değerler:

| Adres | Anlamı |
|-------|--------|
| `192.168.x.x`, `10.x.x.x`, `172.16-31.x.x` | Özel (private) — internete route edilmez, LAN içi |
| `127.0.0.1` | Loopback — cihazın kendisi |
| `169.254.x.x` | Link-local — DHCP bulunamayınca kendine verdiği adres |
| `255.255.255.255` | Broadcast |
| `224.0.0.0 – 239.x.x.x` | Multicast grupları |

## Mask (alt ağ maskesi)

Maske, IP adresini ikiye böler: **ağ kısmı + cihaz kısmı**.

```text
IP     192.168.1.40
Mask   255.255.255.0   =  /24  (ilk 24 bit ağ)

Ağ     192.168.1.0     ← "mahalle"
Cihaz  .40             ← "kapı numarası"
```

Maskenin tek görevi şu kararı verdirmek:

> Hedef IP benim ağımda mı? (hedef AND mask == benim ağım AND mask?)
>
> - **Evet** → doğrudan gönder (ARP ile MAC'ini bul)
> - **Hayır** → gateway'e (router'a) gönder

`/24` = 254 cihazlık ağ. `/16` = 65534. İki cihaz haberleşemiyorsa ve ping gitmiyorsa ilk bakılacak şey: aynı alt ağda mılar, maskeleri tutarlı mı?

## Route ve gateway

**Routing tablosu**, "şu ağa gitmek için paketi şuraya ver" listesidir. Basit bir cihazda iki satır yeter:

```text
Hedef ağ          Nereye
192.168.1.0/24    doğrudan (kendi LAN'im)
0.0.0.0/0         192.168.1.1'e ver (default gateway)
```

- **Gateway (router):** iki ağı birleştiren, paketi IP'ye bakarak öteki ağa aktaran cihaz.
- **Default gateway (`0.0.0.0/0`):** "başka hiçbir kural uymazsa buraya" satırı. Evdeki modemin `192.168.1.1` olması budur.
- **TTL:** IP başlığındaki sayaç; her router'da 1 azalır, sıfırda paket atılır. Sonsuz döngü koruması.

Paket internete giderken IP başlığındaki adresler sabit kalır ama her hop'ta yeni bir Ethernet çerçevesine sarılır: MAC'ler değişir, IP'ler değişmez. Katman 2 / katman 3 ayrımının özü budur.

## ARP — IP'den MAC'e köprü

Hedef IP'yi bildim; ama çerçeveye **MAC** yazmam gerek. ARP bunu çözer:

```text
Ben:    (broadcast) "192.168.1.40 kimde? MAC'ini söyle"
Cihaz:  (unicast)   "Bende. MAC'im 00:80:E1:12:34:56"
```

Cevap **ARP cache**'e yazılır (dakikalarca geçerli). `arp -a` ile görürsün. "Ping ilk seferde 1 paket kaybediyor" klasiği çoğu zaman ARP çözümlemesinin gecikmesidir.

## ICMP — ağın kontrol kanalı

IP'nin yardımcısıdır; port yok, bağlantı yok.

- `ping` = ICMP Echo Request/Reply → "bu IP'ye erişebiliyor muyum?"
- "Destination unreachable", "TTL exceeded" (traceroute bununla çalışır) gibi hata mesajları.

Projede karşılığı: yeni kartı ağa taktığında test sırası her zaman **link → ping gateway → ping hedef**. Ping çalışmadan TCP debug etme.

---

# KATMAN 4 — Taşıma: port, TCP, UDP

**Soru: o cihazdaki hangi uygulamaya, hangi garantiyle?**

## Port ve soket

IP cihazı bulur; cihazda onlarca uygulama çalışır. **Port** (0–65535), uygulamanın kapı numarasıdır.

- **Soket = IP + port + protokol.** `192.168.1.40:80/TCP` → o cihazdaki web sunucusu.
- TCP portları ve UDP portları **ayrı defterlerdir**; TCP/53 ile UDP/53 farklı kapılar.
- Sunucu iyi bilinen portta **dinler** (listen); istemci rastgele yüksek bir porttan (ephemeral, 49152+) bağlanır.

Ezber portlar: HTTP 80, HTTPS 443, DNS 53, DHCP 67/68, NTP 123, mDNS 5353, MQTT 1883.

## TCP — güvenilir bayt akışı

Telefon görüşmesi: önce bağlantı kur, sonra konuş, sonunda kapat.

**Kurulum — üç yollu el sıkışma:**

```text
İstemci → SYN         "bağlanmak istiyorum"
Sunucu  → SYN+ACK     "kabul, ben de hazırım"
İstemci → ACK         "başladık"
```

**Verdiği garantiler ve mekanizmaları:**

| Garanti | Nasıl |
|---------|-------|
| Kayıp yok | Her veri ACK'lenir; ACK gelmezse yeniden gönderilir |
| Sıra korunur | Segmentler sıra numarası taşır, alıcı dizer |
| Alıcıyı boğmaz | Akış kontrolü (pencere): alıcı "şu kadar alabilirim" der |
| Ağı boğmaz | Tıkanıklık kontrolü: kayıp görünce yavaşlar |

**Kritik model — bayt akışı:** TCP sana paket değil **musluktan akan bayt** verir. İki `send()` çağrısı karşıda tek parça gelebilir, biri bölünebilir. Mesaj sınırını üst katman koyar (HTTP'nin `Content-Length`'i gibi). Kendi TCP protokolünü yazarken ilk kural: mesaj çerçeveleme (uzunluk öneki vb.) senin işin.

Kapanış: `FIN/ACK` (kibarca) veya `RST` (ani kesme).

**TCP güvenilirdir ama şifreli değildir.** Şifreleme TLS'in (HTTPS) işidir.

## UDP — bağlantısız datagram

Kartpostal: yaz, at, umut et.

- El sıkışma yok, ACK yok, yeniden gönderim yok, sıra garantisi yok.
- Bir `send()` = bir datagram; karşıda ya bütün gelir ya hiç gelmez (TCP gibi akışa yapışmaz).
- Başlığı 8 bayt (TCP ~20+); gecikmesi ve maliyeti düşük.
- **Multicast/broadcast yalnız UDP ile olur.** TCP bire-bir bağlantıdır.

| | TCP | UDP |
|--|-----|-----|
| Bağlantı | var | yok |
| Teslimat | garantili | best-effort |
| Sıra | korunur | korunmaz |
| Model | bayt akışı | datagram |
| Multicast | ✗ | ✓ |
| Tipik kullanım | HTTP, dosya, komut-cevap | DNS, DHCP, mDNS, ses/video, sensör yayını |

**Seçim kuralı:** "Her bayt yerine ulaşmalı, sıra önemli" → TCP. "Taze veri eskisinden değerli, kaybolan ölsün" (canlı konum, ölçüm akışı) → UDP. UDP'de güvenilirlik gerekiyorsa uygulama kendi çözer — DNS cevap gelmezse tekrar sorar, o kadar.

---

# KATMAN 5 — Uygulama protokolleri

Artık altyapı tamam: adresim var, taşıma seçebiliyorum. Uygulama protokolleri "içerikte ne var" sorusudur. Aşağıdakilerin ilki (DHCP) aslında adresi **veren** protokol; kronolojik olarak cihaz açılınca ilk o çalışır.

## DHCP — "IP'mi kim verecek?"

Elle IP/mask/gateway yazmak yerine ağdaki DHCP sunucusu (router, sunucu) dağıtır. UDP kullanır: istemci 68, sunucu 67. İstemcinin henüz IP'si olmadığı için **broadcast** ile başlar.

**DORA akışı:**

```text
D  DISCOVER  istemci → herkese : "DHCP sunucu var mı?"
O  OFFER     sunucu  → istemci : "192.168.1.40'ı önereyim"
R  REQUEST   istemci → sunucu  : "onu istiyorum"
A  ACK       sunucu  → istemci : "senindir" + mask + gateway + DNS + lease
```

- **Lease:** adres kiralıktır; süre dolmadan istemci yeniler (renew).
- ACK içinde gelen paket: IP, mask, default gateway, **DNS sunucu adresi**, lease süresi. Yani katman 3 ayarlarının tamamı tek protokolle gelir.
- DHCP bulunamazsa cihaz `169.254.x.x` link-local'a düşebilir: LAN içi konuşur, internete çıkamaz. "Neden 169.254 aldım?" = "DHCP sunucuya ulaşamadım."

DHCP **isim çözmez**, servis ilan etmez. Sadece adres ve ağ ayarı verir.

## DNS — "bu isim hangi IP?"

İnsan `example.com` tutar, paket IP ister. DNS dağıtık bir rehberdir.

Akış: DHCP'nin verdiği DNS sunucusuna (ör. `192.168.1.1` veya `8.8.8.8`) **UDP/53** ile sor:

```text
Soru : example.com A kaydı?
Cevap: 93.184.216.34   (+ TTL: şu kadar saniye önbellekle)
```

Sunucu cevabı bilmiyorsa hiyerarşide yukarı sorar (recursive çözümleme); sen tek soru sorarsın.

**Kayıt türleri** (mDNS/DNS-SD'de de aynen kullanılır — bu tabloyu iyi öğren):

| Tür | Eşleme | Not |
|-----|--------|-----|
| **A** | isim → IPv4 | En temel kayıt |
| **AAAA** | isim → IPv6 | |
| **CNAME** | isim → başka isim | Takma ad |
| **PTR** | ters/liste | IP→isim; DNS-SD'de "bu tipte kimler var" |
| **SRV** | servis → **host + port** | + priority, weight. IP vermez! |
| **TXT** | isim → anahtar=değer | Ek bilgi (yol, sürüm...) |

DNS bir **uygulama protokolüdür**; UDP (büyük cevaplarda TCP) üstünde taşınır.

## mDNS — sunucusuz yerel DNS

Evde/araçta DNS sunucusu yok; ama `yazici.local`'a ismiyle erişmek istiyorum.

**mDNS = aynı DNS soru/cevap formatı, farklı teslimat:**

| | Klasik DNS | mDNS |
|--|-----------|------|
| Hedef | sunucu IP'si (unicast) | multicast **224.0.0.251** |
| Port | 53 | **5353** |
| Alan adı | example.com | **.local** |
| Cevabı veren | DNS sunucusu | **adı sorulan cihazın kendisi** |

```text
Ben (multicast): "yazici.local A kaydı?"
Yazıcı:          "Benim: 192.168.1.40"
```

Her cihaz kendi adının sahibidir ve savunur (isim çakışırsa çözümleme kuralı vardır). Sunucu kurulmaz — buna **zero-configuration** denir.

Projede iki rol: **responder** (kendi adını ilan et) ve **querier** (başkasını bul). Gömülü cihaz genelde en az responder olur.

mDNS **multicast'e dayanır** → katman 2'deki multicast MAC filtresi ve IGMP (gruba katılma) açık olmalı. "HTTP çalışıyor ama cihaz .local'da görünmüyor" arızasının bir numaralı nedeni budur.

## DNS-SD — servis keşfi (mDNS'in üstünde)

mDNS "isim → IP" çözer. Asıl proje ihtiyacı çoğu zaman şudur: **"bu ağda hangi servisler var, hangi host:port'ta?"**

DNS-SD yeni bir protokol değildir; DNS kayıtlarını (PTR+SRV+TXT) bir kurala göre kullanmaktır. Yerel ağda taşıyıcısı mDNS'tir.

```text
1) PTR  _http._tcp.local sorgusu
        → "Mutfak._http._tcp.local"      (bu tipte kimler var → liste)

2) SRV  Mutfak._http._tcp.local
        → port 80, hedef yazici.local    (nereye bağlanılır)

3) TXT  aynı isim
        → path=/admin  ver=2             (ek bilgi)

4) A    yazici.local
        → 192.168.1.40                   (nihayet IP)
```

Akılda tutma: **PTR listeler, SRV yer söyler, TXT detay verir, A çözer.** SRV asla IP vermez; host adı + port verir.

## HTTP — istek/cevap dili

TCP üstünde (port 80; TLS ile 443) metin tabanlı istek/cevap:

```text
İstek:   GET /sensors/temp HTTP/1.1
         Host: cihaz.local

Cevap:   HTTP/1.1 200 OK
         Content-Type: application/json
         Content-Length: 27

         {"temp": 23.5, "unit": "C"}
```

- **Metotlar:** GET (oku), POST (gönder/oluştur), PUT (güncelle), DELETE.
- **Durum kodları:** 200 OK, 404 bulunamadı, 500 sunucu hatası; 2xx başarı, 4xx istemci hatası, 5xx sunucu hatası.
- Mesaj sınırı `Content-Length` (veya chunked) ile çizilir — TCP'nin bayt akışı probleminin HTTP'deki çözümü.
- Gömülü cihazda küçük bir HTTP sunucusu, cihazı hem insana (tarayıcı) hem makineye (REST/JSON) açmanın standart yoludur.

## NTP/SNTP — saat (bonus)

UDP/123. Cihaz saatini ağdaki zaman sunucusuna sorarak kurar. Gömülü sistemde sertifika doğrulama ve log zaman damgası için pratikte şarttır. SNTP, NTP'nin basitleştirilmiş istemci hâlidir.

---

# Hepsi birlikte: bir cihazın hayatı

Kartı ağa taktın, tarayıcıdan `http://cihaz.local/sensors` açtın. Alttan üste her katman sırayla:

```text
 1. PHY        link up (100 Mbps)                        [K1]
 2. DHCP       DORA → 192.168.1.40/24, gw .1, DNS .1     [K5, UDP+broadcast ile]
 3. mDNS       "cihaz.local benim" ilanı                 [K5, UDP multicast]
--- kart hazır; şimdi PC tarafı ---
 4. PC, mDNS   "cihaz.local kim?" → 192.168.1.40         [.local = mDNS, 53 değil 5353]
 5. PC, mask   hedef kendi ağımda → gateway'e gerek yok  [K3 kararı]
 6. PC, ARP    192.168.1.40'ın MAC'i? → 00:80:E1:...     [K2/K3 köprüsü]
 7. TCP        SYN, SYN+ACK, ACK → bağlantı :80          [K4]
 8. HTTP       GET /sensors → 200 OK + JSON              [K5]
```

`google.com` açsaydın yalnız iki şey değişirdi: 4. adım klasik DNS (unicast, port 53) olurdu ve 5. adımda maske "kendi ağımda değil" dediği için çerçeve **gateway'in MAC'ine** giderdi.

---

# Arıza arama sırası (projede altın kural)

Alttan üste; bir katman çalışmadan üstünü debug etme:

1. **Link LED** yanıyor mu? (K1)
2. IP aldım mı — DHCP mi, 169.254 mü? (`ipconfig`/`ifconfig`)
3. **Gateway'e ping** atabiliyor muyum? Hedefe? (K3, ICMP)
4. ARP tablosunda hedef görünüyor mu? (`arp -a`)
5. Port açık mı — TCP bağlantısı kuruluyor mu? (`telnet ip port`, `nc`)
6. Uygulama cevap veriyor mu? (`curl`, tarayıcı)
7. mDNS özelinde: multicast filtresi + IGMP açık mı? (`avahi-browse -art` / `dns-sd -B`)

En değerli alışkanlık: **Wireshark** ile trafiği izle. DORA'yı, ARP'ı, üç yollu el sıkışmayı, mDNS sorgusunu bir kez canlı görmek yüz sayfa okumaya bedeldir.

---

# Sık yapılan kavram hataları

- **"TCP/IP" tek protokol değil.** IP adresler (K3), TCP taşır (K4); ikili söz, ailenin tümünü kastetmek için de kullanılır.
- **MAC ve IP rakip değil.** MAC bir hop'luk teslimat, IP uçtan uca. Her pakette ikisi de var.
- **Mask "güvenlik" değil.** Sadece "yerel mi, gateway'e mi" kararı.
- **TCP güvenilir ≠ şifreli.** Şifre TLS.
- **UDP güvenilmez ≠ kötü.** Bilinçli takas: garanti yok, gecikme düşük.
- **DHCP ≠ DNS.** DHCP "sen bu IP'sin", DNS "bu isim şu IP".
- **mDNS ≠ DNS-SD.** mDNS taşıyıcı (sunucusuz DNS), DNS-SD kural (PTR/SRV/TXT ile servis ilanı).
- **SRV IP vermez.** Host adı + port verir; IP'yi A kaydı verir.
- **`.local` yazıp klasik DNS bekleme.** `.local` mDNS demektir.

---

# Kendini sına

1. İki cihaz aynı switch'te ama maskeleri farklı (`/24` vs `/16`) — ping neden tek yönlü çalışabilir?
2. ARP olmasa IP paketi neden hiç yola çıkamaz?
3. DHCP neden broadcast ile başlamak zorunda?
4. TCP'de iki `send()` çağrısı karşıya neden tek parça gelebilir; bunu kim çözmeli?
5. Canlı sensör akışını TCP ile taşımanın sakıncası ne?
6. `cihaz.local` çözülüyor ama `google.com` çözülmüyor — hangi bileşen arızalı? Tersi olsaydı?
7. SRV kaydı aldın; bağlanmadan önce hangi kayda daha ihtiyacın var?

<details><summary>Cevap anahtarı</summary>

1. Maskesi geniş olan karşıyı "yerel" sanıp ARP yapar; dar olan gateway'e yollar, gateway rotayı bilmiyorsa cevap dönmez.
2. Çerçeveye hedef MAC yazılamaz; katman 2 teslimatı yapılamaz.
3. İstemcinin ne IP'si ne de sunucunun adresi vardır; herkese sormaktan başka yolu yok.
4. TCP bayt akışıdır, mesaj sınırı taşımaz; sınırı uygulama protokolü (uzunluk öneki, Content-Length...) koyar.
5. Kayıpta yeniden gönderim gecikme yaratır; eski ölçüm işe yaramaz — UDP ile taze veri tercih edilir.
6. `.local` mDNS ile çözülüyor, klasik DNS (sunucu adresi/erişim) arızalı. Tersiyse mDNS/multicast tarafı (filtre, IGMP) arızalı.
7. Target host adının A (veya AAAA) kaydı — SRV sana IP vermedi.

</details>

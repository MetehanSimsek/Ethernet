# ITxPT araç ağı: Ethernet protokollerini oturtmak

Bu not, ITxPT (Information Technology for Public Transport) uyumlu bir cihazı **STM32H7** üzerinde geliştirirken DHCP, DNS, mDNS, DNS-SD, SRV, TCP/IP, UDP ve HTTP’nin birbirine nasıl bağlandığını tek bir zihinsel modele oturtmak içindir.

Hedef: her kısaltmayı ezberlemek değil. Araçta bir kablo takıldığında **hangi sorunun hangi protokolle cevaplandığını** görmek.

---

## 1. Tek cümlelik model

Otobüsteki ITxPT ağı, ofisteki bir Ethernet LAN’ıdır. Fark, cihazların birbirini **elle IP yazmadan** bulması ve servislerini **standart kayıtlarla** ilan etmesidir.

Bir STM32H7 modülü ağa girdiğinde sırayla şu soruları çözer:

| Sıra | Soru | Protokol |
|------|------|----------|
| 1 | Fiziksel hat var mı? | Ethernet (PHY + MAC) |
| 2 | Benim IP adresim nedir? | DHCP (yoksa link-local) |
| 3 | Paketleri nasıl taşırım? | IP + TCP veya UDP |
| 4 | Benim adım nedir, karşı tarafın adı nedir? | mDNS (LAN içi DNS) |
| 5 | Bu ağda hangi servisler var, hangi host:port’ta? | DNS-SD (PTR + SRV + TXT) |
| 6 | Envanter / yapılandırma nasıl okunur? | HTTP (TCP üstünde) |
| 7 | GNSS / FMS gibi sürekli veri nasıl yayılır? | UDP multicast |

ITxPT S02 bu yığını **S02P00 Networks and Protocols** altında toplar. Üstüne Inventory, Time, GNSSLocation, FMStoIP gibi servisler biner.

---

## 2. Katmanlar: karışıklığın asıl kaynağı

Protokoller yan yana durmaz. Üst üste durur.

```text
  HTTP          DNS-SD / mDNS         GNSS XML yayını
  (uygulama)    (uygulama)            (uygulama)
       |              |                     |
      TCP            UDP                   UDP
       |              |                     |
       +-------+------+----------+----------+
               |                 |
              IP (+ ICMP, IGMP)
               |
           Ethernet
        (MAC + PHY + kablo)
```

Okuma kuralı:

- **HTTP, TCP değildir.** HTTP, TCP’nin üstünde konuşulan istek/cevap dilidir.
- **mDNS, DNS-SD değildir.** mDNS, LAN’de DNS sorusunu herkese bağırarak sorma yöntemidir. DNS-SD, o sorunun *servis keşfi* için nasıl yazılacağıdır.
- **SRV, ayrı bir kablo protokolü değildir.** DNS (veya mDNS) içindeki bir **kayıt türüdür**.

STM32H7 tarafında bu katmanların hepsi pratikte **LwIP** içinde yaşar. Ethernet MAC silikonunuzdadır; PHY genelde harici bir çiptir (LAN8742A vb.).

---

## 3. Ethernet: kablo, MAC, PHY

ITxPT’nin fiziksel omurgası CEN/TS 13149-8’dir: araç içi kablolu Ethernet (10/100/1000BASE-T). Wi‑Fi, ITxPT servis omurgası olarak önerilmez.

STM32H7’de iki ayrı parça vardır:

- **MAC:** çip içindeki Ethernet denetleyicisi. Çerçeve (frame) gönderir/alır.
- **PHY:** bakır hatta elektrik sinyali. CubeMX’te RMII/MII ile bağlanır.

ITxPT için MAC adresiniz Inventory kaydında da görünür. Ağda kimlik, hem MAC hem hostname hem IP ile bağlanır.

Multicast (mDNS, GNSS yayını) çalışacaksa MAC’in multicast çerçeveleri **filtreleyip düşürmemesi** gerekir. STM32H7’de bu, hash tablosu / `ETH->MACPFR` multicast biti ve LwIP `NETIF_FLAG_IGMP` ile ilgilidir. Unicast HTTP çalışıp mDNS’in sessiz kalmasının en sık nedeni budur.

---

## 4. IP: paketlerin adresi

IP, “bu paket kime gitsin?” sorusunu çözer. ITxPT birincil ağ **özel IPv4** kullanır (RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`). IPv6 desteklenebilir; pratikte birçok filo hâlâ IPv4’tür.

Üç teslimat biçimi:

| Biçim | Anlamı | ITxPT örneği |
|-------|--------|----------------|
| Unicast | Bir cihaza | HTTP Inventory, TCP socket |
| Broadcast | Alt ağdaki herkese | DHCP keşfi |
| Multicast | Abone olanlara | mDNS (`224.0.0.251`), GNSS/FMS veri yayını |

Multicast’e abone olmak için **IGMP** gerekir. LwIP’de `LWIP_IGMP=1` olmadan mDNS ve UDP multicast düzgün çalışmaz.

---

## 5. TCP ve UDP: iki taşıma tarzı

İkisi de IP’nin üstündedir. Fark, teslimat sözleşmesidir.

### TCP

- Bağlantı kurulur (üç yollu el sıkışma).
- Sıra, yeniden iletim, akış kontrolü vardır.
- Kayıp paket uygulamaya görünmeden onarılır.
- ITxPT’te: **HTTP**, `_itxpt_http._tcp`, `_itxpt_socket._tcp`.

STM32H7/LwIP: `tcp_new()`, `tcp_bind()`, `tcp_listen()` veya HTTP sunucu uygulaması.

### UDP

- Bağlantı yoktur. Datagram atılır.
- Teslimat, sıra, tekrar gönderim yoktur.
- Gecikme düşüktür; kayıp göze alınır veya uygulama katmanı halleder.
- ITxPT’te: **mDNS** (UDP/5353), **GNSSLocation / FMStoIP multicast**, SNTP.

STM32H7/LwIP: `udp_new()`, `udp_bind()`, `udp_recv()`, `igmp_joingroup()`.

Seçim kuralı (CEN/TS 13149-7 ile uyumlu):

- Envanter, yapılandırma, istek/cevap → TCP/HTTP.
- Yüksek sıklıklı konum/şasi verisi, bir-çok yayın → UDP multicast.
- Zaman → SNTP/NTP (UDP).

TCP “kayıtlı mektup”, UDP “kartpostal”dır: kartpostal daha hızlıdır, kaybolabilir.

---

## 6. DHCP: “Benim IP’m nedir?”

DHCP (Dynamic Host Configuration Protocol) bir **istemci-sunucu** protokolüdür. STM32H7 neredeyse her zaman **DHCP istemcisi**dir. Araçta DHCP sunucusu genelde VCG (Vehicle Communication Gateway) veya ağ anahtarındaki servistir.

Klasik IPv4 diyalog:

```text
STM32H7  --DHCP DISCOVER (broadcast)-->  herkes
Sunucu   --DHCP OFFER------------------>  STM32H7
STM32H7  --DHCP REQUEST---------------->  sunucu
Sunucu   --DHCP ACK-------------------->  STM32H7
         (IP, maske, ağ geçidi, lease süresi)
```

CEN/TS 13149-7 otomatik adresi tercih eder. DHCP yoksa IPv4 **link-local** (`169.254.0.0/16`, RFC 3927) devreye girebilir. Bu, “adres aldım” demektir; ITxPT uyumluluğu için filonun DHCP’si olması beklenir.

LwIP:

```c
#define LWIP_DHCP 1
dhcp_start(netif);   /* link up olduktan sonra */
```

DHCP, hostname vermez (verebilir ama ITxPT keşfi buna dayanmaz). Ad ve servis keşfi **mDNS / DNS-SD** işidir.

---

## 7. DNS ve mDNS: “Bu isim hangi IP?”

### Klasik DNS

Ofiste bir DNS sunucusu vardır. İstemci `OnBoardUnit01.filo.local` sorar, sunucu `192.168.10.41` döner. Araçta böyle merkezi bir sunucu **olmak zorunda değildir**.

### mDNS (RFC 6762)

Aynı DNS kayıt türleri kullanılır, ama soru **unicast sunucuya gitmez**. UDP multicast ile LAN’e bağırılır:

- Adres: `224.0.0.251`
- Port: `5353`
- Alan adı: `.local`

Örnek:

```text
Soru : OnBoardUnit01.local A kaydı nedir?
Cevap: 192.168.10.41   (cihazın kendisi cevaplar)
```

Bu yüzden mDNS’e “sunucusuz DNS” denir. Her cihaz kendi adını savunur.

ITxPT/CEN tarafında modül adı genelde **sınıf + iki haneli indeks**tir: `OnBoardUnit01`, `FrontDisplay02`.

LwIP:

```c
#define LWIP_MDNS_RESPONDER 1
mdns_resp_init();
mdns_resp_add_netif(netif, "OnBoardUnit01", 3600);
```

mDNS hem **yanıtçı** (kendi adını ilan etmek) hem **sorgulayıcı** (başkasını bulmak) olabilir. Bir ITxPT üretici cihazı en azından yanıtçı olmak zorundadır; tüketen cihaz (AVMS, bilet makinesi) sorgulayıcıdır.

---

## 8. DNS-SD ve SRV: “Hangi servis nerede?”

Burada çoğu kişi takılır. Ayrım:

| Kavram | Ne işe yarar |
|--------|----------------|
| DNS | İsim → IP |
| mDNS | Bunu LAN’de sunucusuz yapmak |
| DNS-SD (RFC 6763) | İsimlerin **servis ilanı** için nasıl düzenleneceği |
| SRV kaydı | Bu servis örneği **hangi host ve port** |
| PTR kaydı | Bu servis tipinin **örnek listesi** (browse) |
| TXT kaydı | Ek anahtar=değer bilgisi (yol, sürüm, multicast grup) |
| A / AAAA | Hostname → IPv4 / IPv6 |

DNS-SD, yeni bir kablo protokolü değildir. DNS kayıtlarını belirli bir isimlendirme kuralıyla kullanır.

### İsim ağacı

```text
<örnek>.<servis>.<alan>

Örnek ITxPT:
  VCG._inventory._itxpt_http._tcp.local

  örnek   = VCG
  servis  = _inventory._itxpt_http._tcp
  alan    = local
```

ITxPT resmi keşif tipleri (Linden / S02, `lsinv` aracı ile uyumlu):

- `_itxpt_http._tcp` — HTTP tabanlı servisler (Inventory başta)
- `_itxpt_socket._tcp` — ham TCP soket servisleri

Inventory XML örneğinde servis taşıma adları şöyle geçer: `itxpt_http`, `itxpt_multicast`, `sntp`.

### Üç kayıt birlikte

Bir istemci GNSS veya Inventory ararken kabaca:

```text
1) PTR  _itxpt_http._tcp.local
        →  VCG._inventory._itxpt_http._tcp.local
        →  (başka örnekler...)

2) SRV  VCG._inventory._itxpt_http._tcp.local
        →  öncelik, ağırlık, port, hedef host
        örn.  0 0 8080 OnBoardUnit01.local

3) TXT  aynı isim
        →  txtvers=1  path=/inventory  ...

4) A    OnBoardUnit01.local
        →  192.168.10.41
```

SRV alanları (RFC 2782):

| Alan | Anlamı |
|------|--------|
| Priority | Düşük sayı önce seçilir |
| Weight | Aynı öncelikte yük dağılımı |
| Port | TCP/UDP portu |
| Target | Hostname (IP değil) |

IP’yi SRV vermez. SRV host adı verir; A kaydı IP’yi verir. Bu yüzden mDNS yanıtında SRV + A sıkça **aynı pakette** gelir.

### TXT

DNS-SD TXT, noktalı virgülle bitişik bir blob değil; `anahtar=değer` çiftleridir. ITxPT Inventory keşfinde hostname, path, sürüm gibi bilgiler burada taşınır. Asıl envanter gövdesi TXT’te değildir; TXT sizi HTTP kaynağına götürür.

LwIP mDNS yanıtçısında servis eklemek:

```c
mdns_resp_add_service(netif,
                      "VCG",                 /* instance */
                      "_itxpt_http",         /* service  */
                      DNSSD_PROTO_TCP,       /* _tcp     */
                      80,
                      3600,
                      txt_callback,
                      userdata);
```

Gerçek ITxPT instance adında `_inventory` gibi alt tip bilgisi de bulunur; tarama araçları (`avahi-browse _itxpt_http._tcp`) buna göre süzgeçler.

---

## 9. HTTP: keşiften sonraki konuşma

HTTP, TCP üzerinde istek/cevap protokolüdür.

```text
İstemci: GET /inventory HTTP/1.1
         Host: OnBoardUnit01.local

Sunucu:  HTTP/1.1 200 OK
         Content-Type: application/xml
         ... XML gövde ...
```

ITxPT S02P01 Inventory, cihazın kendisini ve sunduğu servisleri XML (veya JSON, sürüme göre) ile tarif eder. Resmi örnek (`ModulesDelivery.xml`) şunları listeler:

- donanım: üretici, model, seri no, MAC, yazılım sürümü
- durum bitleri
- alt modüller (FMS arayüzü, GPS alıcısı)
- servisler: `inventory/itxpt_http`, `time/sntp`, `gnsslocation/itxpt_multicast`, `fmstoip/itxpt_multicast`, `fmstoip/itxpt_http`

Yani DNS-SD “kapı numarası”dır; HTTP Inventory “içindeki oda planı”dır.

CEN/TS 13149-11 (Vehicle Platform) benzer bir kalıp kullanır: yapılandırma HTTP/TCP, veri yayını UDP multicast. ITxPT FMStoIP / VEHICLEtoIP aynı fikirdir.

STM32H7’de LwIP `httpd` veya kendi TCP sunucunuz yeter. XML üretmek bellek pahalıdır; şablonu flash’ta tutup alanları doldurmak yaygındır.

S02P00 örnekleri ayrıca HTTP üzerinden **Subscribe / Unsubscribe** mesajları gösterir: istemci “şu UDP grubuna yayın yap” diye sunucuya HTTP ile abone olur, sonra multicast dinler. Keşif → HTTP yapılandırma → UDP akış, ITxPT’nin tekrar eden ritmidir.

---

## 10. Hepsi bir açılışta: STM32H7 ITxPT modülü

Aşağıdaki sıra, kavramları tek hikâyede birleştirir. Sayısal adres/port örnekleri öğreticidir; filonuzun S02P00/S02P0x belgesindeki değerler esas alınır.

```text
 t=0    PHY link up (100 Mbps full duplex)
 t=1    DHCP: 192.168.10.41/24 al
 t=2    mDNS: "OnBoardUnit01.local" ilan et
 t=3    DNS-SD: _itxpt_http._tcp üzerinde Inventory SRV+TXT
 t=4    TCP/80 HTTP sunucu: GET /inventory → XML
 t=5    (üretici ise) UDP multicast GNSS XML yayınla
        (tüketici ise) mDNS browse + IGMP join + UDP recv
```

Karşı cihaz (örneğin yolcu bilgi ekranı) GNSS isterse:

1. mDNS ile `_itxpt_http._tcp` tarar, Inventory’yi bulur.
2. HTTP GET ile hangi servislerin `itxpt_multicast` olduğunu okur.
3. GNSSLocation kaydındaki grup/port’a IGMP join yapar.
4. UDP datagramlarını parse eder (`GNSSLocationDelivery.xml` şeması).

Digi gibi VCG uygulamalarında GNSS yayını sıklıkla `239.255.42.21:14005` civarındadır. Bunu ezberlemek yerine **Inventory + DNS-SD TXT/SRV**’den okumak doğrudur; ITxPT’nin amacı da budur: sabit IP/port yazmamak.

---

## 11. ITxPT servisleri bu yığının neresinde?

| Servis | S02 | Keşif / taşıma | STM32H7’de tipik rol |
|--------|-----|----------------|----------------------|
| Inventory | S02P01 | DNS-SD + HTTP | Her modül sunar |
| Time | S02P02 | SNTP (UDP) | İstemci; VCG sunucu |
| GNSSLocation | S02P03 | HTTP yapılandır + UDP multicast | Üretici veya tüketici |
| FMStoIP | S02P04 | HTTP + UDP multicast | CAN/FMS geçidi |
| VEHICLEtoIP | S02P05 | HTTP + UDP multicast | Araç verisi |
| AVMS | S02P06 | HTTP / websocket | Operasyonel durum |
| APC | S02P07 | HTTP / olay | Yolcu sayımı |
| MADT | S02P08 | HTTP | Sürücü terminali |
| MQTT broker | S02P09 / S04 | TCP 1883 + DNS-SD | Yeni nesil omurga |

S04 ile ITxPT, DNS-SD + HTTP/UDP modelinin yanına MQTT tabanlı **Active Inventory** ekler. STM32H7 projesinde önce S02 yığınını (DHCP → mDNS → HTTP Inventory) oturtmak, MQTT’ye geçmeden önceki doğru sıradır.

---

## 12. STM32H7 / LwIP kontrol listesi

CubeMX + LwIP (H7 paketindeki LwIP 2.0.x yeter; mDNS responder vardır):

| LwIP anahtarı | Neden |
|---------------|--------|
| `LWIP_DHCP` | IP almak |
| `LWIP_IGMP` | Multicast grup |
| `LWIP_MULTICAST_TX_OPTIONS` | IGMP ile birlikte |
| `LWIP_MDNS_RESPONDER` | Ad ve servis ilanı |
| `LWIP_NUM_NETIF_CLIENT_DATA` | mDNS’in netif’e bağlanması |
| `MEMP_NUM_IGMP_GROUP` | mDNS + uygulama grupları, 1’den büyük |
| `LWIP_NETIF_STATUS_CALLBACK` | link/DHCP sonrası mDNS announce |
| `LWIP_UDP` / `LWIP_TCP` | mDNS ve HTTP |

Donanım:

- RMII saat ve PHY reset doğru mu?
- MAC adresi unique mi?
- RX DMA multicast çerçeveleri alıyor mu?
- `netif->flags |= NETIF_FLAG_IGMP` set mi?

Yazılım sırası:

1. `netif_add` + `netif_set_up`
2. `dhcp_start` → adres callback
3. `mdns_resp_init` + `mdns_resp_add_netif`
4. `mdns_resp_add_service` (Inventory)
5. HTTP dinleyici
6. İhtiyaç varsa `igmp_joingroup` + UDP

PC’den doğrulama (Linux):

```bash
avahi-browse -art
avahi-browse -vrtp _itxpt_http._tcp
curl http://192.168.10.41/inventory
```

ITxPT’nin kendi tarayıcısı: [ITxPT/lsinv](https://github.com/ITxPT/lsinv) (`_itxpt_http._tcp` ve `_itxpt_socket._tcp` içinde `_inventory`).

---

## 13. Sık karıştırılanlar

**“DNS-SD ayrı bir daemon mı?”**  
Hayır. DNS-SD, mDNS (veya unicast DNS) üzerinde bir isimlendirme kuralıdır. LwIP’de ekstra bir “DNS-SD protokol makinesi” yoktur; mDNS responder’a servis eklersiniz.

**“SRV IP verir.”**  
Vermez. SRV host + port verir. IP için A kaydı gerekir.

**“mDNS sadece isim çözümler, servis çözmez.”**  
mDNS her DNS kaydını taşıyabilir. Servis keşfi DNS-SD’nin PTR/SRV/TXT düzenidir; taşıyıcı yine mDNS’tir.

**“HTTP UDP üstünde de olur.”**  
Standart ITxPT Inventory HTTP’si TCP üzerindedir. HTTP/3 (QUIC) araç ağında kullanılmaz.

**“DHCP hostname ilanıdır.”**  
DHCP adres verir. ITxPT keşfi mDNS hostname + DNS-SD servis kaydıdır.

**“Multicast = broadcast.”**  
Broadcast alt ağdaki herkese gider. Multicast yalnızca IGMP ile katılanlara. GNSS’i herkese flood etmek yerine gruba üye cihazlar dinler.

---

## 14. Kavramı oturtma soruları

Cevapları yazmadan önce kendi cümlelerinle cevapla:

1. Bir cihazın IP’si varken neden hâlâ mDNS gerekir?
2. PTR, SRV, TXT, A kayıtlarından hangisi “port numarasını” taşır?
3. Inventory XML’i neden TXT kaydına gömülmez?
4. GNSS neden HTTP GET ile sürekli çekilmez de UDP multicast yayınlanır?
5. STM32H7’de HTTP çalışıyor, `avahi-browse` boş dönüyor. İlk bakacağın üç yer neresi?
6. `_itxpt_http._tcp` ile `_itxpt_socket._tcp` arasındaki fark nedir?

Kısa cevap anahtarı:

1. IP değişebilir (DHCP lease); diğerleri sizi isim ve servis tipiyle bulur. Ayrıca “bu IP’de hangi uygulama var?” DHCP’de yoktur.
2. SRV.
3. TXT küçüktür (~1300 bayt pratik sınır, mDNS paketi). Envanter büyüktür ve HTTP ile sürümlenir.
4. 1 Hz civarı konum verisini her tüketicinin poll etmesi ağı ve sunucuyu boğar; multicast bir kez üretir, N dinler.
5. IGMP flag, MAC multicast filtresi, `LWIP_MDNS_RESPONDER` / announce’ın DHCP callback’ten sonra yapılması.
6. HTTP uygulama protokolü vs ham TCP; ikisi de Inventory keşfinde kullanılabilir (`lsinv -w` / `-s`).

---

## 15. Resmi kaynaklar

- ITxPT belge merkezi: S00, S01, **S02P00 Networks and Protocols**, S02P01–P09
- Örnek XML/XSD: [github.com/ITxPT/S02](https://github.com/ITxPT/S02) (Linden dalı)
- Mimari çerçeve: CEN/TS 13149-7 (ağ), 13149-8 (fiziksel), 13149-9/10/11 (time/GNSS/vehicle)
- RFC 2131 DHCP, RFC 6762 mDNS, RFC 6763 DNS-SD, RFC 2782 SRV, RFC 2616/9110 HTTP

S02 metninin bağlayıcı kopyası Documentation Centre’dadır. Bu not eğitim modelidir; sertifikasyon için resmi S02P00/P01 maddeleri esas alınır.

---

## 16. Sonraki pratik adım (STM32H7)

Kavram oturduktan sonra kod sırası:

1. Link + DHCP ile ping.
2. `OnBoardUnit01.local` mDNS A kaydı (`avahi-resolve -n`).
3. `_http._tcp` veya `_itxpt_http._tcp` SRV ilanı.
4. HTTP `/inventory` XML.
5. Bir UDP multicast yayın veya join.

Her adımı bir öncekin üzerine ekle. ITxPT “hepsini birden aç” protokolü değildir; üst katman, alt katmanın çalıştığını varsayar.

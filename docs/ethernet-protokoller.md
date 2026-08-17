# Ethernet protokolleri — çalışma notu

Amaç: TCP/IP, UDP, DHCP, DNS, mDNS (ve yanına HTTP, DNS-SD, SRV) kavramlarını **kendi başına** oturtmak. Ezber listesi değil; her protokolün **hangi soruyu** cevapladığını görmek.

---

## 0. Büyük resim

Bir ağda veri gitmesi için üç şey gerekir:

1. **Fiziksel yol** — kablo, RJ45, elektrik sinyali (Ethernet)
2. **Adres** — paket kime gidecek (IP)
3. **Anlaşma** — nasıl taşınacak, nasıl bulunacak (TCP/UDP, DHCP, DNS, …)

Protokoller yan yana değil, **üst üste** durur:

```text
  HTTP, DNS, DHCP, mDNS     ← uygulama (ne konuşuyoruz)
           |
     TCP        UDP         ← taşıma (nasıl teslim)
           |
          IP                ← adresleme (kime)
           |
       Ethernet             ← çerçeve + kablo (hangi hat)
```

Okuma kuralı: üstteki, alttakinin üstünde çalışır. HTTP, TCP değildir. DNS, IP değildir.

---

## 1. Ethernet

**Ne işe yarar?** Aynı yerel ağdaki (LAN) cihazların kablo üzerinden çerçeve (frame) alışverişi.

Ethernet **IP değildir**. Sadece bir hop’luk komşuya teslimat yapar. Evin içindeki koridor gibidir; şehirler arası yol IP’dir.

### Temel parçalar

| Parça | Ne |
|-------|-----|
| **PHY** | Bakır hatta elektrik. Link LED’i burada yanar. |
| **MAC** | Çerçeveyi oluşturur, hata kontrolü (CRC) yapar. |
| **MAC adresi** | 6 bayt donanım adresi, örn. `00:80:E1:12:34:56`. Dünya genelinde unique olması beklenir. |
| **Çerçeve** | Hedef MAC, kaynak MAC, tip (ör. IPv4), veri, CRC. |

İki cihaz aynı switch’e bağlıysa Ethernet yeter. Farklı ağlara (router arkası) geçmek için IP gerekir.

**Unicast / broadcast / multicast** (Ethernet ve IP’de de var):

- Unicast: bir cihaza
- Broadcast: bu LAN’deki herkese (`FF:FF:FF:FF:FF:FF`)
- Multicast: abone olan gruba

---

## 2. TCP/IP nedir?

Günlük dilde “TCP/IP” iki anlama gelir:

1. **Yığın (suite):** Ethernet üstünden internetin çalıştığı protokol ailesi (IP, TCP, UDP, ICMP, DHCP, DNS, …)
2. **Dar anlam:** taşıma olarak TCP + adres olarak IP

Çalışma notunda “TCP/IP yığını” = tüm aile. TCP ve IP ayrı protokoller.

### Katmanlar (pratik model)

| Katman | Soru | Örnek |
|--------|------|--------|
| Uygulama | Ne konuşuyoruz? | HTTP, DNS, DHCP, mDNS |
| Taşıma | Bağlantılı mı, datagram mı? Port? | TCP, UDP |
| İnternet | Hangi IP’ye? | IP, ICMP, IGMP |
| Ağ erişimi | Hangi MAC / kablo? | Ethernet |

Her katman bir **başlık (header)** ekler. Gönderirken dıştan sarılır, alırken soyulur.

```text
[Eth][IP][TCP][HTTP isteği]     gönderilen çerçeve
```

---

## 3. IP (Internet Protocol)

**Soru:** Bu paket **hangi cihaza** gitsin?

IP, uçtan uca adreslemedir. Router’lar paketi IP’ye bakarak ilerletir. Ethernet her hop’ta değişebilir (yeni MAC); IP adresi kaynak ve hedef olarak (NAT hariç) aynı kalır.

### IPv4 adres

32 bit, dört onluk: `192.168.1.40`

Yanında **alt ağ maskesi** vardır: `255.255.255.0` yani `/24`. Maske, “bu adresler benim LAN’im, gerisi router’a” ayrımını yapar.

Özel (private) aralıklar — internette route edilmez, ev/ofis/araç içi kullanılır:

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

`127.0.0.1` = kendi kendine (loopback).  
`169.254.0.0/16` = link-local; DHCP yokken cihazın kendi verdiği adres.

### Paket (datagram)

IP paketi: kaynak IP, hedef IP, TTL (kaç hop yaşasın), protokol alanı (içerde TCP mi UDP mi), veri.

IP **teslimat garantisi vermez**. Kaybolabilir, çoğalabilir, sıra bozulabilir. Garantiyi isteyen TCP üstüne biner.

### ICMP

IP’nin yardımcı protokolü. `ping` = ICMP Echo. “Karşı taraf ulaşılabilir mi?” İçinde TCP/UDP yok.

---

## 4. Port nedir?

IP cihazı bulur. Bir cihazda birçok uygulama çalışır. **Port**, o cihazın içindeki kapı numarasıdır.

- 16 bit: 0–65535
- TCP portları ile UDP portları **ayrı defterlerdir**. TCP/80 ile UDP/80 farklı şeylerdir.

İyi bilinenler:

| Port | Protokol |
|------|----------|
| 80 | HTTP (TCP) |
| 443 | HTTPS (TCP) |
| 53 | DNS (çoğunlukla UDP) |
| 67/68 | DHCP (UDP) |
| 5353 | mDNS (UDP) |

Soket = `IP + port + (TCP veya UDP)`.  
Örnek: `192.168.1.40:80` TCP = o makinedeki HTTP sunucusu.

---

## 5. TCP (Transmission Control Protocol)

**Soru:** Veriyi **güvenilir, sıralı, bağlantılı** nasıl iletirim?

TCP, IP’nin üstünde bir taşıma protokolüdür. İki uç önce bağlantı kurar, sonra bayt akışı gönderir.

### Ne sağlar?

- **Bağlantı:** konuşmadan önce el sıkışma
- **Güvenilirlik:** kayıp paket yeniden gönderilir (ACK)
- **Sıra:** uygulama veriyi gönderildiği sırada görür
- **Akış kontrolü:** karşı tarafın tamponunu boğmama
- **Tıkanıklık kontrolü:** ağı boğmama

Dosya, web sayfası, komut-cevap için varsayılan seçim TCP’dir.

### Üç yollu el sıkışma (kurulum)

```text
İstemci → SYN        “konuşalım”
Sunucu  → SYN-ACK    “tamam, ben de hazırım”
İstemci → ACK        “başlayalım”
```

Kapanış FIN/ACK ile olur. RST ani kesmedir.

### Akış modeli

TCP uygulamaya “paket” değil **bayt akışı** sunar. Bir `send()` bir TCP segmenti olmak zorunda değildir. Karşı tarafta mesaj sınırını sen koyarsın (HTTP’de başlık + uzunluk, vs.).

### Ne zaman kullanılmaz?

Canlı konum, ses, “kaybolursa bir sonrakini gönderirim” verisi. Yeniden iletim gecikme üretir. O iş UDP’nindir.

---

## 6. UDP (User Datagram Protocol)

**Soru:** Küçük bir mesajı **hızlı ve bağlantısız** nasıl atarım?

UDP, IP’ye neredeyse sadece **port** ekler. El sıkışma yok, ACK yok, yeniden iletim yok, sıra yok.

Uygulama bir datagram gönderir. Gider veya gitmez. Giden parça bütün kalır (IP parçalanması ayrı konu); TCP’deki gibi akışa yapışmaz.

### Ne zaman?

- DNS sorgusu (kısa, kaybolursa tekrar sorulur)
- DHCP
- mDNS
- Video/ses, oyun, sensor yayını
- Multicast (TCP multicast yapmaz; TCP bire-bir bağlantıdır)

### TCP ile kısa karşılaştırma

| | TCP | UDP |
|--|-----|-----|
| Bağlantı | Var | Yok |
| Teslimat | Garantili | Best-effort |
| Sıra | Korunur | Korunmaz |
| Hız / gecikme | Daha ağır | Daha hafif |
| Multicast | Yok | Var |
| Örnek | HTTP, SSH | DNS, DHCP, mDNS |

---

## 7. DHCP (Dynamic Host Configuration Protocol)

**Soru:** Bu ağa yeni girdim, **IP adresim ne olacak?**

Elle `192.168.1.50` yazmak yerine bir sunucu (router, sunucu, araçta VCG) adres dağıtır.

UDP kullanır: istemci port **68**, sunucu port **67**. İlk anda istemcinin IP’si yoktur; bu yüzden **broadcast** gider.

### DORA sırası

```text
D  DISCOVER   istemci herkese: “bir sunucu var mı?”
O  OFFER      sunucu: “sana 192.168.1.40 önereyim”
R  REQUEST    istemci: “bu adresi istiyorum”
A  ACK        sunucu: “senindir; maske, gateway, DNS, süre (lease)”
```

Lease bitince yenilenir (renew). Cihaz kapanırken RELEASE gönderebilir.

DHCP’nin verdiği tipik bilgiler:

- IP adresi
- Alt ağ maskesi
- Varsayılan ağ geçidi (router)
- DNS sunucu adresi
- Lease süresi

**DHCP isim çözmez.** Sadece “sen bu IP’sin” der. `google.com` hâlâ DNS işidir. Hostname ilanı da asıl olarak DNS/mDNS işidir.

DHCP yoksa birçok cihaz `169.254.x.x` alır: aynı kabloda link-local konuşulur, internete çıkılmaz.

---

## 8. DNS (Domain Name System)

**Soru:** `www.example.com` **hangi IP?**

İnsan isim tutar, paket IP ister. DNS, dağıtık bir telefon rehberidir.

### Klasik unicast DNS

1. Bilgisayarın elinde DNS sunucu IP’si vardır (çoğu zaman DHCP vermiştir, örn. `8.8.8.8` veya `192.168.1.1`).
2. UDP/53 ile soru gider: “`www.example.com` için A kaydı?”
3. Cevap: `93.184.216.34`
4. Bundan sonra HTTP o IP’ye TCP ile gider.

Cevap büyükse veya bölge aktarımı varsa TCP/53 de kullanılır. Günlük sorgu UDP’dir.

### Kayıt türleri (sonra mDNS/DNS-SD’de de aynılar)

| Tür | Anlamı |
|-----|--------|
| **A** | İsim → IPv4 |
| **AAAA** | İsim → IPv6 |
| **PTR** | Ters yön veya “şu listedekiler” (DNS-SD browse) |
| **SRV** | Servis → host + port (+ öncelik, ağırlık) |
| **TXT** | Serbest metin / `anahtar=değer` |
| **CNAME** | Bu isim başka bir ismin takma adı |

### Önemli ayrım

DNS **uygulama protokolüdür**, UDP (veya TCP) üstünde çalışır. “DNS, TCP/IP’nin katmanı” değil; yığının en üstündedir.

İsim yoksa da IP ile bağlanırsın. DNS konfor ve yönetim içindir, kablonun kendisi değil.

---

## 9. mDNS (Multicast DNS)

**Soru:** Bu **yerel ağda** `yazici.local` kim? Ortada DNS sunucusu yok.

RFC 6762. Klasik DNS ile **aynı soru/cevap formatı**, farklı teslimat:

| | DNS | mDNS |
|--|-----|------|
| Nereye | Bilinen bir sunucu IP’si (unicast) | Multicast `224.0.0.251` |
| Port | 53 | **5353** |
| Alan | `example.com` vb. | `.local` |
| Kim cevaplar | DNS sunucusu | Adı sorulan cihazın kendisi |

```text
Herkese (UDP multicast):  "OnBoardUnit01.local A kaydı nedir?"
Cihaz kendisi:            "Benim, 192.168.1.40"
```

Buna **zero-configuration** denir: sunucu kurmadan LAN içinde isim.

Gereksinimler:

- Cihaz multicast alabilmeli (Ethernet filtresi + IP tarafında IGMP)
- İsim çakışırsa mDNS savunma/yeniden adlandırma vardır

mDNS **interneti çözmez**. `google.com` hâlâ klasik DNS’tir. mDNS ev, ofis, araç içi LAN içindir.

---

## 10. DNS-SD ve SRV (DNS’in servis keşfi)

DNS isim→IP çözer. Asıl ihtiyaç bazen şudur: “Bu LAN’de **yazıcı / web arayüzü / inventory** var mı, **hangi port**?”

**DNS-SD** (RFC 6763) yeni bir kablo protokolü değildir. DNS (veya mDNS) kayıtlarını servis ilanı için düzenleme kuralıdır.

Taşıyıcı çoğu LAN’de **mDNS**’tir. Bu yüzden “mDNS = servis keşfi” denir; tam doğru değil. mDNS taşıyıcı, DNS-SD kuraldır.

### Üç kayıt birlikte

İsim yapısı: `<örnek>.<servis>._tcp.local`  
Örnek: `Mutfak._http._tcp.local`

```text
1) PTR   _http._tcp.local
         → Mutfak._http._tcp.local     “http servisleri şunlar”

2) SRV   Mutfak._http._tcp.local
         → öncelik, ağırlık, port 80, hedef yazici.local

3) TXT   aynı isim
         → path=/, vers=1, ...

4) A     yazici.local
         → 192.168.1.40
```

### SRV kaydı

SRV, “şu servis **şu makinenin şu portunda**” der. Alanları:

- **Priority** — küçük sayı önce
- **Weight** — aynı öncelikte pay
- **Port** — 16 bit kapı
- **Target** — hostname (IP değil)

IP’yi SRV vermez. Host adı verir; A/AAAA IP’yi verir.

PTR = listele (browse). SRV = nereye bağlan. TXT = ek bilgi.

---

## 11. HTTP (kısa, çünkü TCP’nin üstünde yaşar)

**Soru:** Bir kaynaktan belge / API nasıl **isterim**?

HTTP uygulama protokolüdür. Neredeyse her zaman **TCP** üstündedir (80 veya 443).

```text
İstemci: GET /index.html HTTP/1.1
         Host: www.example.com

Sunucu:  HTTP/1.1 200 OK
         Content-Type: text/html
         (gövde)
```

Yığın: Ethernet → IP → TCP → HTTP.

Tarayıcı sırası pratik tekrardır:

1. DHCP ile IP al (zaten alınmış olabilir)
2. DNS ile ismi IP’ye çevir
3. TCP ile 80/443’e bağlan
4. HTTP isteğini yaz

---

## 12. Bir hikâyede hepsi

Ev router’ına laptop taktın, tarayıcıya `http://yazici.local` yazdın.

1. **Ethernet** — link up, MAC’ler konuşur.
2. **DHCP** — laptop `192.168.1.50` alır; router `192.168.1.1`.
3. İsim `.local` ile bitiyor → **klasik DNS’e değil mDNS’e** sorulur (`224.0.0.251:5353`).
4. Yazıcı **mDNS** ile “ben `yazici.local` = `192.168.1.40`” der.
5. Laptop **TCP** ile `192.168.1.40:80` bağlantısı açar.
6. **HTTP** `GET /` gider, sayfa gelir.

Aynı anda `google.com` açsaydın adım 3–4 **unicast DNS** (port 53, router veya 8.8.8.8) olurdu; mDNS devreye girmezdi.

---

## 13. Sık karışanlar

- **TCP/IP = tek protokol değil.** IP adresler, TCP taşır.
- **TCP güvenli (encrypted) demek değil.** Güvenilir teslimat demek. Şifre TLS/HTTPS işidir.
- **UDP “güvenilmez = işe yaramaz” değil.** Bilerek hafif. Uygulama isterse kendi tekrarını yapar (DNS tekrar sorar).
- **DHCP, DNS değildir.** DHCP “sen bu IP’sin”. DNS “bu isim bu IP”.
- **mDNS, DNS’in rakibi değil.** Aynı dil, LAN içi multicast teslimat. `.local` ↔ mDNS.
- **mDNS ≠ DNS-SD.** mDNS isim sorar. DNS-SD servis listeler (PTR/SRV/TXT).
- **SRV, IP vermez.** Host + port verir.
- **HTTP, TCP’nin başka adı değil.** TCP boru, HTTP o borudaki dil.

---

## 14. Mini sözlük

| Terim | Tek cümle |
|-------|-----------|
| Çerçeve | Ethernet’in paketi |
| Paket / datagram | IP’nin (ve UDP’nin) birimi |
| Segment | TCP’nin birimi |
| Header | Katmanın başa eklediği kontrol bilgisi |
| Unicast | Tek hedef |
| Broadcast | Yerel ağdaki herkes |
| Multicast | Gruba üye olanlar |
| Lease | DHCP’nin adresi kiralama süresi |
| Query / response | DNS soru ve cevap |
| Handshake | TCP bağlantı kurulumu |
| Socket | IP + port + protokol üçlüsü |

---

## 15. Kendine sor

1. IP varken Ethernet MAC neden hâlâ lazım?
2. Ping (ICMP) neden “TCP bağlantısı” değildir?
3. DHCP neden UDP ve broadcast kullanır?
4. `ornek.local` neden port 53’teki DNS sunucusuna gitmez?
5. SRV kaydı ile A kaydı arasındaki fark nedir?
6. HTTP’yi UDP üstüne koysan ne bozulur?

Kısa anahtar:  
1) Bir sonraki hop’a teslimat MAC ile olur; router her hop’ta MAC’i değiştirir.  
2) ICMP, IP’nin yanında kontrol mesajıdır; port/bağlantı yok.  
3) Henüz IP’si yoktur; sunucunun adresini de bilmez.  
4) `.local` mDNS’tir (5353, multicast).  
5) A = isim→IP; SRV = servis→host+port.  
6) Mesaj sınırı, kayıp, sıra: tarayıcı/sunucu her şeyi kendisi tamir etmek zorunda kalır; HTTP bu varsayımla yazılmamıştır.

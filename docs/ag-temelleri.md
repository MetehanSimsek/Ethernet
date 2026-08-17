# Ağ Temelleri ve Ethernet Mimarisi: Kapsamlı Çalışma ve Başvuru Kılavuzu

> **Hedef:** Bu doküman; Ethernet ve bilgisayar ağları temellerini en alt fiziksel katmandan en üst uygulama katmanına kadar hiyerarşik bir bütünlük içinde anlatır. Gömülü sistemler (STM32, ESP32, LwIP), Linux tabanlı cihazlar ve PC araçları (Wireshark, `ip`, `curl`, `netcat`) üzerinde çalışan mühendislerin sahada ve geliştirmede doğrudan uygulayabileceği derinlikte hazırlanmıştır.

---

## İçindekiler

1. [Büyük Resim: Katman Hiyerarşisi ve Çalışma Mantığı](#1-büyük-resim-katman-hiyerarşisi-ve-çalışma-mantığı)
   - [Neden Katmanlı Mimari?](#neden-katmanlı-mimari)
   - [5 Katmanlı Model ve Protokol Dağılımı](#5-katmanlı-model-ve-protokol-dağılımı)
   - [Enkapsülasyon ve Dekapsülasyon (Sarma ve Soyma)](#enkapsülasyon-ve-dekapsülasyon-sarma-ve-soyma)
   - [PDU, MTU ve Paket Boyutları](#pdu-mtu-ve-paket-boyutları)
2. [Katman Katman Derinlemesine İnceleme](#2-katman-katman-derinlemesine-inceleme)
   - [Katman 1 — Fiziksel Katman (PHY / Link)](#katman-1--fiziksel-katman-phy--link)
   - [Katman 2 — Veri Bağı Katmanı (Ethernet / MAC / Switch / Multicast)](#katman-2--veri-bağı-katmanı-ethernet--mac--switch--multicast)
   - [Katman 3 — Ağ Katmanı (IP / Mask / Routing / Gateway / ARP / ICMP)](#katman-3--ağ-katmanı-ip--mask--routing--gateway--arp--icmp)
   - [Katman 4 — Taşıma Katmanı (Port / Socket / TCP / UDP)](#katman-4--taşıma-katmanı-port--socket--tcp--udp)
   - [Katman 5 — Uygulama Katmanı (DHCP / DNS / mDNS / DNS-SD / HTTP / NTP)](#katman-5--uygulama-katmanı-dhcp--dns--mdns--dns-sd--http--ntp)
3. [Uçtan Uca Yaşam Döngüsü (Power-On'dan HTTP İsteğine)](#3-uçtan-uca-yaşam-döngüsü-power-ondan-http-isteğine)
4. [Projelerde Katman Katman Arıza Arama Sırası (Troubleshooting)](#4-projelerde-katman-katman-arıza-arama-sırası-troubleshooting)
   - [Sistematik Hata Ayıklama Sırası](#sistematik-hata-ayıklama-sırası)
   - [Kullanılacak Araçlar ve Komutlar](#kullanılacak-araçlar-ve-komutlar)
   - [Gömülü Sistemler (LwIP / RTOS) Özel Kontrol Listesi](#gömülü-sistemler-lwip--rtos-özel-kontrol-listesi)
5. [Sık Yapılan Kavram Yanılgıları ve Doğruları](#5-sık-yapılan-kavram-yanılgıları-ve-doğruları)
6. [Pekiştirme ve Sınama Soruları (Cevap Anahtarlı)](#6-pekiştirme-ve-sınama-soruları-cevap-anahtarlı)
   - [Bölüm A: Kavramsal ve Protokol Soruları](#bölüm-a-kavramsal-ve-protokol-soruları)
   - [Bölüm B: Saha ve Arıza Teşhis Senaryoları](#bölüm-b-saha-ve-arıza-teşhis-senaryoları)
   - [Ayrıntılı Cevap Anahtarı](#ayrıntılı-cevap-anahtarı)

---

# 1. Büyük Resim: Katman Hiyerarşisi ve Çalışma Mantığı

## Neden Katmanlı Mimari?

Bir bilgisayar ağında iki cihazın birbiriyle konuşması; kablodaki mikrovolt seviyesindeki elektrik geriliminden, web tarayıcısında parse edilen JSON verisine kadar uzanan devasa bir operasyon zinciridir. Bu zinciri tek bir parça (monolitik) olarak tasarlamak imkansızdır.

Ağ mühendisliğinde **katmanlı mimari (layered architecture)** şu temel ilkeye dayanır:
- **Görev Ayrımı (Separation of Concerns):** Her katman yalnızca kendi altındaki katmanın sunduğu hizmeti kullanır ve yalnızca kendi üstündeki katmana bir hizmet sunar.
- **Soyutlama (Abstraction):** Katman 5'teki HTTP sunucusu, verinin Cat6 bakır kabloyla mı, fiber optikle mi yoksa Wi-Fi ile mi taşındığını bilmek zorunda değildir. Katman 1'deki PHY çipi de taşınan verinin HTML mi yoksa sensör telemetrisi mi olduğuyla ilgilenmez.

## 5 Katmanlı Model ve Protokol Dağılımı

İnternet ve endüstriyel Ethernet sistemleri pratikte **TCP/IP 5 Katmanlı Hibrit Modeli** ile modellenir:

```text
+------------------------------------------------------------------------------------+
| KATMAN 5: UYGULAMA (Application)                                                   |
| Protokoller: HTTP, REST, DHCP, DNS, mDNS, DNS-SD, NTP, MQTT, CoAP, SSH             |
| Soru: "Ne konuşuyoruz? Hangi servisi çalıştırıyoruz ve veri ne anlama geliyor?"   |
+------------------------------------------------------------------------------------+
| KATMAN 4: TAŞIMA (Transport)                                                       |
| Protokoller: TCP, UDP (+ Port ve Soket kavramları)                                 |
| Soru: "Cihazdaki hangi uygulamaya/porta, hangi güvenilirlik modeliyle gideceğiz?"  |
+------------------------------------------------------------------------------------+
| KATMAN 3: AĞ (Network / Internetwork)                                              |
| Protokoller: IPv4, IPv6, ICMP, ARP*, IGMP, Routing (Yönlendirme), Gateway         |
| Soru: "Hangi hedef IP adresine, yerel ağda mı yoksa router üzerinden mi?"        |
+------------------------------------------------------------------------------------+
| KATMAN 2: VERİ BAĞI (Data Link / Ethernet)                                         |
| Protokoller: Ethernet II Frame, MAC Adresleme, Switch, VLAN (802.1Q)               |
| Soru: "Aynı fiziksel kablodaki/LAN'deki hangi donanım komşusuna (MAC) ileteceğiz?" |
+------------------------------------------------------------------------------------+
| KATMAN 1: FİZİKSEL (Physical / PHY)                                                |
| Donanım: Bakır Kablo (Cat5e/6, T1), RJ45/M12, PHY Çipi, MII/RMII, Sinyal, Link     |
| Soru: "Elektrik/optik sinyal var mı, saat sinyali oturmuş mu, bitler akıyor mu?"   |
+------------------------------------------------------------------------------------+
```

*\* Not: ARP protokolü Katman 3 mantıksal IP adresleri ile Katman 2 fiziksel MAC adresleri arasında bir köprü görevi görür.*

---

## Enkapsülasyon ve Dekapsülasyon (Sarma ve Soyma)

Ağda veri transferi, bir hediye kutusunu iç içe geçmiş daha büyük kutulara koymaya (ve varış noktasında sırayla açmaya) benzer:

1. **Enkapsülasyon (Gönderici Tarafı - Üstten Alta Sarma):**
   - **Uygulama:** HTTP GET isteği metnini hazırlar (`Data`).
   - **Taşıma:** Verinin başına TCP başlığını ekler (`[TCP [Data]]` -> **Segment**).
   - **Ağ:** Segmentin başına IP başlığını ekler (`[IP [TCP [Data]]]` -> **Paket**).
   - **Veri Bağı:** Paketin başına Ethernet başlığını (MAC'ler, EtherType), sonuna da CRC hata kontrolünü ekler (`[ETH [IP [TCP [Data]]] CRC]` -> **Çerçeve / Frame**).
   - **Fiziksel:** Çerçeve PHY tarafından voltaj darbelerine (bit akışına) dönüştürülüp kabloya basılır.

2. **Dekapsülasyon (Alıcı Tarafı - Alttan Üste Soyma):**
   - Alıcı PHY sinyali dijital bitlere çevirir (K1).
   - Ethernet MAC donanımı CRC'yi doğrular; hedef MAC kendi MAC'i, broadcast veya kayıtlı multicast ise çerçeveyi kabul edip Ethernet başlığını soyar (K2).
   - IP yığını hedef IP'yi ve IP Checksum'ı kontrol eder; IP başlığını soyar (K3).
   - TCP yığını hedef portu doğrular, sıra numarasını (Sequence) işler; TCP başlığını soyar (K4).
   - Uygulama saf HTTP verisini alır ve işler (K5).

```text
[ GÖNDERİCİ ]                                                          [ ALICI ]
  Uygulama Katmanı: [ HTTP Verisi ]                                      Uygulama
         | (Enkapsüle et: TCP Başlığı ekle)                                 ^
  Taşıma Katmanı:   [ TCP Başlığı | HTTP Verisi ]                           | (Soy: TCP)
         | (Enkapsüle et: IP Başlığı ekle)                                  |
  Ağ Katmanı:       [ IP Başlığı | TCP | HTTP Verisi ]                      | (Soy: IP)
         | (Enkapsüle et: Ethernet Başlığı + CRC ekle)                      |
  Veri Bağı:        [ ETH Başlığı | IP | TCP | HTTP Verisi | CRC32 ]        | (Soy: ETH)
         |                                                                  |
  Fiziksel:         010110010110101011000101011100010101011101010110001010101 (Kablo)
```

---

## PDU, MTU ve Paket Boyutları

Her katmanda verinin aldığı isim (**PDU - Protocol Data Unit**) farklıdır:

| Katman | PDU Adı | Tipik Başlık Boyutu | Maksimum Boyut |
| :--- | :--- | :--- | :--- |
| **Katman 5 (Uygulama)** | Data / Mesaj | Protokole bağlı | İsteğe bağlı |
| **Katman 4 (Taşıma)** | Segment (TCP) / Datagram (UDP) | TCP: 20–60 bayt, UDP: 8 bayt | MSS (Tipik 1460 bayt) |
| **Katman 3 (Ağ)** | Paket (Packet) | IPv4: 20–60 bayt | MTU (Tipik 1500 bayt) |
| **Katman 2 (Veri Bağı)** | Çerçeve (Frame) | 14 bayt başlık + 4 bayt CRC | 1518 bayt (VLAN ile 1522 B) |
| **Katman 1 (Fiziksel)** | Bit / Sembol | Preamble (7B) + SFD (1B) | Hat hızına göre sürekli akış |

- **MTU (Maximum Transmission Unit):** Katman 2 Ethernet'in taşıyabileceği azami Katman 3 yüküdür. Standart Ethernet için **MTU = 1500 bayttır**.
- **MSS (Maximum Segment Size):** Bir TCP segmentinin taşıyabileceği azami saf veri miktarıdır.
  $$\text{MSS} = \text{MTU} - (\text{IP Başlığı (20 B)} + \text{TCP Başlığı (20 B)}) = 1500 - 40 = 1460 \text{ bayt}$$

---

# 2. Katman Katman Derinlemesine İnceleme

---

## Katman 1 — Fiziksel Katman (PHY / Link)

### 1. Hangi Soruyu Çözer?
> *"İki cihaz arasında elektriksel/optik temas var mı, saat sinyali oturmuş mu ve 0/1 bitleri bozulmadan akıyor mu?"*

Fiziksel katman, dijital dünyadaki mantıksal bitleri (0 ve 1) analog fiziksel ortama (bakır kablodaki diferansiyel gerilimler, fiberdeki ışık darbeleri) aktarır ve karşı taraftan geri toplar.

### 2. Nasıl Çalışır?

#### Donanım Mimarisi
Gömülü bir Ethernet sisteminde fiziksel bağlantı zinciri şu şekildedir:
```text
+-------------------+       +--------------------+       +-------------+       +--------+
| Mikrodenetleyici  |  RMII | PHY Entegresi      | MDI   | İzolasyon   | RJ45  | Cat5e  |
| (MCU MAC Bloğu)   |<=====>| (Örn: LAN8742A,    |<=====>| Trafosu     |<=====>| Soket  |
| STM32 / ESP32     |  SMI  | DP83848, KSZ8081)  |       | (Magnetics) |       | Kablo  |
+-------------------+       +--------------------+       +-------------+       +--------+
```

1. **PHY Entegresi (Physical Layer Transceiver):**
   - Sayısal MII/RMII sinyallerini analog MDI (Medium Dependent Interface) diferansiyel sinyallerine (`TX+`, `TX-`, `RX+`, `RX-`) dönüştürür.
2. **Magnetics (İzolasyon Trafosu):**
   - Cihazı yüksek gerilim sıçramalarından ve toprak döngülerinden korur (1500V galvanik izolasyon). Ortak mod gürültüsünü filtreler.
3. **MCU - PHY Arayüzleri:**
   - **MII (Media Independent Interface):** 16 pin kullanır. TX/RX için 4'er bit paralel veri hatları ve 25 MHz saat sinyali gerektirir.
   - **RMII (Reduced MII):** Pin sayısını azaltmak için 2 bit TX ve 2 bit RX kullanır. Ancak hat hızı 100 Mbps olduğundan referans saat frekansı **50 MHz** olmak zorundadır. Saat sinyali ya harici bir kristal osilatörden ya da MCU'nun PLL çıkışından (MCO) sağlanır.
   - **SMI / MDIO (Management Interface):** MCU'nun PHY içerisindeki kontrol ve durum register'larını okuyup yazmasını sağlayan 2 telli seri veri yoludur (`MDC`: Saat, `MDIO`: Veri).

#### Auto-Negotiation (Otomatik Anlaşma)
Kablo takıldığı anda iki uçtaki PHY çipleri **FLP (Fast Link Pulse)** sinyalleri göndererek karşılıklı anlaşır:
- Desteklenen Hız: 10 Mbps, 100 Mbps veya 1000 Mbps (Gigabit).
- Çalışma Modu: **Half-Duplex** (aynı anda sadece gönderim veya alım - CSMA/CD) veya **Full-Duplex** (aynı anda eşzamanlı gönderim ve alım).
- İki taraf da ortak destekledikleri en yüksek hız ve Full-Duplex modunda uzlaşır ve **Link Up** durumuna geçer.

#### PHY Register'ları (SMI Üzerinden Yönetim)
- `BMCR` (Basic Mode Control Register - Adres 0x00): Resetleme, hız seçimi, auto-negotiation başlatma, duplex seçimi.
- `BMSR` (Basic Mode Status Register - Adres 0x01): Link durumu (Bit 2: `Link Status`), auto-negotiation tamamlandı mı (Bit 5).

### 3. Komşusuyla ve Alternatifleriyle Farkı
- **K1 (PHY) vs K2 (MAC):** PHY analog sinyalle ve tekil bitlerle ilgilenir; baytların veya çerçevenin ne anlama geldiğini bilmez. MAC ise bitleri birleştirip adresli bir **çerçeve (frame)** oluşturur ve CRC kontrolü yapar.
- **MII vs RMII:** MII 16 pin ile 25 MHz çalışırken, RMII 7-8 pin ile 50 MHz çalışır. RMII gömülü sistemlerde pin tasarrufu sağlar ancak 50 MHz PCB hat yerleşimi (kristal jitter ve sinyal bütünlüğü) daha hassastır.

### 4. Pratik Arıza Belirtileri ve Teşhis

| Arıza Belirtisi | Olası Kök Neden | Gömülü / Saha Teşhis Yöntemi |
| :--- | :--- | :--- |
| **RJ45 Link LED'i hiç yanmıyor** | Kablo takılı değil; PHY beslemesi yok; PHY Reset pini donanımsal olarak LOW'da kalmış; 50 MHz saat sinyali yok. | Multimetre ile PHY `VDD` ve `NRST` pinlerini ölç. Osiloskop ile PHY `XTAL1 / REF_CLK` pininde 50 MHz saat sinyalini gözle. |
| **PHY ID okunamıyor (0x0000 veya 0xFFFF dönüyor)** | SMI (MDC/MDIO) pinleri yanlış yapılandırılmış; PHY I2C/SMI adresi kodda yanlış (Örn: Adres 0 yerine 1 verilmiş). | Firmware'de SMI okuma döngüsü yazarak 0'dan 31'e kadar tüm PHY adreslerini tara. |
| **Link LED yanıyor ama paket alışverişi sıfır** | RMII TX/RX pinleri MCU üzerinde yanlış multiplex edilmiş; 50 MHz clock faz kayması var; DMA RX buffer tahsis edilmemiş. | `ethtool eth0` (Linux) veya LwIP `ethernetif_input` fonksiyonuna kesme (interrupt) düşüp düşmediğini kontrol et. |
| **Aşırı CRC/Frame hatası ve paket düşmesi** | Duplex Mismatch (Biri Full, diğeri Half Duplex kalmış); kalitesiz kablo veya magnetics gürültüsü. | Switch port istatistiklerini kontrol et; iki tarafın da Auto-Negotiation modunda olduğunu doğrula. |

---

## Katman 2 — Veri Bağı Katmanı (Ethernet / MAC / Switch / Multicast)

### 1. Hangi Soruyu Çözer?
> *"Aynı yerel ağ (LAN / broadcast domain) içerisindeki hangi donanıma (MAC) çerçeve göndereceğim ve veri aktarım sırasında elektriksel olarak bozuldu mu?"*

Katman 2, birbiriyle doğrudan kablo veya switch ile bağlı cihazlar arasında **tek hop'luk (single-hop)** güvenilir çerçeve teslimatı sağlar.

### 2. Nasıl Çalışır?

#### Ethernet II Frame (Çerçeve) Anatomisi
Standart bir Ethernet çerçevesi şu alanlardan oluşur:

```text
+----------+--------+-----------+------------+-----------+--------------------+-------+
| Preamble |  SFD   | Hedef MAC | Kaynak MAC | EtherType | Payload (Veri)     | CRC32 |
|  7 Bayt  | 1 Bayt |  6 Bayt   |   6 Bayt   |  2 Bayt   |  46 - 1500 Bayt    | 4 Bayt|
+----------+--------+-----------+------------+-----------+--------------------+-------+
|<----------------- Donanım Katmanı -------------------->|<-- Katman 3 PDU -->|<-FCS->|
```

- **Preamble & SFD (Start Frame Delimiter):** Alıcı PHY'nin saatini gelen bit akışına senkronize etmesi için `10101010...` deseni ve sonundaki `10101011` (0xD5) baytı.
- **Hedef ve Kaynak MAC:** 6'şar baytlık fiziksel donanım adresleri.
- **EtherType:** Çerçevenin içinde hangi Katman 3 protokolünün taşındığını belirtir:
  - `0x0800`: IPv4
  - `0x0806`: ARP (Address Resolution Protocol)
  - `0x86DD`: IPv6
  - `0x8100`: IEEE 802.1Q VLAN Tag
- **Payload (Yük):** Katman 3 paketi. Minimum **46 bayt** (64 bayt minimum çerçeve boyutu kuralı gereği gerekirse padding eklenir), maksimum **1500 bayt** (MTU).
- **CRC32 / FCS (Frame Check Sequence):** Çerçevenin yolda bit hatasına uğrayıp uğramadığını kontrol eden 4 baytlık donanımsal kontrol toplamı. Hatalıysa donanım çerçeveyi anında çöpe atar.

#### MAC Adresi Yapısı (48-bit / 6 Bayt)
Örnek: `00:80:E1:12:34:56`
- **OUI (Organizationally Unique Identifier):** İlk 3 bayt (`00:80:E1` = STMicroelectronics). Üreticiye IEEE tarafından tahsis edilir.
- **NIC (Network Interface Controller):** Son 3 bayt (`12:34:56`). Üreticinin karta verdiği benzersiz seri numarası.

```text
Bayt 0            Bayt 1   Bayt 2   Bayt 3   Bayt 4   Bayt 5
+---------------------------------+-------------------------+
|      OUI (Üretici Kodu)         |   Kart Seri Numarası    |
+---------------------------------+-------------------------+
    | |
    | +--> b1 (U/L): 0 = Global Unique (IEEE), 1 = Locally Administered (Kullanıcı Atamış)
    +----> b0 (I/G): 0 = Unicast (Birebir), 1 = Multicast/Broadcast (Gruba/Herkese)
```

#### Üç Farklı Teslimat Biçimi

| Teslimat Biçimi | Hedef MAC Adresi | Davranış ve Kapsam |
| :--- | :--- | :--- |
| **Unicast** | Hedef cihazın tekil MAC adresi (Örn: `00:80:E1:AA:BB:CC`) | Yalnızca o MAC adresine sahip kart alır ve işler. |
| **Broadcast** | `FF:FF:FF:FF:FF:FF` (Tüm bitler 1) | Aynı yerel ağdaki (VLAN) tüm cihazlar alır ve işler (Örn: ARP Request, DHCP Discover). |
| **Multicast** | `01:00:5E:xx:xx:xx` (IPv4 Multicast için) | Yalnızca o multicast grubuna abone olmuş (listen eden) cihazlar alır. |

> **Önemli Kural (IPv4 Multicast -> MAC Eşlemesi):**
> Bir IPv4 Multicast adresi (`224.0.0.0` - `239.255.255.255`), MAC seviyesine çevrilirken IP adresinin son 23 biti `01:00:5E:00:00:00` şablonunun içine yazılır.
> Örnek: mDNS IP'si `224.0.0.251` -> Multicast MAC: `01:00:5E:00:00:FB`.

#### Switch Nasıl Çalışır? (CAM Tablosu)
Switch bir Katman 2 cihazıdır. Gelen çerçevenin IP'sine bakmaz; yalnızca MAC adreslerine bakar:
1. **Öğrenme (Learning):** Port 1'den gelen çerçevenin **Kaynak MAC**'ine bakar. "Demek ki `00:80:E1:AA:BB:CC` MAC'li cihaz Port 1'de" diyerek CAM (Content Addressable Memory) tablosuna yazar.
2. **Yönlendirme (Forwarding):** Çerçevenin **Hedef MAC**'ine bakar. Tabloda hedef MAC Port 3'te görünüyorsa, çerçeveyi yalnızca Port 3'e iletir.
3. **Taşırma (Flooding):** Hedef MAC tabloda henüz kayıtlı değilse (Unknown Unicast) veya hedef Broadcast/Multicast ise, çerçeveyi geldiği port hariç tüm portlara kopyalar.

### 3. Komşusuyla ve Alternatifleriyle Farkı
- **MAC vs IP:** MAC adresi yerel donanımın değişmeyen kimliğidir (aynı ağda geçerli). IP adresi ise cihazın o an bağlı olduğu ağdaki mantıksal konumudur. Paket router'dan geçerken kaynak/hedef MAC değişir, ancak kaynak/hedef IP değişmez!
- **Switch vs Hub vs Router:**
  - **Hub (K1):** Gelen her elektrik sinyalini körü körüne tüm portlara tekrarlar (çarpışma alanı geniştir).
  - **Switch (K2):** MAC adreslerini öğrenir ve paketleri yalnızca ilgili porta filtreleyerek iletir.
  - **Router (K3):** IP adreslerine bakarak farklı ağlar (subnets) arasında yönlendirme yapar.

### 4. Pratik Arıza Belirtileri ve Teşhis

| Arıza Belirtisi | Olası Kök Neden | Gömülü / Saha Teşhis Yöntemi |
| :--- | :--- | :--- |
| **Ağda rastgele bağlantı kopması (Flapping)** | İki karta aynı MAC adresi atanmış (Duplicate MAC). Switch CAM tablosunda port sürekli yer değiştirir. | Switch loglarında `MAC flapping detected on port X and Y` uyarısını ara. Kartların MAC adreslerini kontrol et. |
| **mDNS / Multicast paketleri gömülü karta ulaşmıyor** | MCU Ethernet donanımındaki **Multicast Hash Filter** kapalı veya yanlış kurulmuş. MAC donanımı paketi CPU'ya iletmeden çöpe atıyor. | STM32 / LwIP başlatılırken MAC filter ayarlarında `ETH_MULTICASTFRAMESFILTER_PERFECT` veya `ETH_MULTICASTFRAMESFILTER_NONE` (Pass All Multicast) seçildiğini doğrula. |
| **Wireshark'ta devasa Broadcast fırtınası** | Switch'ler arasında döngü (Loop) oluşmuş ve STP (Spanning Tree Protocol) kapalı; ağ kilitleniyor. | Wireshark ile saniyede binlerce ARP/Broadcast paketi aktığını gözle; STP'yi aktif et veya fiziksel döngüyü kes. |

---

## Katman 3 — Ağ Katmanı (IP / Mask / Routing / Gateway / ARP / ICMP)

### 1. Hangi Soruyu Çözer?
> *"Hedef cihaz hangi mantıksal IP adresinde, benimle aynı yerel ağda mı yoksa başka bir ağda (router arkasında) mı? IP'sini bildiğim cihazın donanım (MAC) adresini nasıl bulurum?"*

Katman 3, farklı ağların birbiriyle haberleşmesini (**Internetworking**), küresel adreslemeyi ve yönlendirmeyi (routing) çözer.

### 2. Nasıl Çalışır?

#### IPv4 Başlığı Anatomisi
IPv4 başlığı standart olarak 20 bayttır (opsiyonlarla 60 bayta kadar çıkabilir):

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |        Header Checksum        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source IP Address                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination IP Address                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **TTL (Time to Live):** Paketin yaşam süresi (maksimum hop sayısı, örn: 64 veya 128). Her router geçişinde 1 azaltılır. Sıfıra ulaşırsa paket imha edilir ve göndericiye ICMP Time Exceeded yanıtı dönülür (Sonsuz yönlendirme döngülerini engeller).
- **Protocol:** Taşınan Katman 4 protokolü (`0x01` = ICMP, `0x06` = TCP, `0x11` = UDP).
- **Header Checksum:** Yalnızca IP başlığının bütünlüğünü doğrulayan kontrol toplamı.

#### IP Adres Sınıfları ve Özel Bloklar

| IP Bloğu | Kullanım Amacı / Standart | Açıklama |
| :--- | :--- | :--- |
| `10.0.0.0/8` | Özel Ağ (RFC 1918 Private IP) | Yerel ağlar, kurumsal ağlar (İnternete doğrudan route edilmez). |
| `172.16.0.0/12` | Özel Ağ (RFC 1918 Private IP) | `172.16.0.0` - `172.31.255.255` arası yerel ağlar. |
| `192.168.0.0/16` | Özel Ağ (RFC 1918 Private IP) | Ev ve ofis ağları, gömülü cihaz test ağları. |
| `127.0.0.1/8` | Loopback (Localhost) | Cihazın kendi kendine konuşması için ayrılmış yığın arayüzü. |
| `169.254.0.0/16` | Link-Local (APIPA - RFC 3927) | DHCP sunucusu bulunamadığında cihazın rastgele aldığı yerel IP. |
| `224.0.0.0/4` | Multicast Grubu (D Sınıfı) | `224.0.0.0` - `239.255.255.255` arası grup yayınları (Örn: mDNS). |
| `255.255.255.255`| Limited Broadcast | Yerel segmentteki tüm cihazlara yayın. Router'lar bu paketi dışarı geçirmez. |

#### Alt Ağ Maskesi (Subnet Mask) ve Yönlendirme Kararı
Alt ağ maskesi 32 bitlik IP adresini iki parçaya böler: **Ağ Adresi (Network ID)** ve **Cihaz Adresi (Host ID)**.

Örnek: IP = `192.168.1.50`, Maske = `255.255.255.0` (`/24`):
- Ağ Kısmı (İlk 24 bit): `192.168.1.0`
- Cihaz Kısmı (Son 8 bit): `.50`
- Kullanılabilir Cihaz IP Aralığı: `192.168.1.1` – `192.168.1.254` (`.0` Ağ adresi, `.255` Broadcast adresidir).

```text
BİR PAKET GÖNDERİLİRKEN VERİLEN MANTIKSAL KARAR:
(Hedef IP & Kendi Maskem) == (Kendi IP'm & Kendi Maskem) ?
       |
       +--- [ EVET ] ---> Hedef benimle AYNI yerel ağda!
       |                  Doğrudan hedefin MAC adresine gönder.
       |                  (Hedefin MAC'ini bulmak için yerel ARP sorgusu yap).
       |
       +--- [ HAYIR ] --> Hedef BAŞKA bir ağda!
                          Paketi DEFAULT GATEWAY (Router)'a teslim et.
                          (Gateway'in MAC'ini bulmak için ARP sorgusu yap).
```

#### Routing Tablosu (Yönlendirme Tablosu)
Her IP cihazının hafızasında bir yönlendirme tablosu bulunur (`ip route show` veya `route -n`):

```text
Hedef Ağ          Alt Ağ Maskesi     Ağ Geçidi (Gateway)   Arayüz
192.168.1.0       255.255.255.0      0.0.0.0 (On-link)     eth0
0.0.0.0           0.0.0.0            192.168.1.1           eth0  <-- Default Gateway
```
`0.0.0.0/0` (Default Route): Yukarıdaki kuralların hiçbirine uymayan tüm harici IP paketleri (örneğin `8.8.8.8` veya `google.com`) `192.168.1.1` IP'li gateway cihazına yollanır.

#### ARP (Address Resolution Protocol - RFC 826)
> **Problem:** TCP/IP yığını hedef IP'nin `192.168.1.50` olduğunu bilir; ancak Ethernet kartı paketi kabloya basabilmek için hedef **MAC adresini** bilmek zorundadır.

```text
1. Cihaz A (192.168.1.10), Cihaz B'nin (192.168.1.50) MAC adresini bilmiyor.
2. ARP Request (BROADCAST - FF:FF:FF:FF:FF:FF):
   "Kimde 192.168.1.50 IP'si var? Lütfen 192.168.1.10'a MAC adresini söylesin!"
3. Switch bu paketi tüm porta yayar.
4. Yalnızca Cihaz B (192.168.1.50) cevap verir.
5. ARP Reply (UNICAST - Cihaz A'nın MAC'ine):
   "192.168.1.50 bende! Benim MAC adresim: 00:80:E1:AA:BB:CC"
6. Cihaz A bu eşleşmeyi kendi ARP Önbelleğine (ARP Cache) kaydeder ve Ethernet çerçevesini yollar.
```

- **Gratuitous ARP (GARP):** Bir cihaz ağa ilk bağlandığında veya IP değiştirdiğinde, kendi IP'sini soran bir ARP isteği yayınlar (`Who has 192.168.1.50? Tell 192.168.1.50`).
  - **Amacı:** Ağda bu IP'yi kullanan başka biri var mı (IP Çakışma Tespiti - Duplicate Address Detection) ve switch CAM tablolarını tazelemek.

#### ICMP (Internet Control Message Protocol - RFC 792)
Ağın durumunu ve hatalarını raporlayan Katman 3 protokolüdür (Port kavramı yoktur):
- `Echo Request` (Type 8) ve `Echo Reply` (Type 0): `ping` komutunun temelidir.
- `Destination Unreachable` (Type 3): Hedef ağa, hosta veya porta ulaşılamadığında router veya hedef tarafından dönülür.
- `Time Exceeded` (Type 11): TTL sıfırlandığında dönülür (`traceroute` bu mekanizmayla her router'ın IP'sini tespit eder).

### 3. Komşusuyla ve Alternatifleriyle Farkı
- **ARP vs DNS:**
  - **ARP (K2/K3 Köprüsü):** Yalnızca yerel LAN içinde çalışır; **IP -> MAC** dönüşümü yapar.
  - **DNS (K5 Uygulama):** İnternet/ağ genelinde çalışır; **Alan Adı (Hostname) -> IP** dönüşümü yapar.
- **Subnet Mask vs Firewall:** Maske bir güvenlik kuralı değildir; sadece "bu paket doğrudan kablodaki komşuya mı gitsin, yoksa router'a mı verilsin" mantıksal ayrımını yapar.

### 4. Pratik Arıza Belirtileri ve Teşhis

| Arıza Belirtisi | Olası Kök Neden | Gömülü / Saha Teşhis Yöntemi |
| :--- | :--- | :--- |
| **Aynı switch'teki iki karttan biri diğerine ping atamıyor** | Subnet Mask uyumsuzluğu veya IP sınıfı farklılığı (Örn: Biri `192.168.1.10/24`, diğeri `192.168.2.10/24`). | Cihazların IP ve maske değerlerini `ifconfig` / `ip addr` ile kontrol et. |
| **Kart yerel PC ile konuşuyor ama internete çıkamıyor** | Default Gateway (Router IP) tanımlanmamış veya yanlış yazılmış; ya da Gateway ARP sorgusuna yanıt vermiyor. | `ip route` kontrol et. Gateway IP'sine ping at. Gateway üzerinde NAT/Routing ayarını kontrol et. |
| **IP Çakışması (IP Conflict): Ağdaki cihazlar donuyor** | İki farklı cihaza aynı statik IP atanmış. İkisi de ARP Reply dönüyor; trafik bölünür. | Wireshark'ta `Duplicate IP address detected for 192.168.1.x` uyarısını ara. Gratuitous ARP yanıtlarını incele. |
| **İlk ping paketi kayboluyor, sonrakiler geçiyor** | Normal bir durumdur; ilk pakette hedef MAC bilinmediği için ARP çözümlemesi beklenir ve zaman aşımına uğrar. | İkinci ping anında geçiyorsa ARP tablosu oturmuştur, endişelenecek bir durum yoktur. |

---

## Katman 4 — Taşıma Katmanı (Port / Socket / TCP / UDP)

### 1. Hangi Soruyu Çözer?
> *"Paket hedef cihaza ulaştı; peki o cihazda çalışan onlarca farklı uygulamadan hangisine teslim edilecek? İletişim yüzde yüz garantili, hatasız ve sıralı mı olmalı (TCP), yoksa en düşük gecikmeyle hafif bir mesajlaşma mı olmalı (UDP)?"*

Katman 4, süreçten sürece (**Process-to-Process / End-to-End**) iletişimi sağlar.

### 2. Nasıl Çalışır?

#### Port ve Soket Kavramı
- **Port Numarası (16-bit, 0 – 65535):** Cihaz üzerindeki spesifik bir uygulamanın/servisin kapı numarasıdır.
  - **Well-Known Portlar (0 – 1023):** Standart sistem servisleri (HTTP: 80, HTTPS: 443, DNS: 53, DHCP: 67/68, NTP: 123, SSH: 22).
  - **Registered Portlar (1024 – 49151):** Kullanıcı servisleri (MQTT: 1883, mDNS: 5353, MySQL: 3306).
  - **Dynamic / Ephemeral Portlar (49152 – 65535):** İstemcinin dışarıya bağlanırken işletim sistemi tarafından rastgele atanan geçici çıkış portları.
- **Soket (Socket) / 5-Tuple (5'li Demet):** Bir ağ bağlantısını dünyada benzersiz kılan 5 parametre:
  $$\text{Bağlantı Kimliği} = \{\text{Protokol (TCP/UDP)}, \text{Kaynak IP}, \text{Kaynak Port}, \text{Hedef IP}, \text{Hedef Port}\}$$

#### TCP (Transmission Control Protocol - RFC 793)
TCP; güvenilir, sıralı, hata kontrollü ve **bayt akışı (byte stream)** tabanlı çift yönlü bir bağlantı protokolüdür.

```text
TCP 3-YOLLU EL SIKIŞMA (THREE-WAY HANDSHAKE)
İstemci (Client)                                   Sunucu (Server - LISTEN:80)
       |                                                       |
       |  1. SYN (Seq = X)                                     |
       |------------------------------------------------------>| (SYN-RECEIVED)
       |                                                       |
       |  2. SYN + ACK (Seq = Y, Ack = X + 1)                  |
       |<------------------------------------------------------|
       |                                                       |
       |  3. ACK (Seq = X + 1, Ack = Y + 1)                    |
       |------------------------------------------------------>|
       |                                                       |
(ESTABLISHED)                                            (ESTABLISHED)
[ Artık güvenli çift yönlü bayt aktarımı başlayabilir ]
```

- **Güvenilirlik ve Sıralama (Sequence & ACK):** Gönderilen her bayt bir Sequence numarası alır. Alıcı aldığı her veriyi ACK (Acknowledgment) ile onaylar. Belirli sürede (RTO - Retransmission Timeout) ACK gelmezse veri tekrar gönderilir (**Retransmission**).
- **Akış Kontrolü (Flow Control - Sliding Window):** Alıcı, TCP başlığındaki `Window Size` alanıyla "Hafızamda şu an en fazla 4096 bayt yer var, beni boğma" der. Gönderici alıcının kapasitesinden hızlı veri basamaz.
- **Tıkanıklık Kontrolü (Congestion Control):** Ağdaki router'ların tıkanıp paket düşürmemesi için gönderim hızını dinamik ayarlar (Slow Start, Congestion Avoidance algoritmaları).
- **Bayt Akışı (Byte Stream) Mantığı ve Framing Sorunu:**
  > **Kritik Gömülü Yazılım İlkesi:** TCP paket değil, musluktan akan su gibi bayt akışıdır!
  > İstemci `send(soket, "MESAJ1", 6)` ve hemen ardından `send(soket, "MESAJ2", 6)` yapsa bile; sunucu tarafındaki `recv()` tek seferde `"MESAJ1MESAJ2"` (12 bayt) okuyabilir veya önce `"MES"`, sonra `"AJ1MESAJ2"` alabilir.
  > **Çözüm:** Mesaj sınırlarını (Framing) uygulama katmanı çizmelidir (Örn: HTTP `Content-Length`, satır sonu `\r\n` veya 2 baytlık uzunluk öneki).
- **Bağlantı Kapatma:** Kibarca 4 adımlı `FIN` / `ACK` el sıkışması ile veya anlık hata durumunda `RST` (Reset) bayrağı ile yapılır.

#### UDP (User Datagram Protocol - RFC 768)
UDP; bağlantısız (connectionless), durumsuz (stateless), minimum ek yüke sahip (başlığı sadece **8 bayt**) hafif bir protokoldür.

- El sıkışma yok, ACK yok, sıralama garantisi yok, yeniden gönderim yok.
- **Datagram Tabanlıdır:** 1 `sendto()` çağrısı = 1 bağımsız UDP paketi. Karşı taraf `recvfrom()` yaptığında veriyi ya tam bir paket olarak alır ya da paket ağda düşmüşse hiç almaz (TCP gibi parçalar birbirine yapışmaz).
- **Multicast ve Broadcast Desteği:** TCP yalnızca iki nokta (unicast) arasında çalışabilirken, UDP broadcast ve multicast yayınlarını destekler.

### 3. TCP vs UDP Karşılaştırması ve Seçim Kriterleri

| Özellik | TCP | UDP |
| :--- | :--- | :--- |
| **Bağlantı Durumu** | Bağlantı odaklı (3-Way Handshake şart) | Bağlantısız (Direkt paketi basar) |
| **Güvenilirlik** | Yüzde yüz garantili (Kayıp paket tekrar yollanır) | Garantisiz (Best-effort - Paket kaybolabilir) |
| **Veri Sırası** | Sıra garantilidir (Alıcı eksik parçayı bekler) | Sıra garantisi yoktur (Sonraki paket önce gelebilir) |
| **İletişim Modeli** | Bayt Akışı (Byte Stream - Sınır yok) | Mesaj / Datagram (Sınırlar korunur) |
| **Başlık Boyutu** | 20 – 60 Bayt (Ağır) | Sabit 8 Bayt (Çok hafif) |
| **Yayın Türü** | Yalnızca Unicast (Birebir) | Unicast, Broadcast, Multicast |
| **Tipik Kullanım** | HTTP/REST, WebSockets, Dosya/Firmware Güncelleme (OTA), MQTT | DNS, DHCP, mDNS, NTP, Video/Ses Akışı, Yüksek Hızlı Sensör Telemetrisi |

### 4. Pratik Arıza Belirtileri ve Teşhis

| Arıza Belirtisi | Olası Kök Neden | Gömülü / Saha Teşhis Yöntemi |
| :--- | :--- | :--- |
| **`Connection Refused` (Bağlantı Reddedildi)** | Hedef cihaz açık ve erişilebilir; ancak bağlanılmak istenen portta dinleyen (`LISTEN`) hiçbir uygulama yok. Hedef anında `RST` paketi döner. | Hedef cihazda `netstat -tuln` veya `ss -tuln` ile portun açık olup olmadığını doğrula. |
| **`Connection Timeout` (Bağlantı Zaman Aşımı)** | Hedef IP'ye ulaşılamıyor veya arada bir Firewall / Güvenlik Duvarı gelen `SYN` paketini sessizce düşürüyor (`DROP`). | `ping` atarak Katman 3'ü test et; `nc -zv <IP> <Port>` veya Wireshark ile SYN paketine cevap gelip gelmediğine bak. |
| **JSON / Veri Ayrıştırıcı (Parser) Rastgele Çöküyor** | TCP Framing hatası: Gömülü yazılımcı `recv()` fonksiyonunun her zaman tam 1 paket döndüreceğini varsaymış; büyük veri iki parçada gelince parser patlamış. | Uygulama protokolüne uzunluk alanı (Length-prefix) veya sonlandırıcı karakter (`\n`) ekle; tamponlama mekanizması kur. |
| **UDP Paketleri Ağ Yoğunluğunda Kayboluyor** | Alıcı cihazın soket tamponu (Receive Buffer) dolmuş; işletim sistemi veya LwIP yeni gelen paketleri sessizce çöpe atmış. | Soket tampon boyutunu büyüt (`SO_RCVBUF`); UDP alıcı task'ının önceliğini artır. |

---

## Katman 5 — Uygulama Katmanı (DHCP / DNS / mDNS / DNS-SD / HTTP / NTP)

### 1. Hangi Soruyu Çözer?
> *"Ağ parametrelerimi kimden alacağım (DHCP)? İnsanların yazdığı alan adlarını IP'ye nasıl çevireceğim (DNS/mDNS)? Ağdaki servisleri nasıl keşfedeceğim (DNS-SD)? Cihazın durumunu ve verilerini nasıl okuyup değiştireceğim (HTTP/REST)? Saati nasıl senkronize edeceğim (NTP)?"*

Katman 5, doğrudan son kullanıcıya veya gömülü cihazın iş mantığına (business logic) hizmet eden protokolleri içerir.

---

### 2. Protokol Protokol Detaylı İnceleme

#### A. DHCP (Dynamic Host Configuration Protocol - RFC 2131)
- **Amaç:** Cihazların ağa bağlandığında otomatik olarak IP adresi, Alt Ağ Maskesi, Default Gateway ve DNS Sunucusu almasını sağlar.
- **Taşıma:** UDP Port **67** (Sunucu) ve Port **68** (İstemci).

```text
DHCP D.O.R.A. ÇALIŞMA DÖNGÜSÜ
İstemci (Henüz IP'si yok: 0.0.0.0)                         DHCP Sunucusu (192.168.1.1)
       |                                                               |
       | 1. DISCOVER (Broadcast: 255.255.255.255, Port 67)             |
       |    "Ağda DHCP sunucusu var mı? Bana bir IP lazım!"            |
       |-------------------------------------------------------------->|
       |                                                               |
       | 2. OFFER (Unicast/Broadcast, Port 68)                         |
       |    "Sana 192.168.1.50 IP'sini 24 saatliğine önerebilirim."   |
       |<--------------------------------------------------------------|
       |                                                               |
       | 3. REQUEST (Broadcast: 255.255.255.255, Port 67)              |
       |    "Harika! 192.168.1.50 IP'sini kiralamak istiyorum."       |
       |-------------------------------------------------------------->|
       |                                                               |
       | 4. ACK (Acknowledgment, Port 68)                              |
       |    "Onaylandı! IP: 192.168.1.50, Mask: /24, GW: .1, DNS: .1"  |
       |<--------------------------------------------------------------|
```

- **Kritik DHCP Seçenekleri (Options):**
  - `Option 1`: Alt Ağ Maskesi (Subnet Mask)
  - `Option 3`: Router / Default Gateway IP'si
  - `Option 6`: DNS Sunucu IP adresleri
  - `Option 42`: NTP (Zaman) Sunucu IP'leri
  - `Option 51`: IP Kiralama Süresi (Lease Time)
- **Lease Süresi ve Yenileme (Renewal):** Kiralama süresinin %50'si dolduğunda (T1 süresi) istemci sunucuya tekil (unicast) `DHCP Request` atarak süreyi uzatır. Sunucu yanıt vermezse %87.5 süresinde (T2) broadcast yayını yapar.

---

#### B. Klasik DNS (Domain Name System - RFC 1035)
- **Amaç:** Hiyerarşik ve dağıtık bir sistemle insan dostu alan adlarını (`api.firmware.com`) makine dostu IP adreslerine (`93.184.216.34`) dönüştürür.
- **Taşıma:** UDP Port **53** (Büyük yanıtlar veya bölge transferlerinde TCP Port 53).

**Temel DNS Kayıt Türleri:**

| Kayıt Türü | Eşleme Mantığı | Açıklama ve Kullanım |
| :--- | :--- | :--- |
| **A** | İsim $\rightarrow$ IPv4 Adresi | `cihaz.firma.com` $\rightarrow$ `192.168.1.50` |
| **AAAA** | İsim $\rightarrow$ IPv6 Adresi | `cihaz.firma.com` $\rightarrow$ `2001:db8::1` |
| **CNAME** | İsim $\rightarrow$ Başka bir İsim (Alias) | `www.firma.com` $\rightarrow$ `firma.com` |
| **PTR** | IP $\rightarrow$ İsim (Ters DNS) veya Servis Keşfi | `50.1.168.192.in-addr.arpa` $\rightarrow$ `cihaz.local` |
| **SRV** | Servis $\rightarrow$ Host Adı + Port | Servisin hangi cihazda ve hangi portta olduğunu belirtir (**IP vermez!**). |
| **TXT** | İsim $\rightarrow$ Metin (Anahtar=Değer) | Cihaz veya servis hakkında metadata taşır (`path=/api`, `ver=1.0`). |

---

#### C. mDNS (Multicast DNS - RFC 6762)
- **Amaç:** Merkezi bir DNS sunucusu veya DHCP altyapısı bulunmayan yerel ağlarda (Zero-Configuration / Zeroconf) cihazların birbirini `.local` alan adıyla bulmasını sağlar.
- **Protokol Parametreleri:**
  - **Hedef IP:** `224.0.0.251` (IPv4) veya `FF02::FB` (IPv6)
  - **Hedef Port:** UDP **5353**
  - **Özel Alan Adı:** `.local`
- **Çalışma Mantığı:**
  1. PC tarayıcısına `http://sensor-karti.local` yazıldığında, PC `224.0.0.251:5353` adresine multicast bir DNS sorgusu fırlatır: *"sensor-karti.local kimde? A kaydını söyleyin!"*
  2. Ağdaki tüm mDNS destekli cihazlar bu paketi alır.
  3. Adı `sensor-karti` olan gömülü kart multicast yanıt döner: *"sensor-karti.local benim, IP adresim 192.168.1.50!"*
  4. Hiçbir sunucu kurulumuna gerek kalmadan isim çözülmüş olur.

---

#### D. DNS-SD (DNS-Based Service Discovery - RFC 6763)
- **Amaç:** Yalnızca IP adresini değil, ağdaki cihazların **hangi servisleri (HTTP, MQTT, RTSP video, yazıcı vb.) sunduğunu ve hangi portta çalıştığını** otomatik keşfetmeyi sağlar.
- **Mekanizma:** mDNS taşıyıcısı üzerinde standart DNS kayıtlarının (PTR, SRV, TXT, A) hiyerarşik kullanımıdır.

```text
DNS-SD 4 ADIMLI SERVİS KEŞİF ZİNCİRİ

1. ADIM: PTR Sorgusu (Ağda bu tipte hangi servis örnekleri var?)
   Soru : _http._tcp.local PTR ?
   Cevap: "Stm32-Sensor-Node._http._tcp.local"   <-- Servis Örnek Adı (Instance)

2. ADIM: SRV Sorgusu (Bu servis hangi cihazda ve hangi portta?)
   Soru : Stm32-Sensor-Node._http._tcp.local SRV ?
   Cevap: Port = 80, Target = stm32-cihaz.local   <-- Hostname ve Port (IP değil!)

3. ADIM: TXT Sorgusu (Servis hakkında ek parametreler/metadata var mı?)
   Soru : Stm32-Sensor-Node._http._tcp.local TXT ?
   Cevap: "path=/api/v1" "model=STM32H7" "fw=2.1.0"

4. ADIM: A Sorgusu (Bu hostname'in IP adresi nedir?)
   Soru : stm32-cihaz.local A ?
   Cevap: 192.168.1.50                            <-- Nihai IP adresi bulundu!
```

---

#### E. HTTP (Hypertext Transfer Protocol - RFC 7230 / 7231)
- **Amaç:** İstemci-sunucu mimarisinde metin ve ikili veri transferi (REST API, web arayüzleri).
- **Taşıma:** TCP Port **80** (HTTPS için TLS şifrelemeli Port **443**).

**HTTP İstek Formatı (Örnek Telemetri Okuma):**
```http
GET /api/v1/sensors HTTP/1.1\r\n
Host: stm32-cihaz.local\r\n
User-Agent: curl/8.5.0\r\n
Accept: application/json\r\n
\r\n
```

**HTTP Yanıt Formatı:**
```http
HTTP/1.1 200 OK\r\n
Content-Type: application/json; charset=utf-8\r\n
Content-Length: 42\r\n
Connection: close\r\n
\r\n
{"sicaklik": 24.8, "nem": 48.2, "durum": "OK"}
```

- **HTTP Metotları:** `GET` (Oku), `POST` (Yeni kayıt oluştur / komut gönder), `PUT` (Mevcut kaynağı güncelle), `DELETE` (Sil).
- **HTTP Durum Kodları:**
  - `2xx Başarılı:` `200 OK`, `201 Created`, `204 No Content`
  - `3xx Yönlendirme:` `301 Moved Permanently`, `304 Not Modified`
  - `4xx İstemci Hatası:` `400 Bad Request`, `401 Unauthorized`, `404 Not Found`
  - `5xx Sunucu Hatası:` `500 Internal Server Error`, `503 Service Unavailable`

---

#### F. NTP / SNTP (Network Time Protocol - RFC 5905 / RFC 4330)
- **Amaç:** Cihazların iç saatlerini (RTC) mikrosaniye hassasiyetinde senkronize eder.
- **Taşıma:** UDP Port **123**.
- **Gömülü Sistem Önemi:** HTTPS/TLS bağlantılarında SSL sertifikalarının geçerlilik tarihini doğrulamak ve log zaman damgalarını (Timestamp) doğru basmak için gömülü sistemlerde SNTP olmazsa olmazdır.

---

### 3. Komşularıyla ve Birbiriyle Farkları Tablosu

| Protokol Çifti | Temel Ayrım ve Karışıklık Çözümü |
| :--- | :--- |
| **DHCP vs DNS** | **DHCP** cihaza kendi IP/Ağ ayarlarını *verir*; **DNS** ise cihazın başka hedeflerin isimlerini IP'ye *çevirmesini* sağlar. |
| **Klasik DNS vs mDNS** | **Klasik DNS** merkezi bir DNS sunucusuna unicast (Port 53) sorar; **mDNS** sunucusuz yerel ağda multicast (`224.0.0.251:5353`) yayınıyla çalışır. |
| **mDNS vs DNS-SD** | **mDNS** isim çözümleme altyapısıdır (taşıyıcıdır); **DNS-SD** ise bu altyapıyı kullanarak servisleri keşfetme kuralıdır (PTR/SRV/TXT). |
| **SRV Kaydı vs A Kaydı** | **SRV kaydı asla IP adresi vermez**, sadece Hostname ve Port verir! IP'yi almak için o hostname'in **A kaydı** sorgulanır. |

---

### 4. Pratik Arıza Belirtileri ve Teşhis

| Arıza Belirtisi | Olası Kök Neden | Gömülü / Saha Teşhis Yöntemi |
| :--- | :--- | :--- |
| **Cihaz `169.254.x.x` IP adresi alıyor** | DHCP sunucusuna ulaşılamadı veya ağda DHCP sunucusu yok. Cihaz AutoIP moduna düştü. | DHCP sunucusunun açık olduğunu ve switch portunun broadcast engellemediğini kontrol et. |
| **Cihaza IP ile erişiliyor ama `.local` ile erişilemiyor** | mDNS responder çalışmıyor; switch üzerinde IGMP snooping multicast'i engelliyor; MCU MAC multicast filtresi kapalı. | Linux PC'de `avahi-browse -art` veya Windows'ta `dns-sd -B _http._tcp` çalıştırarak mDNS ilanlarını izle. |
| **HTTP Sunucusu 400 Bad Request dönüyor** | İstemcinin gönderdiği HTTP başlıkları bozuk; `Host` başlığı eksik veya `Content-Length` değeri gerçek veri uzunluğuyla uyuşmuyor. | Wireshark ile HTTP paketini yakala ve metin formatını (`\r\n` satır sonları) denetle. |
| **HTTPS bağlantısı handshake aşamasında başarısız oluyor** | Cihazın SNTP ile saati ayarlanmamış; cihaz yılı 1970 gördüğü için sunucunun SSL sertifikasını "Süresi henüz başlamamış" diyerek reddediyor. | Cihaz saatinin güncel olduğunu seri port logundan kontrol et. |

---

# 3. Uçtan Uca Yaşam Döngüsü (Power-On'dan HTTP İsteğine)

Aşağıdaki senaryoda; gömülü bir IoT sensör kartı (STM32 + LwIP) açılmakta ve aynı ağdaki bir mühendis PC'sinin web tarayıcısından `http://sensor-node.local/api/data` adresine istek atılmaktadır.

Tüm katmanların sırayla nasıl devreye girdiğini adım adım izleyelim:

```text
=======================================================================================================
ZAMAN AKIŞI               PROTOKOL / KATMAN                   AÇIKLAMA VE PAKET İÇERİĞİ
=======================================================================================================
[ T0: CİHAZ AÇILIŞI ]
01. Donanım Başlatma      [K1] Fiziksel / PHY                 MCU açılır, RMII 50 MHz clock başlar, PHY resetlenir.
02. Auto-Negotiation      [K1] Auto-Neg (FLP)                 Switch ile PHY anlaşır -> 100 Mbps Full-Duplex Link UP!
03. Yığın İlklendirme     [K2] LwIP MAC Init                  MAC adresi (00:80:E1:11:22:33) donanıma yazılır, RX DMA hazır.
-------------------------------------------------------------------------------------------------------
[ T1: IP ADRESİ ALMA ]
04. DHCP Discover         [K5/K4/K3/K2] UDP Broadcast         Kart: "IP'm yok, kimse var mı?" (0.0.0.0:68 -> 255.255.255.255:67)
05. DHCP Offer            [K5/K4/K3/K2] UDP Unicast           Router: "192.168.1.50 IP'sini teklif ediyorum."
06. DHCP Request          [K5/K4/K3/K2] UDP Broadcast         Kart: "192.168.1.50 IP'sini istiyorum."
07. DHCP ACK              [K5/K4/K3/K2] UDP Unicast           Router: "Onaylandı! IP: 192.168.1.50, Mask: /24, GW: .1, DNS: .1"
-------------------------------------------------------------------------------------------------------
[ T2: ÇAKIŞMA ÖNLEME VE SERVİS İLANI ]
08. Gratuitous ARP        [K3/K2] ARP Broadcast               Kart: "Kimde 192.168.1.50 var?" (Cevap gelmez -> IP güvenli).
09. mDNS Servis İlanı     [K5/K4/K2] UDP Multicast            Kart: 224.0.0.251:5353'e duyurur:
                                                              - sensor-node.local A 192.168.1.50
                                                              - _http._tcp.local PTR Sensor._http._tcp.local
                                                              - Sensor._http._tcp.local SRV Port 80, Target: sensor-node.local
-------------------------------------------------------------------------------------------------------
[ T3: PC TARAFINDAN İSTEK BAŞLATILMASI ]
10. Tarayıcıya Giriş      Kullanıcı Eylemi                    Mühendis tarayıcıya yazar: "http://sensor-node.local/api/data"
11. mDNS İsim Çözme       [K5/K4/K2] UDP Multicast            PC -> 224.0.0.251:5353: "sensor-node.local kimin IP'si?"
12. mDNS Yanıtı           [K5/K4/K2] UDP Multicast            Kart -> PC: "Benim IP'm 192.168.1.50"
-------------------------------------------------------------------------------------------------------
[ T4: MANTIKSAL YÖNLENDİRME VE MAC ÇÖZÜMLEME ]
13. Alt Ağ (Mask) Kararı  [K3] Mantıksal Karar                PC bakar: (192.168.1.50 & /24) == Kendi Ağı -> "Hedef yerel LAN'de!"
14. ARP Sorgusu           [K3/K2] ARP Broadcast               PC: "192.168.1.50'nin MAC adresi nedir?" (FF:FF:FF:FF:FF:FF)
15. ARP Yanıtı            [K3/K2] ARP Unicast                 Kart -> PC: "MAC adresim 00:80:E1:11:22:33" (PC ARP cache'e yazar).
-------------------------------------------------------------------------------------------------------
[ T5: TCP BAĞLANTISI VE HTTP VERİ AKTARIMI ]
16. TCP SYN               [K4/K3/K2] TCP Port 80              PC -> Kart: [SYN] (Seq=1000)
17. TCP SYN+ACK           [K4/K3/K2] TCP Port 80              Kart -> PC: [SYN, ACK] (Seq=5000, Ack=1001)
18. TCP ACK               [K4/K3/K2] TCP Port 80              PC -> Kart: [ACK] (Seq=1001, Ack=5001) -> [ BAĞLANTI KURULDU ]
19. HTTP GET İsteği       [K5/K4/K3/K2] HTTP Request          PC -> Kart: "GET /api/data HTTP/1.1\r\nHost: sensor-node.local\r\n\r\n"
20. TCP ACK               [K4/K3/K2] TCP ACK                  Kart -> PC: Veriyi aldım onayı ([ACK]).
21. HTTP 200 OK Yanıtı    [K5/K4/K3/K2] HTTP Response         Kart -> PC: "HTTP/1.1 200 OK\r\nContent-Length: 17\r\n\r\n{\"temp\":24.5}"
22. TCP FIN / Kapanış     [K4/K3/K2] TCP Teardown             Bağlantı kibarca kapatılır ([FIN/ACK]).
=======================================================================================================
```

---

# 4. Projelerde Katman Katman Arıza Arama Sırası (Troubleshooting)

Projelerde karşılaşılan en büyük hata; web arayüzü açılmadığında doğrudan HTTP veya LwIP kodunu debug etmeye çalışmaktır. **Alt katman çalışmıyorsa üst katmanların hiçbir anlamı yoktur.**

```text
ARIZA ARAMA AKIŞ DİYAGRAMI:

  [ 1. Link LED Yanıyor mu? ] ---------> [ HAYIR ] ---> Kablo, PHY Besleme, 50 MHz Clock kontrol et.
               | (EVET)
  [ 2. Geçerli IP Alındı mı? ] --------> [ HAYIR ] ---> DHCP Sunucusu, Statik IP, VLAN kontrol et.
               | (EVET)
  [ 3. Gateway / Hedef Pingleniyor mu? ]-> [ HAYIR ] ---> Subnet Mask, Gateway IP, ARP tablosu kontrol et.
               | (EVET)
  [ 4. Hedef Port Açık mı? ] ----------> [ HAYIR ] ---> Cihazda servis dinliyor mu (LISTEN), Firewall var mı?
               | (EVET)
  [ 5. mDNS ile İsim Çözülüyor mu? ] --> [ HAYIR ] ---> MAC Multicast Filtresi, IGMP, Port 5353 kontrol et.
               | (EVET)
  [ 6. Uygulama Verisi Doğru mu? ] ----> [ HAYIR ] ---> TCP Framing, HTTP Header, JSON Parser debug et.
```

---

## Sistematik Hata Ayıklama Sırası

### 1. Adım: Fiziksel Katman (K1) Testi
- **Soru:** Kablo takılı mı, elektrik sinyali var mı?
- **Kontrol:** RJ45 üzerindeki Link LED'i yeşil yanıyor mu?
- **Gömülü Kontrol:** SMI/MDIO üzerinden PHY `BMSR` register'ını oku; `Link Status` biti 1 mi? 50 MHz referans saati osiloskopla temiz mi?

### 2. Adım: Veri Bağı Katmanı (K2) Testi
- **Soru:** Donanım paket alıp verebiliyor mu, MAC adresi benzersiz mi?
- **Kontrol:** Switch portundaki sayaçlarda CRC/FCS hatası artıyor mu?
- **Gömülü Kontrol:** Kartın MAC adresi `00:00:00:00:00:00` veya `FF:FF:FF:FF:FF:FF` kalmış mı? Multicast alacaksan donanım multicast filtresi açık mı?

### 3. Adım: Ağ Katmanı (K3) Testi
- **Soru:** Cihazın IP'si ve alt ağ maskesi doğru mu? Yerel ve harici hedeflere ulaşılabiliyor mu?
- **Kontrol:**
  - `ping <Kendi_IP>` (Kendi ağ yığını çalışıyor mu?)
  - `ping <Gateway_IP>` (Router'a ulaşılabiliyor mu?)
  - `arp -a` veya `ip neigh` (Hedef cihazın MAC adresi ARP tablosunda görünüyor mu?)

### 4. Adım: Taşıma Katmanı (K4) Testi
- **Soru:** İlgili port dinleniyor mu ve TCP el sıkışması tamamlanıyor mu?
- **Kontrol:**
  - `netstat -tuln` / `ss -tuln` (Cihaz üzerinde ilgili port `LISTEN` durumunda mı?)
  - `nc -zv <IP> <Port>` veya `telnet <IP> <Port>` (TCP 3-way handshake başarılı mı?)
  - Wireshark'ta sürekli `SYN` gidip `RST` dönüyorsa -> Port kapalı.
  - `SYN` gidip hiçbir şey dönmüyorsa -> Firewall paketi düşürüyor.

### 5. Adım: Uygulama Katmanı (K5) Testi
- **Soru:** Uygulama protokolü doğru formatta veri üretiyor mu?
- **Kontrol:**
  - `curl -v http://<IP>:<Port>/path` (HTTP başlıklarını ve durum kodunu incele).
  - DNS için: `nslookup hedef.com` veya `dig hedef.com`.
  - mDNS için: `avahi-browse -art` (Linux) veya `dns-sd -B _http._tcp` (macOS/Windows).

---

## Kullanılacak Araçlar ve Komutlar

| Araç / Komut | Katman | Kullanım Amacı | Örnek Komut |
| :--- | :--- | :--- | :--- |
| **Multimetre / Osiloskop** | K1 | PHY voltajı, saat frekansı ve sinyal kalitesi ölçümü | 50 MHz RMII clock pini ölçümü |
| **`ethtool` / `mii-tool`** | K1 | Link hızı, duplex durumu ve PHY register kontrolü | `ethtool eth0` |
| **`ip link` / `ifconfig`** | K2 | MAC adresi, interface UP/DOWN ve RX/TX paket sayaçları | `ip link show eth0` |
| **`ip addr` / `ip route`** | K3 | IP adresi, alt ağ maskesi ve yönlendirme tablosu | `ip route show` |
| **`arp` / `ip neigh`** | K2/K3 | IP - MAC eşleşme tablosunu (ARP Cache) listeleme | `arp -n` veya `ip neigh` |
| **`ping` / `traceroute`** | K3 | ICMP ile erişilebilirlik ve yol üzerindeki router analizi| `ping 192.168.1.1` |
| **`ss` / `netstat`** | K4 | Açık portlar, dinleyen soketler ve TCP bağlantı durumları | `ss -tulnp` |
| **`nc` (Netcat)** | K4 | Ham TCP/UDP port bağlantı ve veri transfer testi | `nc -zv 192.168.1.50 80` |
| **`curl`** | K5 | HTTP/REST API testleri ve header analizi | `curl -i -X GET http://192.168.1.50/` |
| **`avahi-browse` / `dns-sd`**| K5 | mDNS ve DNS-SD servis keşfi ve anons takibi | `avahi-browse -art` |
| **Wireshark / `tcpdump`** | K1–K5 | Kablodan geçen tüm paketleri canlı yakalama ve inceleme | `tcpdump -i eth0 -nn -vv` |

---

## Gömülü Sistemler (LwIP / RTOS) Özel Kontrol Listesi

Gömülü sistem projelerinde (STM32 Ethernet + FreeRTOS + LwIP) en sık karşılaşılan tuzaklar:

1. **Bellek Havuzu (PBUF / Pool) Sızıntıları:**
   - LwIP'de `pbuf_free()` çağrısı unutulursa veya yetersiz `PBUF_POOL_SIZE` tanımlanırsa, cihaz birkaç dakika çalıştıktan sonra paket almayı durdurur.
   - `stats_display()` veya `LWIP_STATS` aktif edilerek `MEMP_PBUF_POOL` kullanımı izlenmelidir.
2. **Ethernet DMA RX/TX Descriptor Taşması:**
   - MCU Ethernet DMA'sının RX descriptor halkası (`ETH_RX_DESC_CNT`) dolar ve CPU paketleri zamanında boşaltamazsa DMA kilitlenir (`Rx Buffer Unavailable`).
3. **MAC Donanım Multicast Filtresi (mDNS Problemi):**
   - STM32 MAC konfigürasyonunda `ETH_MULTICASTFRAMESFILTER_NONE` veya ilgili multicast MAC hash tablosu (`ETH_MACMHR`, `ETH_MACMLR`) kurulmazsa, donanım `224.0.0.251` paketlerini CPU'ya iletmeden filtreler!
4. **RMII 50 MHz Saat Sinyali Faz Uyumsuzluğu:**
   - MCU GPIO hızları `GPIO_SPEED_FREQ_VERY_HIGH` seçilmelidir. Saat sinyalindeki aşırı kapasitif yük veya jitter DMA transferlerinde sessiz paket bozulmalarına yol açar.
5. **Thread-Safety (İş Parçacığı Güvenliği):**
   - LwIP `raw API` fonksiyonları thread-safe değildir! FreeRTOS kullanılıyorsa tüm LwIP çağrıları `tcpip_thread` üzerinden (`Netconn API` veya `Socket API`) yapılmalıdır.

---

# 5. Sık Yapılan Kavram Yanılgıları ve Doğruları

### 1. "TCP/IP tek bir protokoldür."
- ❌ **Yanlış:** TCP ve IP birbirinden bağımsız çalışamayan tek bir protokoldür.
- ✔️ **Doğru:** IP Katman 3'te cihazları adresleyen ve paket yönlendiren protokoldür; TCP ise Katman 4'te güvenilir bayt akışı sağlayan protokoldür. Birbirlerinin üzerine otururlar (IP üzerinde UDP de taşınabilir).

### 2. "MAC adresi ile IP adresi birbirinin alternatifidir; biri varken diğerine gerek yoktur."
- ❌ **Yanlış:** Sadece MAC adresiyle veya sadece IP adresiyle internette haberleşilebilir.
- ✔️ **Doğru:** MAC adresi yerel donanım teslimatı için şarttır (Switch IP bilmez). IP adresi ise ağlar arası yönlendirme için şarttır (Router MAC'e göre dünyayı dolaşamaz). Her Ethernet paketinde ikisi de bulunmak zorundadır.

### 3. "Alt ağ maskesi (Subnet Mask) bir güvenlik duvarı veya filtreleme aracıdır."
- ❌ **Yanlış:** Maskeyi daraltarak cihazları birbirinden izole edip güvenlik sağlayabiliriz.
- ✔️ **Doğru:** Subnet mask sadece bir yönlendirme metriğidir. Cihazın "Paketi doğrudan yerel MAC'e mi atayım, yoksa Gateway'e mi göndereyim?" kararını vermesini sağlar. Güvenlik ve izolasyon VLAN veya Firewall (Erişim Listeleri - ACL) ile yapılır.

### 4. "TCP güvenilirdir; dolayısıyla TCP kullanırsak verilerimiz şifrelidir ve güvendedir."
- ❌ **Yanlış:** TCP güvenilir olduğu için veriler çalınamaz veya değiştirilemez.
- ✔️ **Doğru:** TCP'nin "güvenilirliği" (Reliability) kriptografik güvenlik değil; **iletim garantisidir** (Kayıp baytların tekrar yollanması ve sıranın korunması). Veri şifrelemesi TLS/SSL (HTTPS) gibi üst katmanların görevidir.

### 5. "UDP paketleri kaybolabilir; bu yüzden profesyonel projelerde UDP kullanılmamalıdır."
- ❌ **Yanlış:** UDP güvenilmez olduğu için kötü bir protokoldür.
- ✔️ **Doğru:** UDP bilinçli bir tasarım tercihidir. El sıkışma ve yeniden gönderim maliyeti olmadığı için minimum gecikme sağlar. Canlı ses, video, sensör akışı ve yayın (mDNS/DHCP) protokolleri için UDP en doğru seçimdir.

### 6. "DHCP sunucusu aynı zamanda alan adlarını (DNS) da çözer."
- ❌ **Yanlış:** DHCP istemciye alan adlarını çözen sunucudur.
- ✔️ **Doğru:** DHCP yalnızca bir yapılandırma dağıtıcısıdır. İstemciye *"Senin IP'n bu, DNS sunucunun IP'si de şudur"* der. İsim çözme işlemini DNS protokolü ve DNS sunucusu yapar.

### 7. "mDNS ile DNS-SD tamamen aynı şeydir."
- ❌ **Yanlış:** mDNS ve DNS-SD aynı protokolün iki farklı adıdır.
- ✔️ **Doğru:** **mDNS** yerel ağda isimden IP çözme altyapısıdır (`.local`). **DNS-SD** ise bu altyapıyı kullanarak ağdaki servisleri (PTR/SRV/TXT kayıtları ile) keşfetme standardıdır.

### 8. "DNS SRV kaydı doğrudan hedef servisin IP adresini döndürür."
- ❌ **Yanlış:** SRV sorgusu yaptığımızda servisin IP'sini ve portunu alırız.
- ✔️ **Doğru:** **SRV kaydı asla IP adresi döndürmez!** Yalnızca hedef cihazın **Hostname'ini** ve **Port numarasını** verir. IP'yi almak için dönen Hostname'e ayrıca bir **A kaydı** sorgusu atılmalıdır.

### 9. "`.local` alan adları için internetteki DNS sunucusuna (ör. 8.8.8.8) sorgu atılır."
- ❌ **Yanlış:** Tarayıcıya `cihaz.local` yazıldığında modem veya 8.8.8.8 bunu çözer.
- ✔️ **Doğru:** `.local` özel bir TLD'dir (RFC 6762). İşletim sistemi bu ismi görünce klasik DNS sunucusuna sormaz; yerel ağa `224.0.0.251:5353` adresinden **mDNS multicast** paketi atar.

### 10. "TCP soketinde `recv()` çağrıldığında göndericinin `send()` ettiği paketin aynısı tek parça olarak gelir."
- ❌ **Yanlış:** Gönderici 100 baytlık bir paket yolladıysa, alıcı tek bir `recv()` ile kesinlikle 100 bayt alır.
- ✔️ **Doğru:** TCP bir bayt akışıdır (Stream). 100 baytlık veri yolda parçalanıp 40 + 60 olarak gelebilir veya ardışık iki paket birleşip tek seferde 200 bayt olarak okunabilir. Mesaj çerçeveleme (Framing) uygulama katmanının sorumluluğundadır.

---

# 6. Pekiştirme ve Sınama Soruları (Cevap Anahtarlı)

---

## Bölüm A: Kavramsal ve Protokol Soruları

#### Soru 1
Bir gömülü kartta RMII arayüzü ile PHY çipi bağlanmıştır. Kartın RJ45 Link LED'i yanmakta ancak LwIP hiçbir Ethernet çerçevesi alamamakta ve gönderememektedir. Osiloskop ile yapılan ölçümde RMII referans saatinin 25 MHz olduğu görülmüştür. Sorun nedir?

#### Soru 2
İki cihaz aynı switch'e bağlıdır:
- Cihaz A: IP = `192.168.1.10`, Mask = `255.255.255.0` (`/24`), Gateway = `192.168.1.1`
- Cihaz B: IP = `192.168.1.200`, Mask = `255.255.0.0` (`/16`), Gateway = Tanımsız
- Ağda fiziksel bir Gateway (Router) bulunmamaktadır.

Cihaz B, Cihaz A'ya ping atabilmekte; ancak Cihaz A, Cihaz B'ye ping attığında yanıt alamamaktadır. Bu durumun nedeni nedir?

#### Soru 3
Bir gömülü sistem geliştiricisi, cihazın sunduğu HTTP REST servisini DNS-SD ile ağa duyurmuştur. Bir mobil uygulama bu servisi keşfetmek istediğinde sırasıyla hangi 4 DNS kayıt türünü sorgulamalıdır ve her biri ne bilgi döner?

#### Soru 4
Bir ağda Wireshark ile paket yakalanırken, bir istemcinin sunucuya gönderdiği TCP SYN paketine karşılık sunucudan anında `RST, ACK` bayraklı bir TCP paketi döndüğü görülmüştür. Bu durum ne anlama gelir?

#### Soru 5
UDP protokolü kullanarak saniyede 100 kez telemetri paketi basan bir sensör kartında, alıcı PC tarafında ara sıra paket kayıpları yaşanmaktadır. Ancak Wireshark ile incelendiğinde paketlerin PC'nin Ethernet kartına eksiksiz ulaştığı görülmüştür. Paketler nerede kaybolmaktadır?

---

## Bölüm B: Saha ve Arıza Teşhis Senaryoları

#### Senaryo 1: "Cihaz Pingleniyor Ama .local ile Web Sayfası Açılmıyor"
Sahaya kurulan bir STM32 tabanlı endüstriyel cihazın IP adresine (`192.168.1.45`) tarayıcıdan girildiğinde web arayüzü sorunsuz açılmakta ve `ping 192.168.1.45` yanıt vermektedir. Ancak tarayıcıya `http://endustriyel-kart.local` yazıldığında "Sunucu bulunamadı" hatası alınmaktadır. Wireshark ile bakıldığında PC'nin `224.0.0.251:5353` adresine sorgu attığı ancak karttan hiçbir yanıt dönmediği görülmüştür.

- Hata hangi katmandadır?
- Gömülü yazılım ve donanım tarafında kontrol edilmesi gereken 2 kritik nokta nedir?

#### Senaryo 2: "İlk TCP Bağlantısı Çok Yavaş Açılıyor"
Gömülü bir karta TCP üzerinden bağlanan bir PC istemcisi, ilk bağlantı isteğinde (SYN) yaklaşık 1 saniyelik bir gecikme yaşamakta, ancak bağlantı bir kez kurulduktan sonra veri aktarımı mikrosaniyeler içinde akmaktadır. Wireshark kaydında ilk SYN paketinden hemen önce bir ARP isteği olduğu ve bu isteğin tekrarlandığı görülmüştür. Kök neden ne olabilir?

#### Senaryo 3: "Aynı Ağda İki Kart Takılınca Switch Çıldırıyor"
Laboratuvarda tek bir kart çalışırken her şey kusursuzdur. Ancak ikinci özdeş kart switch'e takıldığı anda her iki kartın da iletişimi kesilmekte, pingler %90 oranında düşmekte ve switch'in port LED'leri sürekli yanıp sönmektedir. Sorun ne olabilir?

---

## Ayrıntılı Cevap Anahtarı

<details>
<summary><b>Cevapları Görüntülemek İçin Tıklayın</b></summary>

### Bölüm A Cevapları

#### Soru 1 Cevabı:
- **Kök Neden:** RMII (Reduced MII) standardı, 2 bitlik veri yolu üzerinden 100 Mbps hız sağlayabilmek için **50 MHz** referans saat frekansı ile çalışmak zorundadır. 25 MHz saat sinyali standart MII (4 bit) arayüzüne aittir.
- **Sonuç:** PHY 25 MHz saat ile çalıştığı için gelen bitleri yanlış örneklemekte, çerçeveler donanım seviyesinde CRC hatası alarak çöpe atılmakta ve MAC kontrolcüsüne ulaşamamaktadır.

#### Soru 2 Cevabı:
- **Kök Neden:** Alt ağ maskesi ve yönlendirme kararı mantığı.
- **Açıklama:**
  - Cihaz B'nin maskesi `/16` (`255.255.0.0`) olduğu için `192.168.1.10` adresini kendisiyle aynı yerel ağda kabul eder. Doğrudan ARP sorgusu yapar, Cihaz A'nın MAC'ini alır ve paketi başarıyla teslim eder.
  - Cihaz A'nın maskesi ise `/24` (`255.255.255.0`) olduğu için `192.168.1.200` IP'sini aslında kendi ağında görür; fakat Cihaz A'nın maskesi yanlış yapılandırılmış veya farklı bir alt ağ algısı oluşmuşsa paketi Gateway'e yollamaya çalışır. Gateway olmadığı için ARP cevapsız kalır ve paket yola çıkamaz.

#### Soru 3 Cevabı:
DNS-SD keşif zincirinde sorgulanan 4 kayıt:
1. **PTR Kaydı (`_http._tcp.local`):** Ağda çalışan tüm HTTP servis örneklerinin adlarını (Instance Name) listeler (Örn: `SensorNode._http._tcp.local`).
2. **SRV Kaydı (`SensorNode._http._tcp.local`):** Bu servisi sunan cihazın Hostname'ini (`sensor.local`) ve Port numarasını (`80`) verir (IP vermez!).
3. **TXT Kaydı (`SensorNode._http._tcp.local`):** Servise ait metadata ve anahtar-değer çiftlerini (`path=/api`, `version=1.0`) verir.
4. **A Kaydı (`sensor.local`):** Bulunan hostname'in nihai IPv4 adresini (`192.168.1.50`) verir.

#### Soru 4 Cevabı:
- **Anlamı:** Katman 3 seviyesinde IP adresine ulaşılmıştır (cihaz açıktır); ancak bağlanılmak istenen Katman 4 portunda dinleyen (`LISTEN` durumunda olan) hiçbir uygulama/servis yoktur.
- İşletim sistemi veya LwIP yığını, kapalı bir porta gelen SYN paketine karşılık bağlantıyı anında reddetmek için `RST, ACK` (Reset) paketi fırlatır.

#### Soru 5 Cevabı:
- **Kök Neden:** İşletim Sistemi / Soket Alıcı Tamponu (Socket Receive Buffer Overflow).
- **Açıklama:** Paketler kablodan ve ağ kartından başarıyla geçmiş ancak PC'deki kullanıcı uygulaması `recvfrom()` ile paketleri yeterince hızlı çekememiştir. İşletim sisteminin UDP soket tamponu dolduğu için yeni gelen paketler çekirdek (kernel) seviyesinde sessizce çöpe atılmıştır.

---

### Bölüm B Cevapları

#### Senaryo 1 Cevabı:
- **Katman:** Katman 2 (Veri Bağı) ve Katman 5 (Uygulama - mDNS).
- **Kontrol Noktaları:**
  1. **MAC Donanım Multicast Filtresi:** STM32 MAC kontrolcüsünde Multicast Hash Filtresi kapalıysa veya `ETH_MULTICASTFRAMESFILTER_NONE` seçilmemişse, donanım `224.0.0.251` IP'sine ait `01:00:5E:00:00:FB` multicast MAC adresli paketleri CPU'ya aktarmadan çöpe atar.
  2. **LwIP IGMP / Multicast Ayarı:** `lwipopts.h` dosyasında `LWIP_IGMP 1` ve `LWIP_MDNS_RESPONDER 1` bayraklarının aktif olduğunu ve mDNS responder'ın ilgili interface'e (`netif`) bağlandığını doğrula.

#### Senaryo 2 Cevabı:
- **Kök Neden:** Gömülü kartın ARP isteklerine geç yanıt vermesi veya gelen ARP broadcast paketlerini CPU kuyruğunda geciktirmesi.
- **Açıklama:** PC, karta TCP SYN atmadan önce kartın MAC adresini öğrenmek için ARP Request yayınlamıştır. Kartın Ethernet interrupt'ı veya LwIP ana döngüsü başka bir bloklayıcı görev (ör. Flash yazma veya uzun süren bir döngü) nedeniyle geciktiği için ARP cevabı gecikmiş, PC ilk SYN paketini zaman aşımına uğratıp ikinci denemede bağlanabilmiştir.

#### Senaryo 3 Cevabı:
- **Kök Neden:** **MAC Adresi Çakışması (Duplicate MAC Address).**
- **Açıklama:** İki karta da fabrika çıkışında veya kod içinde aynı statik MAC adresi (`00:80:E1:12:34:56`) gömülmüştür. İki kart da aynı switch'e bağlandığında switch'in CAM tablosu sürekli olarak o MAC adresinin Port 1'de mi yoksa Port 2'de mi olduğunu şaşırır (**MAC Table Flapping / Thrashing**). Paketler rastgele bir porta gider ve iki cihazın da iletişimi felç olur. Çözüm: Her karta benzersiz bir MAC adresi atamaktır (ör. MCU Unique ID'den türetilerek).

</details>

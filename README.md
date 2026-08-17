# Ethernet ve Ağ Temelleri Çalışma ve Başvuru Kılavuzu

Bu depo; Ethernet, TCP/IP ve ilişkili ağ protokollerini en alt fiziksel katmandan (PHY/Link) en üst uygulama katmanına (HTTP/DNS-SD) kadar hiyerarşik bir bütünlükle ele alan kapsamlı Türkçe dokümantasyon ve çalışma setini içerir.

Dokümanlar; **gömülü sistemler (STM32, ESP32, LwIP)**, **Linux tabanlı ağ cihazları** ve **PC araçları (Wireshark, netcat, curl, iproute2)** ile çalışan mühendislerin sahada ve geliştirmede doğrudan başvurabileceği teknik derinlikte hazırlanmıştır.

---

## 📚 Dokümantasyon İndeksi

- 📖 **[Ağ Temelleri ve Ethernet Mimarisi Çalışma Kılavuzu](docs/ag-temelleri.md)** — Kablodan HTTP'ye hiyerarşik, derinlemesine başvuru notu.

---

## 🧱 Kapsanan Katman Hiyerarşisi

Dokümantasyon, her bir kavramı **"Hangi soruyu çözer?"**, **"Nasıl çalışır?"**, **"Komşusuyla farkı nedir?"** ve **"Pratik arıza belirtileri ve teşhis yöntemleri"** olmak üzere 4 temel boyutta ele alır:

```text
+------------------------------------------------------------------------------------+
| KATMAN 5: UYGULAMA (Application)                                                   |
| DHCP (DORA), DNS (A/AAAA/SRV/PTR/TXT), mDNS (.local), DNS-SD, HTTP, NTP/SNTP        |
| -> "Ne konuşuyoruz? Ağ ayarları nasıl dağıtılır, servisler nasıl keşfedilir?"     |
+------------------------------------------------------------------------------------+
| KATMAN 4: TAŞIMA (Transport)                                                       |
| Portlar (0-65535), 5-Tuple Soket, TCP (3-Way Handshake, Sliding Window), UDP       |
| -> "Hangi uygulamaya, güvenilir bayt akışıyla mı yoksa hafif datagramla mı?"      |
+------------------------------------------------------------------------------------+
| KATMAN 3: AĞ (Network)                                                             |
| IPv4/IPv6, Alt Ağ Maskesi (Subnet Mask & AND işlemi), Routing, Gateway, ARP, ICMP  |
| -> "Hedef nerede, aynı yerel ağda mı yoksa router arkasında mı?"                   |
+------------------------------------------------------------------------------------+
| KATMAN 2: VERİ BAĞI (Data Link / Ethernet)                                         |
| Ethernet II Frame, MAC Adresleme (Unicast/Broadcast/Multicast), Switch, CRC        |
| -> "Aynı fiziksel kablodaki hangi donanıma (MAC) iletilecek?"                      |
+------------------------------------------------------------------------------------+
| KATMAN 1: FİZİKSEL (Physical / PHY)                                                |
| PHY Çipi (LAN8742 vb.), Magnetics, MII/RMII (50 MHz Clock), Auto-Negotiation, Link |
| -> "Elektrik sinyali ve saat oturmuş mu, bitler bozulmadan akıyor mu?"             |
+------------------------------------------------------------------------------------+
```

---

## 🚀 Öne Çıkan Başlıklar

1. **Uçtan Uca Yaşam Döngüsü (End-to-End Lifecycle):**
   - Bir IoT sensör kartının açılış anından (Power-on Reset) başlayarak PHY Link-up, DHCP DORA ile IP alımı, Gratuitous ARP, mDNS servis ilanı, PC tarafından keşif, ARP çözümleme, TCP 3-way handshake ve HTTP 200 OK yanıtına kadar uzanan 22 adımlı kronolojik akış.

2. **Projelerde Katman Katman Arıza Arama Kılavuzu (Troubleshooting):**
   - K1'den K5'e sistematik hata ayıklama sırası.
   - Multimetre/Osiloskoptan Wireshark, `tcpdump`, `ethtool`, `ip`, `ss`, `nc` ve `curl` araçlarına kadar pratik komut tablosu.
   - Gömülü sistemlere (LwIP PBUF sızıntıları, DMA descriptor taşması, MAC multicast hash filtreleri) özel kontrol listesi.

3. **Sık Yapılan 10 Kavram Yanılgısı:**
   - TCP bayt akışı ile paket sınırı (framing) yanılgısı, MAC-IP ayrımı, maske-güvenlik karmaşası, mDNS vs DNS-SD, SRV kaydı çalışma mantığı vb.

4. **Pekiştirme ve Sınama Soruları (Cevap Anahtarlı):**
   - Protokol mantığını pekiştiren kavramsal sorular ve sahada karşılaşılan arıza senaryoları.
   - Ayrıntılı gerekçeleriyle hazırlanmış açılır-kapanır cevap anahtarı.

---

## 🛠️ Hızlı Başvuru Komutları

| Amaç | Linux / PC Komutu | Gömülü / Donanım Karşılığı |
| :--- | :--- | :--- |
| **Link Durumu** | `ethtool eth0` | PHY `BMSR` Register (Link Status Biti) |
| **IP ve Maske** | `ip addr show` | LwIP `netif->ip_addr`, `netif->netmask` |
| **Yönlendirme** | `ip route show` | LwIP `netif->gw` (Default Gateway) |
| **ARP Tablosu** | `ip neigh` / `arp -n` | LwIP `etharp_find_entry()` |
| **Port Dinleme**| `ss -tulnp` | LwIP `tcp_bind()`, `tcp_listen()` |
| **Servis Keşfi**| `avahi-browse -art` | LwIP `mdns_resp_announce()` |
| **HTTP İstek**  | `curl -v http://cihaz.local/` | LwIP HTTPD / REST Handler |

# Cisco Packet Tracer Ağ Topolojileri

Cisco Packet Tracer 9.0 ile hazırladığım yedi ağ tasarımı. İlk beşi tek bir konuya odaklanan laboratuvar çalışmaları, son ikisi ise bu konuları bir araya getiren, bitirme projesi ölçeğinde kapsamlı tasarımlar.

Her klasörde şunlar var:

- `*.pkt` — Packet Tracer dosyası (9.0.0 ve üzeri ile açılır)
- `topoloji.png` — mantıksal çalışma alanının görüntüsü
- `configs/` — yönlendirici ve anahtarların çalışan yapılandırmaları (`show running-config` çıktısı)

| # | Topoloji | Konular | Cihaz |
|---|---|---|---|
| 01 | [VLAN ve Router-on-a-Stick](#01--vlan-ve-router-on-a-stick) | VLAN, 802.1Q trunk, inter-VLAN routing, DHCP, port-security | 10 |
| 02 | [Çok Alanlı OSPF](#02--çok-alanlı-ospf) | OSPF alanları, ABR, MD5 doğrulama, totally stubby alan | 14 |
| 03 | [HSRP, EtherChannel ve Rapid-PVST+](#03--hsrp-etherchannel-ve-rapid-pvst) | Ağ geçidi yedekliliği, LACP, STP kök köprü tasarımı | 11 |
| 04 | [NAT/PAT, ACL ve DMZ](#04--natpat-acl-ve-dmz) | PAT, statik NAT, genişletilmiş ACL, DMZ | 12 |
| 05 | [Merkez–Şube Kurumsal Ağ](#05--merkezşube-kurumsal-ağ) | VLAN, OSPF, merkezi DHCP + relay, PAT, misafir ağı | 12 |
| 06 | [Üniversite Kampüs Ağı](#06--üniversite-kampüs-ağı) | Omurga, bina başına OSPF alanı, veri merkezi, DMZ, internet kenarı | 55 |
| 07 | [Holding – Çok Şubeli WAN](#07--holding--çok-şubeli-wan) | Servis sağlayıcı halkası, GRE tünelleri, OSPF, PAT, yedekli dağıtım katmanı | 31 |

Kullanılan cihazlar: Cisco 2911 yönlendirici, Catalyst 2960-24TT anahtar, PC ve sunucu.

## Dosyaları açmak

1. Depoyu indirin ya da klonlayın.
2. İlgili klasördeki `.pkt` dosyasını Packet Tracer 9.0.0 veya daha yeni bir sürümle açın.
3. Açılıştan sonra protokollerin yakınsaması için kısa bir süre bekleyin (küçük topolojilerde yaklaşık bir dakika, 06 ve 07'de iki–üç dakika). İsterseniz *Fast Forward Time* düğmesiyle hızlandırabilirsiniz.

Tüm cihazlarda `enable` parolası `cisco`, SSH kullanıcısı `admin` / `cisco`'dur (yalnızca laboratuvar amaçlı).

## Ortak yönetim ayarları

Her topolojide yönlendirici ve anahtarlara yalnızca SSH (sürüm 2) ile, yerel kullanıcıyla girilir; Telnet kapalıdır. VTY hatlarına `YONETIM-ERISIM` erişim listesi uygulanır, böylece yalnızca senaryoda belirtilen yönetim ağlarından bağlanılabilir. 01, 05, 06 ve 07'de cihazlar saatlerini NTP sunucusundan alır ve günlüklerini merkezi syslog sunucusuna gönderir; sunucu adları DNS ile çözülür.

## Doğrulama

Dosyaların hepsi Packet Tracer 9.0.0'da açılıp yeniden kaydedildi; kaydedilen çalışan yapılandırmalar tasarımla satır satır karşılaştırıldı.

| # | Yapılan test | Sonuç |
|---|---|---|
| 01 | 5 PC'nin yönlendiricideki DHCP havuzlarından adres alması | 5/5 |
| 02 | R4'ten (alan 1) alan 0 ve alan 2'deki PC'lere ping, OSPF komşuluğu | Başarılı, komşuluk FULL |
| 03 | HSRP durum geçişleri; R1'den iki VLAN'daki PC'lere ve CORE'a ping | Başarılı |
| 04 | İç ağdaki PC1'den internet sunucusuna ping (PAT üzerinden) | Başarılı |
| 05 | Merkez ve şubedeki 5 PC'nin merkezi DHCP sunucusundan adres alması | 5/5 |
| 06 | 20 PC'nin veri merkezindeki DHCP sunucusundan adres alması; bina yönlendiricisinde OSPF rotaları | 20/20, tüm alanlar öğrenildi |
| 07 | 13 PC'nin adres alması; GRE tünellerinin kurulması | 13/13 |
| 07 | Genel müdürlükteki PC'den `intranet.holding.local` adına ping (DNS) | 4/4 |
| 07 | Aynı PC'den R-GM'ye SSH ile giriş; `show clock` ile NTP'den alınan saat | Başarılı |
| 07 | Genel müdürlükten İzmir şubesine GRE tüneli üzerinden ping | 4/4 |

Ping testlerinde ilk paket ARP çözümlemesi sırasında düşer; bu beklenen davranıştır.

Henüz test edilmeyenler: 04 ve 06'da dışarıdan DMZ'ye statik NAT ile erişim ve ACL'lerin engellediği trafik; 03'te bağlantı koparıldığında HSRP devralması; SSH, DNS ve NTP'nin 07 dışındaki topolojilerde denenmesi (yapılandırma aynı kalıptan üretildi ve Packet Tracer tarafından eksiksiz kabul edildi); syslog sunucusuna düşen kayıtlar.

Kapsam dışı bırakılanlar: IPv6, IPsec VPN, RADIUS/TACACS+, kablosuz ağ ve katman 3 anahtarlar.

---

## 01 — VLAN ve Router-on-a-Stick

![01](01-vlan-inter-vlan-routing/topoloji.png)

**Senaryo.** Üç katlı küçük bir ofis; muhasebe, insan kaynakları ve bilgi işlem birimleri aynı anahtarları paylaşıyor ama ağları ayrı olmalı.

**İstenenler**

1. Her birim için ayrı VLAN (10, 20, 30) ve ayrı bir yönetim VLAN'ı (99, native)
2. VLAN'lar arası yönlendirme tek yönlendirici üzerinden (router-on-a-stick)
3. PC'ler adreslerini DHCP ile alsın; ilk 10 adres ayrılmış olsun
4. Erişim portlarında port-security (en fazla 2 MAC, sticky) ve BPDU guard
5. Kullanılmayan portlar kapalı ve ayrı bir VLAN'da
6. Cihazlara yalnızca SSH ile ve yalnızca bilgi işlem ile yönetim ağından erişilsin
7. Saat NTP ile eşitlensin, günlükler sunucuya gönderilsin

**Çözüm**

Üç anahtarlı küçük bir ofis ağı. VLAN'lar arası yönlendirmeyi tek bir yönlendirici alt arayüzlerle yapar.

| VLAN | Ad | Ağ | Ağ geçidi |
|---|---|---|---|
| 10 | MUHASEBE | 192.168.10.0/24 | 192.168.10.1 |
| 20 | INSAN-KAYNAKLARI | 192.168.20.0/24 | 192.168.20.1 |
| 30 | BILGI-ISLEM | 192.168.30.0/24 | 192.168.30.1 |
| 99 | YONETIM (native) | 192.168.99.0/24 | 192.168.99.1 |

- R1 üzerinde her VLAN için DHCP havuzu; ilk 10 adres hariç tutulur.
- Trunk'larda native VLAN 99 ve izinli VLAN listesi.
- Erişim portlarında port-security (en fazla 2 MAC, sticky, restrict), PortFast ve BPDU guard.
- Kullanılmayan portlar VLAN 999'da ve kapalı.
- SW1 tüm VLAN'lar için kök köprü.

## 02 — Çok Alanlı OSPF

![02](02-cok-alanli-ospf/topoloji.png)

**Senaryo.** Bir merkez ve iki uzak ofisi olan kurum; yönlendirme tabloları küçük kalmalı ve uzak ofislerdeki değişiklikler omurgayı etkilememeli.

**İstenenler**

1. OSPF üç alana bölünsün: omurga (0), Şube-A (1), Şube-B (2)
2. Router-ID'ler loopback adreslerinden elle verilsin
3. Omurga komşulukları MD5 ile doğrulansın
4. Alan 2 totally stubby olsun (yalnızca varsayılan rota alsın)
5. Kullanıcı ağlarına OSPF paketi gönderilmesin (passive-interface)
6. Tüm PC'ler birbirine ulaşabilsin
7. Yönlendiricilere yalnızca SSH ile ve yalnızca merkez ağından erişilsin

**Çözüm**

Beş yönlendiricili, üç alanlı bir OSPF tasarımı.

| Alan | Yönlendiriciler | Ağlar |
|---|---|---|
| 0 (omurga) | R1, R2, R3 | 10.0.12.0/30, 10.0.13.0/30, 10.0.23.0/30, 172.16.0.0/24 |
| 1 | R1 (ABR), R4 | 10.1.14.0/30, 172.16.1.0/24 |
| 2 (totally stubby) | R3 (ABR), R5 | 10.2.35.0/30, 172.16.2.0/24 |

- Router-ID'ler Loopback0 adreslerinden (1.1.1.1 – 5.5.5.5).
- Omurgada MD5 kimlik doğrulama.
- Alan 2 totally stubby: R5 yalnızca varsayılan rota alır.
- LAN arayüzleri passive; referans bant genişliği 1000 Mb/sn.

## 03 — HSRP, EtherChannel ve Rapid-PVST+

![03](03-hsrp-etherchannel-stp/topoloji.png)

**Senaryo.** Ağ geçidi ya da bir anahtar arızalandığında kullanıcıların bağlantısı kesilmemesi gereken bir ofis.

**İstenenler**

1. İki yönlendirici HSRP ile tek sanal ağ geçidi sunsun
2. Yük paylaşımı: VLAN 10'da R1, VLAN 20'de R2 aktif olsun
3. Uplink koparsa aktiflik diğer yönlendiriciye geçsin (track + preempt)
4. Dağıtım anahtarları arası iki hat LACP ile tek kanal olsun
5. STP kök köprüleri HSRP aktif yönlendiricileriyle aynı tarafta olsun
6. Erişim anahtarları iki dağıtım anahtarına da bağlı olsun
7. Yönetim erişimi yalnızca SSH ile

**Çözüm**

Yedekli ağ geçidi ve yedekli anahtarlama katmanı.

| VLAN | Ağ | Sanal ağ geçidi | Aktif | Yedek |
|---|---|---|---|---|
| 10 PERSONEL | 192.168.10.0/24 | 192.168.10.1 | R1 | R2 |
| 20 MUHENDISLIK | 192.168.20.0/24 | 192.168.20.1 | R2 | R1 |
| 99 YONETIM | 192.168.99.0/24 | 192.168.99.1 | — | — |

- HSRP önceliği, preempt ve uplink arayüzü izleme (track).
- SW1–SW2 arasında iki hatlı LACP EtherChannel (Po1).
- STP kök köprüleri HSRP ile hizalı: SW1 VLAN 10, SW2 VLAN 20 için kök.
- Erişim anahtarları her iki dağıtım anahtarına bağlı.

## 04 — NAT/PAT, ACL ve DMZ

![04](04-nat-acl-dmz/topoloji.png)

**Senaryo.** İnternete tek bir kenar yönlendiricisiyle çıkan ve dışarıya web ile FTP hizmeti veren bir şirket.

**İstenenler**

1. İç ağ internete tek genel adres üzerinden çıksın (PAT)
2. DMZ'deki web ve FTP sunucuları sabit genel adreslerle yayınlansın (statik NAT)
3. Dışarıdan yalnızca web (80/443) ve FTP (21) kabul edilsin
4. DMZ'den iç ağa bağlantı başlatılamasın
5. İç ağ hem DMZ'ye hem internete erişebilsin
6. Kenar yönlendiricisine yalnızca yönetici PC'sinden SSH ile erişilsin

**Çözüm**

İç ağ, DMZ ve internet olmak üzere üç bölgeli kenar tasarımı.

| Bölge | Ağ | Çeviri |
|---|---|---|
| İç ağ | 192.168.1.0/24 | PAT (EDGE Gi0/2 üzerinden overload) |
| DMZ | 172.16.10.0/24 | Statik NAT: WEB → 198.51.100.10, FTP → 198.51.100.11 |
| ISP bağlantısı | 198.51.100.0/30 | — |
| İnternet | 203.0.113.0/24 | — |

- `OUTSIDE-IN`: dışarıdan yalnızca WEB (80/443) ve FTP (21); dönüş trafiği için `established` ve `echo-reply`.
- `DMZ-IN`: DMZ, iç ağa bağlantı başlatamaz; yalnızca yanıt verebilir.

## 05 — Merkez–Şube Kurumsal Ağ

![05](05-merkez-sube-kurumsal-ag/topoloji.png)

**Senaryo.** Bir merkez ofisi ve kiralık hatla bağlı bir şubesi olan işletme; misafirlere internet verilecek ama iç ağ korunacak.

**İstenenler**

1. Merkezde personel, misafir ve sunucu VLAN'ları
2. Merkez ile şube arasında OSPF
3. Tek bir DHCP sunucusu her iki konuma da adres dağıtsın (relay)
4. İnternet çıkışı merkezden PAT ile
5. Misafir ağı internete çıkabilsin ama iç ağa erişemesin
6. Sunucu adları DNS ile çözülsün
7. Cihaz yönetimi SSH ile, yalnızca sunucu ve yönetim ağından
8. NTP ve merkezi günlük (syslog)

**Çözüm**

Bir merkez ve bir şubeden oluşan küçük işletme ağı.

| Konum | VLAN / Ağ | Adres bloğu |
|---|---|---|
| Merkez | 10 PERSONEL | 10.10.10.0/24 |
| Merkez | 20 MISAFIR | 10.10.20.0/24 |
| Merkez | 30 SUNUCU | 10.10.30.0/24 |
| Merkez | 99 YONETIM | 10.10.99.0/24 |
| Şube | LAN | 10.20.10.0/24 |
| Merkez–şube hattı | — | 10.255.0.0/30 |

- Merkez ile şube arasında OSPF (alan 0); merkez varsayılan rotayı dağıtır.
- Tek DHCP sunucusu (10.10.30.10); diğer ağlar `ip helper-address` ile adres alır.
- İnternet çıkışı R-MERKEZ üzerinde PAT.
- `MISAFIR-IN`: misafir VLAN iç ağa erişemez, internete çıkabilir.

## 06 — Üniversite Kampüs Ağı

![06](06-universite-kampus-agi/topoloji.png)

**Senaryo.** Beş binalı bir üniversite kampüsü. Personel, öğrenci ve laboratuvar ağları ayrı olacak; tüm binalar tek bir veri merkezinden hizmet alacak ve internete tek noktadan çıkılacak.

**İstenenler**

1. Her binada personel, öğrenci, laboratuvar ve yönetim VLAN'ları
2. Binalar yedekli bir omurga üzerinden bağlansın (LACP)
3. Her bina ayrı bir OSPF alanı olsun ve omurgaya tek özet rota duyursun
4. Adresler veri merkezindeki tek DHCP sunucusundan dağıtılsın
5. Öğrenci ağları yalnızca sunuculara ve internete erişebilsin
6. İnternet çıkışı PAT ile; web ve e-posta sunucuları DMZ'de, statik NAT ile yayınlansın
7. Dışarıdan yalnızca DMZ hizmetlerine erişilebilsin
8. Sunucu adları DNS ile çözülsün (www, ftp)
9. Cihaz yönetimi yalnızca SSH ile ve yalnızca veri merkezinden
10. Tüm cihazlar NTP ile saat eşitlesin ve günlüklerini merkezi sunucuya göndersin

**Çözüm**

Beş bina, bir veri merkezi, DMZ ve internet kenarından oluşan kampüs ağı: 8 yönlendirici, 19 anahtar, 22 PC, 6 sunucu.

**Omurga.** İki omurga anahtarı arasında LACP EtherChannel; tüm yönlendiriciler VLAN 100 (10.0.0.0/24) üzerinden OSPF alan 0'da buluşur. Hızlı yakınsama için hello/dead süreleri 1/4 saniye. EDGE atanmış yönlendirici (DR), R-VERIMERKEZI yedek (BDR).

**Binalar.** Her bina kendi OSPF alanındadır ve omurgaya tek bir /16 özet rota duyurur.

| Bina | OSPF alanı | Adres bloğu | Omurga adresi |
|---|---|---|---|
| Rektörlük | 1 | 10.1.0.0/16 | 10.0.0.11 |
| Mühendislik | 2 | 10.2.0.0/16 | 10.0.0.12 |
| Fen-Edebiyat | 3 | 10.3.0.0/16 | 10.0.0.13 |
| Kütüphane | 4 | 10.4.0.0/16 | 10.0.0.14 |
| Yurt | 5 | 10.5.0.0/16 | 10.0.0.15 |

Her binada bir dağıtım ve iki erişim anahtarı, dört VLAN bulunur: 10 PERSONEL (10.N.10.0/24), 20 OGRENCI (10.N.20.0/24), 30 LAB (10.N.30.0/24), 99 YONETIM (10.N.99.0/24).

**Veri merkezi (alan 200, 10.200.10.0/24).** Merkezi DHCP sunucusu (15 bina havuzu), intranet web sunucusu ve FTP sunucusu. Binalar `ip helper-address` ile adres alır.

**İnternet kenarı.** Kampüs için PAT, DMZ'deki web ve e-posta sunucuları için statik NAT (198.51.100.10 ve .11), `OUTSIDE-IN` ve `DMZ-IN` erişim listeleri.

**Güvenlik.** `OGRENCI-IN`: öğrenci VLAN'ları yalnızca veri merkezi sunucularına ve internete erişir; diğer iç ağlara erişemez. Erişim portlarında port-security ve BPDU guard.

## 07 — Holding – Çok Şubeli WAN

![07](07-holding-cok-subeli-wan/topoloji.png)

**Senaryo.** Genel müdürlüğü ve üç şehirde şubesi olan bir holding. Sahalar internet üzerinden birbirine bağlanacak ve şubeler genel müdürlükteki sunucuları kullanacak.

**İstenenler**

1. Genel müdürlükte beş VLAN ve yedekli dağıtım katmanı (LACP, çift bağlantılı erişim anahtarları)
2. Şubeler genel müdürlüğe GRE tünelleriyle bağlansın
3. Tüneller üzerinde OSPF çalışsın; şubeler birbirine genel müdürlük üzerinden ulaşsın
4. Her saha internete kendi hattından PAT ile çıksın; sahalar arası trafik NAT'a girmesin
5. Genel müdürlükte merkezi DHCP, şubelerde yerel DHCP
6. Misafir ağları iç ağa erişemesin
7. Sunucu adları DNS ile çözülsün (intranet, dosya)
8. Cihaz yönetimi yalnızca SSH ile ve yalnızca bilgi işlem ile yönetim ağından
9. NTP ve merkezi günlük (syslog)

**Çözüm**

Bir genel müdürlük ve üç şubenin servis sağlayıcı ağı üzerinden bağlandığı WAN tasarımı: 8 yönlendirici, 7 anahtar, 13 PC, 3 sunucu.

**Servis sağlayıcı.** Dört yönlendiricili halka (ISP1–ISP4), kendi içinde OSPF 100. ISP3 üzerindeki 8.8.8.8 internet test hedefidir.

**Sahalar.**

| Saha | Adres bloğu | WAN adresi | GM'ye tünel |
|---|---|---|---|
| Genel Müdürlük | 10.1.0.0/16 | 203.0.113.2 | — |
| Ankara | 10.2.0.0/16 | 203.0.113.6 | 172.16.0.0/30 |
| İzmir | 10.3.0.0/16 | 203.0.113.10 | 172.16.0.4/30 |
| Trabzon | 10.4.0.0/16 | 203.0.113.14 | 172.16.0.8/30 |

- Genel müdürlük ile şubeler arasında GRE tünelleri (hub-and-spoke); tüneller üzerinde OSPF alan 0.
- Her sahada internet için PAT; sahalar arası trafik NAT dışında tutulur (`NAT-INTERNET`).
- Genel müdürlük: 5 VLAN (Yönetim Kurulu, Finans, Bilgi İşlem, Sunucu, Yönetim), LACP ile bağlı iki dağıtım anahtarı, her iki dağıtım anahtarına bağlı erişim anahtarları, merkezi DHCP sunucusu.
- Şubeler: Personel ve Misafir VLAN'ları, yönlendiricide yerel DHCP; misafir VLAN iç ağa erişemez.

---

Hazırlayan: Safa Şentürk · [safasenturk.github.io](https://safasenturk.github.io)

# Ubuntu Firewall. UFW և nftables

## Տեսություն

**Firewall-ը** (հայերեն՝ «պատնեշ») համակարգի կամ ցանցի բաղադրիչ է, որը վերահսկում է, թե ինչ մուտքային (incoming) և ելքային (outgoing) կապեր են թույլատրվում։ Սերվերի վրա դա սովորաբար **host-based firewall** է, որն աշխատում է Linux-ի միջուկի (kernel) ներսում՝ դեռ ծրագրերին հասնելուց առաջ ֆիլտրելով ցանցային փաթեթները։

### Ubuntu-ի լռելյայն firewall-ը՝ UFW

Ubuntu-ի վերջին տարբերակներում (ներառյալ 22.04 / 24.04 LTS) լռելյայն firewall-կառավարման գործիքը **UFW**-ն է (**U**ncomplicated **F**irewall՝ «ոչ բարդ firewall»)։ Կարևոր է հասկանալ, որ UFW-ն **frontend** է (միջերես), ոչ թե առանձին firewall. այն պարզ հրամաններով կառավարում է kernel-ի իրական firewall-մեխանիզմը, որպեսզի օգտատերը ստիպված չլինի ուղղակի աշխատել բարդ iptables-ի հետ։

Երկու կարևոր փաստ UFW-ի մասին.

- **Նախապես տեղադրված է** (available by default) Ubuntu-ի բոլոր տեղադրումներում (8.04-ից մինչև այսօր)։
- **Լռելյայն անջատված է** (inactive). Տեղադրելուց հետո բոլոր մուտքային կապերը բաց են, մինչև այն ձեռքով միացնենք։ Սա զգուշավոր քայլ է, որ նոր համակարգը «կողպված» չմնա դեռևս կարգաբերված չլինելու պատճառով. միացնում ենք `sudo ufw enable` հրամանով:

### UFW-ի հենքը՝ nftables

Ubuntu-ի 20.04-ից UFW-ն աշխատում է **nftables**-ի վրա (նախկինում՝ iptables-ի)։ Դա կարևոր է հասկանալ, որ պատասխանենք «ո՞ր firewall-ն է իմ մոտ ակտիվ» հարցին.

- **nftables** (կամ կարճ՝ `nft`) Linux-ի kernel-ի firewall-ի ժամանակակից շրջանակն է (framework), որը նախատեսված է հին **iptables**-ը (legacy xtables) փոխարինելու համար։
- UFW-ն, սակայն, դեռ օգտագործում է **iptables-nft** համատեղելիության շերտը. այս շերտը «թարգմանում» է iptables-ի հրամանները nftables-ի kernel-ի մակարդակի համար։ Դրա արդյունքում.
  - `sudo iptables -L -n -v` հրամանը ցույց է տալիս նույն կանոնները, բայց iptables-ի հին ձևաչափով (սա **wrapper** է, ոչ թե առանձին firewall) ;
  - `sudo nft list ruleset`-ը ցույց է տալիս նույն կանոնները՝ բայց nftables-ի «մաքուր» ձևաչափով՝ հենց այն, ինչ իրականում տեսնում է kernel-ը:

Որպես հիշեցում՝ `nft list ruleset`-ի ելքում կարող եք տեսնել նախազգուշացում.

```
# Warning: table ip filter is managed by iptables-nft, do not touch!
```

Այն ասում է, որ այդ աղյուսակը կառավարվում է iptables-nft-ի միջոցով, բայց ֆիզիկապես գտնվում է nftables-ում։ Ուստի եզրակացությունը մեկն է՝ **միշտ աշխատում է nftables-ը, իսկ UFW-ն ու iptables-ը ընդամենը կառավարման միջերեսներ են։**

### Հիմքում ընկած համակարգը՝ Netfilter, hooks, tables, chains, rules

Kernel-ի firewall-ենթակառուցվածքը կոչվում է **Netfilter**։ nftables-ն այն կարգավորելու հրամանատարն է, իսկ UFW-ն՝ դրանից էլ ավելի վերևում կանգնած հեշտ ինտերֆեյսը։

**Hooks (կեռիկներ):** Փաթեթը Linux-ի ցանցային ստեկում անցնում է 5 հիմնական կետով, և կանոնները կարելի է «կախել» այդ կետերից՝ կախված նրանից, ինչ ենք ուզում անել.

| Hook | Երբ է ակտիվանում | Հիմնական կիրառում |
| --- | --- | --- |
| `PREROUTING` | Փաթեթը հենց մտավ ցանցային քարտից՝ routing-ից առաջ | DNAT (պորտերի/հասցեների վերահասցեավորում) |
| `INPUT` | Փաթեթը հասցեագրված է հենց այս սերվերին | Սերվերի մուտքային տրաֆիկի զտում |
| `FORWARD` | Փաթեթը պետք է անցնի սերվերի միջով (router, bridge) | Տրաֆիկի փոխանցման (forwarding) զտում՝ օր. Docker-ի ցանցերում |
| `OUTPUT` | Փաթեթը ստեղծվում է հենց սերվերի կողմից | Սերվերի ելքային տրաֆիկի զտում |
| `POSTROUTING` | Փաթեթը պատրաստ է դուրս գալ ցանցային քարտից | SNAT/masquerade |

**Tables (աղյուսակներ):** Խմբավորում են շղթաներն ըստ «ընտանիքի» (family).

| Family | Նշանակություն |
| --- | --- |
| `ip` | Միայն IPv4 |
| `ip6` | Միայն IPv6 |
| `inet` | Միավորում է IPv4-ը և IPv6-ը մեկ աղյուսակում (*best practice*) |

**Chains (շղթաներ):** Կանոնների հերթական ցուցակ. երկու տիպ.

| Տիպ | Նկարագրություն | Օրինակ |
| --- | --- | --- |
| Base chain | Ամրացված է hook-ին, ունի `type` և `priority` | `chain input { type filter hook input priority filter; policy drop; }` |
| Regular chain | Համակարգ ստեղծած, hook-ին չի ամրացվում, կանչվում է `jump`-ով | `chain ufw-user-input` (UFW-ի կանոնների շղթան) |

`jump`-ը, կատարելուց հետո, վերադառնում է այն կետը, որտեղից կանչվել էր. իսկ `goto`-ն՝ ոչ (մոռանում է մնացածը)։ UFW-ն ձևավորում է իր `ufw-user-input` regular chain-ները և base chain-երից `jump`-ով «մատնացույց է անում» դրանց պարունակությանը։

**Rules (կանոններ):** Յուրաքանչյուր կանոն = `match` (պայման) + `verdict` (որոշում)։ Օրինակ՝ `ip saddr 104.16.0.0/13 tcp dport 443 accept` նշանակում է՝ «եթե աղբյուրը 104.16.0.0/13 է և պորտը 443-ն է, ապա ընդունիր»։ Verdict-ներն են՝ `accept` (թողնել), `drop` (լուռ գցել), `reject` (գցել և պատասխան ուղարկել), `jump`, `log`։

### Ի՞նչ է ցույց տալիս `type`-ը

Base chain-ը պարտադիր ունի `type` դաշտը, որն ասում է, թե ինչով է զբաղվում շղթան.

| `type` | Դերը | Ինչ verdict-ներ է թույլատրվում |
| --- | --- | --- |
| `filter` | Զտում (`accept`/`drop`/`reject`) | Ամենատարածվածը՝ INPUT/OUTPUT/FORWARD-ի համար |
| `nat` | Հասցեների փոխարկում | `snat`, `dnat`, `masquerade` (միայն նախատեսված hook-երում) |
| `route` | Փաթեթի վերաուղղորդում (հազվադեպ) | `dup`, `fwd` |

`type`-ը աշխատում է նաև `priority`-ի հետ, ինչը որոշում է կատարման հերթականությունը նույն hook-ի վրա. օրինակ՝ NAT-ն աշխատում է filter-ից առաջ՝ որովհետև նախ պետք է փոխել հասցեն, հետո զտել։ Regular chain-երը `type` չունեն՝ քանի որ ժառանգում են իրենց կանչող base chain-ի համատեքստը։

### Sets. մեծ ցուցակների արդյունավետ կառավարում

nftables-ի ամենաուժեղ հնարավորություններից մեկը **sets**-ն է (հավաքածուներ)՝ IP-ների, պորտերի և այլ արժեքների ցուցակ, որը kernel-ում պահվում և որոնվում է որպես մեկ կառուցվածք։ Սա հատկապես օգտակար է մեծ ցուցակների դեպքում՝ օրինակ՝ Cloudflare-ի IP-ների։ Մեկ set-ով մենք գրում ենք **մեկ** կանոն՝ փոխանակ յուրաքանչյուր subnet-ի համար առանձին տող գրելու.

```bash
# set-ի սահմանում և ավելացում
sudo nft add set inet filter cloudflare_v4 { type ipv4_addr; flags interval; }
sudo nft add element inet filter cloudflare_v4 { 104.16.0.0/13, 172.64.0.0/13, 162.158.0.0/15 }
# մեկ կանոն` ամբողջ set-ի համար
sudo nft add rule inet filter input ip saddr @cloudflare_v4 tcp dport { 80, 443 } accept
```

### ct. connection tracking (կապերի հետևում)

`ct`-ն **connection tracking**-ի հապավումն է՝ kernel-ի մեխանիզմ, որը «հիշում է» կապերի վիճակը։ Այն թույլ է տալիս firewall-ին տարբերել՝ «սա վտանգավոր անծանոթն է» (new/invalid) և «սա մեր հին բարեկամն է» (established/related)։ Հիմնական վիճակները.

- `new` — նոր կապ (օրինակ՝ առաջին SYN-փաթեթը) ;
- `established` — արդեն հաստատված կապ՝ տրաֆիկի գերակշիռ մասը (օրինակ՝ սերվերի պատասխան փաթեթները) ;
- `related` — առնչվող կապ (FTP-ի տվյալային կապ, ICMP սխալ) ;
- `invalid` — անվավեր/կեղծ փաթեթ՝ հաճախ հաքերային ;

Գրեթե յուրաքանչյուր firewall-ի առաջին կանոնը `ct state { established, related } accept` է՝ արդեն հաստատված կապերը չմշակելու և արագություն ապահովելու համար։ `invalid`-ները սովորաբար անմիջապես `drop` են արվում։ Connection tracking-ի աղյուսակը կարելի է դիտել՝ `sudo conntrack -L` կամ `cat /proc/net/nf_conntrack` հրամանով։

### Docker-ը և firewall-ը

Docker-ը ինքնուրույն է կառավարում firewall-ի կանոնները՝ iptables/nftables-ի միջոցով (հիմնականում NAT-ը և FORWARD-ը)։ Սա կարևոր նրբություն է.

- Docker-ը ստեղծում է իր `DOCKER`, `DOCKER-FORWARD`, `DOCKER-USER` շղթաները և՝ `FORWARD`-ի `policy drop`-ի ֆոնին՝ ավելացնում է ներքին ցանցերի միջև տրաֆիկի թույլտվություններ։
- Ձեր host firewall-ը (UFW/nftables) չի խանգարում Docker-ի ներքին ցանցերին, բայց վերահսկում է դեպի **host-ի 0.0.0.0** պորտերը եկող մուտքային տրաֆիկը (օրինակ՝ proxy-ի 80/443-ը)։
- Զգուշացում՝ «# Warning: ... do not touch!» տողը նշանակում է, որ այդ աղյուսակը կառավարում է Docker-ը/iptables-nft-ը. ձեռքով կարգավորվող են միայն ձեր աղյուսակները. իսկ Docker-ի աղյուսակները ջնջելը կկանգնեցնի Docker-ի աշխատանքը՝ և ջնջվածն էլ նորից կստեղծվի գործարկման ժամանակ:

### Cloudflare-ի հետ աշխատանքը. իրական IP-ն «թաքցնելու» սխեման

Պրոդաքշն WordPress/nginx սերվերները հաճախ ծածկվում են **Cloudflare**-ով՝ DDoS-ից պաշտպանության համար։ Այս սխեմայում.

- Հաճախորդի հարցումը գնում է Cloudflare, այլ ոչ թե ուղիղ ձեր սերվերի IP-ին.
- Cloudflare-ը զտում է այն և հարցումն ուղարկում ձեր սերվերի իրական IP-ին՝ **Cloudflare-ի IP-ներից** (ցուցակը՝ `https://www.cloudflare.com/ips-v4` և `ips-v6`),
- Ձեր firewall-ը կարող է սահմանել, որ 80/443 պորտերը ընդունվեն **միայն** Cloudflare-ի IP-ներից՝ այսպես հաքերը չի կարող ուղիղ միանալ ձեր իրական IP-ին (կհայտնվի firewall-ի DROP-ի առաջ),
- SSH-ը (22) պահվում է բաց՝ ձեր կառավարման կապը չկորցնելու համար.

Ստուգման ժամանակ `nmap`-ը ցույց է տալիս, որ 22-ը `open` է (հասանելի է), իսկ 80-ը/443-ը՝ `filtered` (ոչ-Cloudflare աղբյուրից միանալու դեպքում)՝ հենց այդ «միայն Cloudflare» քաղաքականության արդյունքն է։

## Հիմնական հրամաններ

| Հրաման | Նպատակ | Սպասվող արդյունք | Անվտանգության ռիսկ |
| --- | --- | --- | --- |
| `sudo ufw status` | UFW-ի կարգավիճակի ստուգում | `Status: inactive` կամ `active` + կանոններ | Չկա |
| `sudo ufw allow ssh` | Բացել 22-րդ պորտը (SSH) բոլորի համար | 22-ը թույլատրված է | SSH-ը հասանելի է արտաքին աշխարհին. խորհուրդ՝ IP-ով սահմանափակել |
| `sudo ufw allow from <IP> to any port 22 proto tcp` | SSH-ը սահմանափակել կոնկրետ IP/ցանցով | Աղբյուրով սահմանափակված կանոն | Ցածր՝ միայն ձեր IP-ը |
| `sudo ufw enable` | Միացնել UFW-ն | «Firewall is active» | **Կարող է կտրել SSH կապը**՝ եթե 22-ը առաջ բացած չէ |
| `sudo ufw allow 80/tcp` / `443/tcp` | Բացել HTTP/HTTPS | Կանոն | Բոլորին բացելը մերկացնում է իրական IP-ն. ավելի լավ՝ միայն Cloudflare-ի IP-ներով |
| `sudo nft list ruleset` | Դիտել kernel-ի բոլոր ակտիվ կանոնները | nftables-ի «մաքուր» ցուցակ | Չկա (միայն կարդալ) |
| `sudo nft --handle list ruleset` | Դիտել կանոններն իրենց `handle`-ներով | Յուրաքանչյուր կանոնի ID | Չկա (միայն կարդալ) |
| `sudo nft delete rule inet filter input handle <ID>` | Ջնջել կոնկրետ կանոնը | Կանոնը հեռացված | Սխալ ID-ը ջնջում է պաշտպանիչ կանոն |
| `sudo nft flush ruleset` | Ջնջել **բոլոր** կանոնները | Բոլորը մաքրված՝ բոլոր պորտերը բաց | **Շատ վտանգավոր**՝ միայն վթարային դեպքում |
| `sudo nft -c -f /etc/nftables.conf` | Կոնֆիգի շարահյուսության ստուգում (dry-run) | Սխալ/հաջող հաղորդում | Չկա |
| `sudo nft -f /etc/nftables.conf` | Կիրառել կոնֆիգը ֆայլից | Կանոնները թարմացված | Կտրում է կապը՝ եթե SSH-ը բաց չէ կոնֆիգում |

## Փորձարկում (Lab)

**Միջավայր.** Սովորական Linux VM (Ubuntu 22.04+) կամ կոնտեյներ՝ SSH-մուտքով։ Թիրախը՝ կառավարել nftables-ը ուղիղ՝ `inet` աղյուսակով՝ ապահովելով, որ SSH-ը մնա բաց, իսկ մնացածը՝ փակ։ Օգտագործվող ցանցերը (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) **TEST-NET** (RFC 5737)՝ ուսումնական, ինտերնետում չուղղորդվող հասցեներ՝ միայն թեստավորման համար. փոխարինեք դրանք ձեր իրական IP/ցանցով։

1. **Ստուգեք, ինչ back-end է օգտագործում UFW-ն**
   ```bash
   sudo ufw status
   ls -l /etc/ufw/
   ```
   Նկատեք՝ `/etc/ufw/*.rules` ֆայլերը. UFW-ն կանոնները պահում է տեքստային ֆայլերում՝ աշխատելու՝ iptables-restore/nft-ի միջերեսի հետ։

2. **Միացրեք UFW-ն և դիտեք iptables/nft-ի տարբերությունը**
   ```bash
   sudo ufw allow from 192.0.2.0/24 to any port 22 proto tcp   # TEST-NET` ձեր IP-ով
   sudo ufw enable
   sudo iptables -L -n -v | head -n 30   # UFW-ի կանոնները` iptables-ի ձևաչափով
   sudo nft list ruleset | head -n 50    # նույնը` nft-ի «մաքուր» ձևաչափով
   ```

3. **Ստեղծեք սեփական nftables-ի կոնֆիգ** `/etc/nftables.conf`-ում (նախ `backup)`
   ```bash
   sudo cp /etc/nftables.conf /etc/nftables.conf.bak
   ```
   և տեղադրեք `inet filter` աղյուսակ `input` շղթա՝ `policy drop`-ով `որտեղ` առաջին հերթին `ct state established,related`-ը, loopback-ը և SSH-ը թույլատրվում են։ Ապա.
   ```bash
   sudo nft -c -f /etc/nftables.conf   # dry-run` շարահյուսության ստուգում
   sudo nft -f /etc/nftables.conf      # կիրառել
   sudo nft --handle list chain inet filter input
   ```

4. **Ստուգեք, արդյոք SSH-ը մնաց բաց, իսկ արտաքինից (այլ IP) մուտքը՝ փակ**
   ```bash
   sudo nft list chain inet filter input
   ss -tlnp | grep -E ':(22|80|443)'
   ```

Եզրակացություն. Lab-ը ցույց է տալիս, որ UFW-ն և nftables-ը նույն kernel-ի firewall-ի երկու միջերեսն են և որ `/etc/nftables.conf`-ի կիրառումը պաշտպանում է SSH-ը՝ միաժամանակ փակելով մնացածը:

! Best Practices
    - Firewall-ի ցանկացած փոփոխությունից առաջ (հատկապես `ufw enable` կամ `nft -f`) համոզվեք, որ SSH (22) կանոնն արդեն կա, և ունեք out-of-band (կոնսոլային) մուտք՝ կապը չկորցնելու համար:
    - `INPUT`-ի/`FORWARD`-ի համար դրեք `policy drop`, իսկ `OUTPUT`-ը՝ `accept` (կամ՝ եթե խիստ եք՝ `drop` + միայն անհրաժեշտ ելքային պորտերը):
    - `ct state { established, related } accept` դրեք `input`-ի **ամենասկզբում** (կատարողականության համար):
    - Cloudflare-ի կամ մեծ IP-ցուցակների համար օգտագործեք nftables **sets**, ոչ թե յուրաքանչյուր subnet-ի համար առանձին կանոն:
    - Կանոնները պահեք ֆայլում և կիրառեք նախ `nft -c` (dry-run), այնուհետև `-f`-ով live:
    - `nft flush ruleset`-ը մի օգտագործեք պրոդաքշնում՝ այն ջնջում է ամբողջ ruleset-ը:

## Իրական DevOps իրավիճակ

### Ախտանիշ

Աշխատող սերվերում (Ubuntu, Docker + nginx) որոշվեց 80/443 պորտերը թողնել միայն Cloudflare-ի համար։ Սակայն `sudo nmap -sS -Pn -p 22,80,443 <server_ip>`-ի արդյունքում 22-ը `open` է, 80/443-ը՝ նույնպես `open` (ոչ `filtered`), և մյուս պորտերը ևս երևում են արտաքինից՝ չնայած Cloudflare-ի «միայն» կանոններին, որոնք պետք է ցույց տային `filtered`:

### Ախտորոշում

1. Ստուգեք UFW-ի կարգավիճակը.
   ```bash
   sudo ufw status
   ```
   Եթե `inactive` է՝ firewall-ն ընդհանրապես չի աշխատում՝ ինչն էլ բացատրում է ամեն ինչ:

2. Դիտեք kernel-ի ակտիվ կանոնները՝ nftables-ով.
   ```bash
   sudo nft list ruleset
   ```
   Փնտրեք արդյոք կան `policy accept` ունեցող **դատարկ** `INPUT` base chain-եր `ip filter`/`ip6 filter` աղյուսակներում (հին iptables/UFW-ի մնացորդներ)։ Եթե կան՝ դրանք նույն hook-ի վրա **շրջանցում են** ձեր `inet filter`-ի `policy drop`-ը՝ ընդունելով ամեն ինչ, ինչու է nmap-ը ցույց տալիս `open`:

3. Համեմատեք `sudo iptables -L -n -v`-ի և `sudo nft list ruleset`-ի ցուցադրությունը՝ բացահայտելով՝ որ աղյուսակներն են Docker-ի (և ունեն «# Warning ... do not touch!»), իսկ որոնք՝ ձերը:

### Լուծում

- Եթե UFW-ն `inactive` էր միացրեք նախ SSH-ով ապահովելով 22-ը, հետո՝ Cloudflare-ի IP-ներից՝ 80/443:
  ```bash
  sudo ufw allow from 203.0.113.0/24 to any port 22 proto tcp   # TEST-NET` ձեր IP/ցանցով
  sudo ufw allow 80/tcp
  sudo ufw allow 443/tcp   # կամ` միայն Cloudflare-ի IP-ներից (set մոտեցում)
  sudo ufw enable
  ```
- Եթե դատարկ `policy accept` աղյուսակները (մնացորդներ) խանգարում են ձեր `inet filter`-ին՝ ջնջեք դրանք**միայն** այն դեպքում, երբ վստահ եք, որ դրանք ձերն են (ոչ Docker-ի `#Warning` աղյուսակները):
  ```bash
  sudo nft delete table ip filter
  sudo nft delete table ip6 filter
  ```
  Ապա ստուգեք, որ 80/443 այժմ `filtered` են, իսկ SSH-ը՝ `open`:
  ```bash
  sudo nft list chain inet filter input
  ```
- Վթարային դեպքում (կապը կորել է)՝ կոնսոլից՝ `sudo nft flush ruleset` + `sudo iptables -P INPUT ACCEPT`, ապա նորից կիրառեք աշխատող ֆայլը:

### Կանխարգելում

- Պահեք աշխատող կանոնակազմը `/etc/nftables.conf`-ում և `systemctl enable nftables`-ով ապահովեք boot-ի ժամանակ վերականգնումը.
- Բոլոր փոփոխություններից առաջ՝ dry-run (`nft -c`) և backup SSH/կոնսոլ.
- Պարբերաբար, որոշ կայուն IP-ից՝ nmap-ի սկան՝ որպես ռեգրեսիոն թեստ՝ ակնկալելով՝ 22-ը՝ `open`, իսկ 80/443-ը՝ `filtered` ոչ-Cloudflare-ից:
- Թարմացրեք runbook-ը՝ նկարագրելով՝ որ աղյուսակի/շղթայի վրա է հիմնված պետությունը, և ինչպես զատել nftables-ի և Docker-ի աղյուսակները:

## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչ է UFW-ն, և ո՞ր firewall-ն է իրականում աշխատում Ubuntu-ի նոր տարբերակներում (mid-level)

UFW-ն (Uncomplicated Firewall, «ոչ բարդ firewall») Ubuntu-ի լռելյայն firewall-ի **կառավարման frontend** գործիքն է՝ նախապես տեղադրված, բայց լռելյայն `inactive` վիճակում՝ մինչև `sudo ufw enable`-ը։ Իրական firewall-մեխանիզմը, որը ֆիլտրում է փաթեթները, kernel-ի **Netfilter**-ն է՝ ժամանակակից Ubuntu-ում (20.04-ից) կարգավորվող՝ **nftables**-ի միջոցով։ UFW-ն օգտագործում է `iptables-nft` համատեղելիության շերտը, ուստի նույն կանոնները երևում են և՛ `iptables -L`-ում, և՛ `nft list ruleset`-ում՝ տարբեր ձևաչափով։ Իսկապես ակտիվը՝ nftables-ն է (kernel-ի մակարդակ), մինչդեռ UFW-ն ու iptables-ը միայն կառավարման միջերեսներ են։

### Ինչպե՞ս եք տարբերակում «open» և «filtered» պորտը, և ի՞նչ է այն ասում ձեր firewall-ի մասին (senior-level)

Nmap-ի համար՝ «open» նշանակում է, որ պորտը պատասխանում է SYN-ACK-ով (հասանելի է), «filtered»՝ որ firewall-ը DROP է անում (կամ պատասխան չի գալիս), ուստի պորտի վիճակն անորոշ է, բայց մուտքը մերժված է։ Այս երկուսի տարբերությունն ախտորոշիչ արժեք ունի. օրինակ՝ եթե 80/443-ը `filtered` են ձեր (ոչ-Cloudflare) IP-ից՝ ապա «միայն Cloudflare» քաղաքականությունն աշխատում է. իսկ եթե դրանք `open` են՝ ապա scope-ի սահմանափակումը թերի է (օրինակ՝ դատարկ `policy accept` base chain-ը nftables-ում, որը շրջանցում է ձեր `inet filter`-ը)։ Senior-ը կսկսի `sudo nft list ruleset`-ից՝ փնտրելով նման աղյուսակներ, և՝ միայն ապա՝ ջնջել service-ին չպատկանող, կասկածելի base chain-երը.

## Ինքնաստուգում

1. Ի՞նչ է UFW-ն, և ինչո՞ւ է այն լռելյայն `inactive` Ubuntu-ում՝ չնայած, որ «firewall-ը» տեղադրված է։
2. Ի՞նչ back-end է օգտագործում UFW-ն Ubuntu-ի 20.04+-ում՝ nftables, թե iptables, և ի՞նչ դեր ունի `iptables-nft`-ը.
3. Նշեք Netfilter-ի 5 hooks-ը և ինչի համար են դրանցից յուրաքանչյուրը (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING)։
4. Ի՞նչ տարբերություն կա Base chain-ի և Regular chain-ի միջև nftables-ում, և ի՞նչ դեր ունի `jump`-ը.
5. Ի՞նչ է `ct state { established, related }`-ը, և ինչո՞ւ են այն դնում input-ի ամենասկզբում։
6. Ի՞նչ է nftables-ի set-ը, և ինչո՞ւ է մեկ set ավելի արդյունավետ, քան Cloudflare-ի IP-ների համար առանձին կանոնների երկար ցուցակը։
7. Ինչո՞ւ է `sudo nft flush ruleset`-ը «վտանգավոր», և ինչպե՞ս է կոնֆիգը վերականգնվում boot-ից հետո (`systemctl enable nftables`)։
8. Ի՞նչ է ցույց տալիս nmap-ի `filtered` վիճակը, և ինչպե՞ս է այն կապված «միայն Cloudflare» firewall-քաղաքականության հետ.

## Հաջորդ քայլեր

Շարունակիր [Networks](../networking/index.md) բաժնով՝ հետագա՝ TCP/IP, routing, DNS և TLS-ի մասին. firewall-ը ցանցային շղթայի «վերջին դարպասն» է դեպի սերվեր։ Ավելի խոր՝ [Linux-ի ցանցային կարգավորումներ](networking.md)՝ ինտերֆեյսների ու routing-ի համար։ Քանի որ Docker-ն ինքնուրույն է կառավարում iptables/nftables-ը, ֆայլը սերտորեն կապված է [Docker և Kubernetes](../docker-kubernetes/index.md) բաժնի հետ՝ արժե կարդալ՝ նախքան կոնտեյներների ցանցային պորտերի մեկուսացումը պրակտիկայում կիրառելը։
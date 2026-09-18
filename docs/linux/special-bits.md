# SUID, SGID և Sticky bit

## Տեսություն

Linux-ում ֆայլերն ու գրացանակները (directories) ունեն 9 հիմնական թույլտվության բիթ՝ read (`r`), write (`w`), execute (`x`)՝ երեք խմբի համար. տեր (owner/user), խումբ (group), մնացածը (others)։ Այս 9 բիթերին գումարվում է ևս **3 հատուկ բիթ**, որոնք փոխում են համակարգի վարքը՝ **SUID** (Set User ID), **SGID** (Set Group ID) և **Sticky bit**։ Դրանք հանդիպում են երկու տեղում՝ `chmod`-ի թվային 4-նիշ ձևաչափում (առաջին նիշը) և `ls -l`-ի ելքի `x` դիրքերում՝ `s`, `S`, `t`, `T` տառերով։

Կարևոր հիմք. այս բիթերը **չեն ավելացնում** նոր թույլտվություն, այլ փոխում են **ում անունից** է կատարվում գործողությունը (SUID/SGID) կամ **ում ֆայլերը** կարելի է ջնջել (sticky)։ Դրանք permissions-ի մոդելի մաս են և չեն շրջանցում այն. արդյունավետ իրավունքները դեռ որոշվում են `r`/`w`/`x` բիթերով։

### SUID (Set User ID)

**Ինչ է անում.** երբ ֆայլը executable է և ունի SUID բիթ, ապա գործարկման պահին պրոցեսը ստանում է **ֆայլի տիրոջ** (owner) իրավունքներ, ոչ թե նրան, ով գործարկում է։ Դա կոչվում է effective UID (EUID) փոխարկում. իրական UID-ը (RUID) մնում է գործարկողի, իսկ effective-ը դառնում է ֆայլի տիրոջը։

Դասական օրինակ՝ `/usr/bin/passwd`։

```text
-rwsr-xr-x 1 root root 68208 ... /usr/bin/passwd
```

`passwd`-ը գործարկվում է սովորական օգտատիրոջ կողմից, բայց `/etc/shadow` ֆայլը կարդալ/գրել կարող է միայն root-ը։ SUID-ի շնորհիվ `passwd`-ը աշխատում է root-ի EUID-ով և կարողանում է թարմացնել shadow-ը։

**Նրբություններ.**

- SUID-ն իմաստ ունի **միայն ֆայլերի** վրա (ոչ գրացանակների)։ Linux-ում գրացանակի SUID բիթը անտեսվում է։
- Եթե owner-ի `x` բիթը դրված չէ, `ls -l`-ը ցույց է տալիս **մեծատառ `S`** (`rwS`), որը նշանակում է. SUID բիթը դրված է, բայց ֆայլը executable չէ. այդպիսի բիթը գործնականում չի աշխատի։
- Linux kernel-ը **չի կատարում** shell սկրիպտերի SUID-ն (ինչպես նաև `#!`-ով սկսվող ինտերպրետավորվող սկրիպտների)։ Այնուամենայնիվ, եթե binary-ն ինքն է կանչում այլ ծրագիր, այդ ծրագիրը կարող է դիտարկել ժառանգված EUID-ը. այստեղից է առաջանում **PATH hijacking** ռիսկը, երբ SUID ծրագիրն օգտագործում է հարաբերական ուղի կամ չմաքրված `PATH`։
- Սա ամենամեծ անվտանգության ռիսկերից է. SUID root binary-ի ցանկացած bug (buffer overflow, անապահով ֆայլի գրառում) վերածվում է **privilege escalation**-ի։ Այդ իսկ պատճառով SUID root ֆայլերի ցանկը պետք է պարբերաբար աուդիտ անել և նվազագույնի հասցնել։

### SGID (Set Group ID)

SGID-ն ունի երկու տարբեր իմաստ՝ կախված նրանից՝ ֆայլի՞, թե՞ գրացանակի վրա է։

**ա) Ֆայլի վրա.** executable ֆայլը գործարկվում է **ֆայլի խմբի** իրավունքներով (effective GID)։ Նույն տրամաբանությամբ, ինչ SUID-ը, բայց group-ի համար։ Այսօր հազվադեպ է հանդիպում և նույնպես համարվում է անվտանգության ռիսկ։

**բ) Գրացանակի վրա (հիմնական գործնական օգտագործումը).** այդ գրացանակում ստեղծված **նոր ֆայլերն ու ենթագրացանակները** ժառանգում են **գրացանակի խումբը**, ոչ թե ստեղծողի հիմնական (primary) խումբը։ Սա **shared folder**-ի կարևորագույն մեխանիզմն է. առանց SGID-ի մի քանի օգտատերեր, որոնք գրում են նույն գրացանակում, ստեղծում են տարբեր group-ներով ֆայլեր, և մյուսները չեն կարողանում կարդալ/գրել դրանք։ SGID-ով բոլոր ֆայլերը մնում են մեկ ընդհանուր խմբի տիրապետության տակ։

```bash
sudo mkdir -p /srv/shared/project
sudo chgrp devteam /srv/shared/project
sudo chmod 2775 /srv/shared/project      # rwxrwsr-x + SGID
sudo chmod g+s /srv/shared/project       # համարժեք՝ սիմվոլիկ ձև
```

Այժմ `devteam` խմբի ցանկացած անդամ այստեղ ստեղծած ֆայլը կունենա `devteam` խումբ՝ անկախ իր primary group-ից։

**Նրբություններ.**

- Գրացանակի SGID-ն **ժառանգվում է** ենթագրացանակների վրա (նրանք նույնպես ստանում են SGID բիթ և նույն խումբը)։
- SGID-ն չի ազդում **umask**-ի վրա. նոր ֆայլի `r`/`w`/`x` բիթերը դեռ որոշում է umask-ը (տիպիկ `002` shared folder-ի համար)։ Սովորական umask `022`-ով նոր ֆայլերը չեն լինի group-writable՝ նույնիսկ SGID-ով. shared folder-ում պետք է `umask 002`։
- Եթե group-ի `x` բիթը բացակայում է, `ls -l`-ը ցույց է տալիս **մեծատառ `S`** (`rwS`), որը նույնպես գործնականում անիմաստ է։

### Sticky bit

Sticky bit-ը այսօր իմաստ ունի **միայն գրացանակների** վրա (ֆայլերի վրա այն պատմական ժամանակներում էր օգտագործվում և ժամանակակից kernel-երը անտեսում են)։ Երբ գրացանակն ունի sticky bit, ֆայլ **ջնջել կամ վերանվանել կարող է միայն՝**

1. ֆայլի տերը,
2. գրացանակի տերը,
3. root-ը։

Մնացած բոլորի համար գրացանակում գրելը թույլատրված է, բայց ուրիշի ֆայլը ջնջելը՝ ոչ։

Դասական օրինակ՝ `/tmp`։

```text
drwxrwxrwt 1 root root 4096 ... /tmp
```

Բոլորը կարող են գրել `/tmp`-ում (ինչպես նաև ստեղծել ֆայլեր), բայց ոչ ոք չի կարող ջնջել ուրիշի ֆայլերը։ Իրավական մոդելը կապված է գրացանակի `w` + `x` բիթերի հետ. ֆայլ ջնջելու համար անհրաժեշտ է **գրացանակի** `w` + `x` (ոչ թե ֆայլի), ապա sticky-ն ավելացնում է տիրոջ ստուգումը։

**Նրբություններ.**

- Sticky bit-ը **չի արգելում** ուրիշի ֆայլը կարդալ կամ փոփոխել, եթե ֆայլի `r`/`w` բիթերը դա թույլ են տալիս. այն սահմանափակում է միայն **ջնջելը/վերանվանելը**։
- Եթե others-ի `x` բիթը բացակայում է, `ls -l`-ը ցույց է տալիս **մեծատառ `T`** (`rwT`)։
- Սա առաջին պաշտպանության գիծն է միայն. `/tmp`-ում ուրիշի ֆայլի բովանդակությունը կարող է դեռ ընթեռնելի լինել։ Համատեղ ժամանակավոր գրացանակների համար ավելի ապահով է օգտագործել per-user գրացանակներ կամ `TMPDIR`։
- `nosuid`/`noexec`/`nodev` mount option-ների հետ միասին sticky-ն սովորական hardening միջոց է `/tmp`-ի և կոնտեյներների volume-ների համար։

## Հիմնական հրամաններ

| Հրաման | Նպատակ |
| --- | --- |
| `ls -l file` | Տեսնել հատուկ բիթերը `s`/`S`/`t`/`T` տառերով |
| `chmod u+s file` | Դնել SUID ֆայլի վրա (նպատակահարմար է միայն executable binary-ի) |
| `chmod u-s file` | Հանել SUID-ն |
| `chmod 4755 file` | Թվային ձևով SUID + `rwxr-xr-x` |
| `chmod g+s dir` | Դնել SGID գրացանակի վրա (shared folder) |
| `chmod g-s dir` | Հանել SGID-ն |
| `chmod 2775 dir` | Թվային ձևով SGID + `rwxrwsr-x` |
| `chmod +t dir` | Դնել sticky bit |
| `chmod -t dir` | Հանել sticky bit-ը |
| `chmod 1777 dir` | Թվային ձևով sticky + `rwxrwxrwx` (ինչպես `/tmp`) |
| `stat -c '%A %a %U %G %n' file` | Ցույց տալ թույլտվությունները և սիմվոլիկ, և՛ octal ձևով |
| `find / -perm -4000 -type f 2>/dev/null` | SUID ֆայլերի աուդիտ ամբողջ համակարգում |
| `find / -perm -2000 -type f 2>/dev/null` | SGID ֆայլերի աուդիտ |
| `find / -perm -1000 -type d 2>/dev/null` | Sticky bit ունեցող գրացանակների ցանկ |
| `sudo mount -o remount,nosuid /tmp` | Ժամանակավորապես անջատել SUID-ն mount-ի վրա (եթե partition-ը առանձին է) |

Թվային ձևաչափի տրամաբանությունը. `4xxx` SUID, `2xxx` SGID, `1xxx` sticky, և դրանք **գումարվում են**՝ `chmod 6755` = SUID + SGID + `rwxr-xr-x`, `chmod 3775` = SGID + sticky + `rwxrwxr-x`։

## Փորձարկում (Lab)

Միջավայր՝ ցանկացած Linux VM կամ Ubuntu container՝ `sudo` իրավունքով։ Փորձարկումները կատարվում են առանձին թեստային գրացանակներում, որպեսզի համակարգի ֆայլերը չփոփոխվեն։

```bash
# 1. Ստեղծենք թեստային օգտատերեր և ընդհանուր խումբ
sudo groupadd -f devteam
sudo useradd -m -G devteam alice
sudo useradd -m -G devteam bob

# 2. SUID-ի տարբերությունը. կրկնօրինակենք passwd-ը թեստային ուղու վրա
sudo cp /usr/bin/passwd /tmp/passwd-copy
ls -l /tmp/passwd-copy          # -rwxr-xr-x  (առանց SUID)
ls -l /usr/bin/passwd           # -rwsr-xr-x  (SUID, տերը root)
sudo chmod u+s /tmp/passwd-copy
ls -l /tmp/passwd-copy          # -rwsr-xr-x  (s = SUID դրված է)
sudo chmod u-s /tmp/passwd-copy
ls -l /tmp/passwd-copy          # -rwxr-xr-x  (s հանված է)

# 3. SUID առանց x բիթի. մեծատառ S
touch /tmp/noexec-file
sudo chmod 4644 /tmp/noexec-file
ls -l /tmp/noexec-file          # -rwSr--r--  (S = SUID կա, բայց x չկա. անիմաստ է)

# 4. SGID գրացանակի վրա. shared folder
sudo mkdir -p /srv/shared/project
sudo chgrp devteam /srv/shared/project
sudo chmod 2775 /srv/shared/project
ls -ld /srv/shared/project      # drwxrwsr-x  (s = SGID group-ի դիրքում)

# 5. Ստուգենք խմբի ժառանգումը. ֆայլերը ստանում են devteam խումբը
sudo -u alice touch /srv/shared/project/alice-file
sudo -u bob   touch /srv/shared/project/bob-file
ls -l /srv/shared/project       # երկուսն էլ alice/bob ունեն, բայց group-ը devteam է

# 6. Համեմատության համար՝ առանց SGID-ի
sudo mkdir -p /srv/nosgid
sudo chmod 777 /srv/nosgid
sudo -u alice touch /srv/nosgid/alice-file
ls -l /srv/nosgid               # group-ը alice-ի primary group-ն է, ոչ թե devteam
```

Սպասվող արդյունք. SGID ունեցող գրացանակում նոր ֆայլի `group` դաշտը ցույց է տալիս `devteam`, իսկ առանց SGID-ի՝ տվյալ օգտատիրոջ primary group-ը։ Այս տարբերությունն է shared folder-ի ամբողջ իմաստը։

```bash
# 7. Sticky bit. ուրիշի ֆայլը ջնջելը արգելվում է
mkdir -p /tmp/sticky-demo
chmod 1777 /tmp/sticky-demo
ls -ld /tmp/sticky-demo          # drwxrwxrwt  (t = sticky others-ի դիրքում)

sudo -u alice touch /tmp/sticky-demo/alice-file
sudo -u bob rm /tmp/sticky-demo/alice-file      # Operation not permitted
sudo -u bob touch /tmp/sticky-demo/bob-file     # սեփական ֆայլը՝ ստեղծվում է
sudo -u bob rm /tmp/sticky-demo/bob-file        # իր ֆայլը՝ ջնջվում է

# 8. Sticky-ն առանց x բիթի → մեծատառ T
chmod 1776 /tmp/sticky-demo
ls -ld /tmp/sticky-demo          # drwxrwxrwT

# 9. Աուդիտ. ինչ SUID/SGID/sticky ֆայլեր կան համակարգում
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null
find / -perm -1000 -type d 2>/dev/null
stat -c '%A %a %U %G %n' /usr/bin/passwd
```

Մաքրում. փորձարկումից հետո հեռացրու `sudo rm -rf /tmp/passwd-copy /tmp/noexec-file /tmp/sticky-demo /srv/shared /srv/nosgid`-ով ստեղծված ամեն ինչ, այլապես թեստային օգտատերերն ու SGID-ով գրացանակները կմնան համակարգում։

!!! tip "Best Practices"
    - SUID-ը դիր միայն այն binary-ների վրա, որոնք իսկապես պետք են (սովորաբար՝ միայն փաթեթների կողմից տրամադրվածները)։
    - Shared folder-ների համար օգտագործիր SGID + `umask 002`-ը համատեղ. SGID-ն տալիս է խումբը, umask-ը՝ գրելու իրավունքը։
    - Sticky bit-ը պահպանիր այն գրացանակներում, որտեղ գրելու արտոնություն ունեն բազմաթիվ օգտատերեր (`/tmp`, `/var/tmp`)։
    - SUID/SGID ֆայլերի ցանկը պարբերաբար աուդիտ արա և փոփոխությունները գրանցիր monitoring-ով (AIDE, `auditd`)։
    - Ժամանակավոր գրացանակների համար, որտեղ exec-ի արգելք է պետք, միավորիր sticky-ն `noexec`/`nosuid` mount option-ների հետ։

## Իրական DevOps իրավիճակ

### Ախտանիշ

CI/CD deployment-ից հետո application-ը (սովորական `app` օգտատիրոջ տակ) կարողանում է ֆայլեր ստեղծել shared `/srv/shared/releases` գրացանակում, բայց report-ների generator-ը (մեկ այլ օգտատեր՝ `reporter`, նույն `devteam` խմբում) ստանում է «Permission denied», երբ փորձում է թարմացնել այնտեղի ֆայլերը։

### Ախտորոշում

1. Ստուգիր գրացանակի և ֆայլերի տերին ու խումբը.
   ```bash
   ls -ld /srv/shared/releases
   ls -l /srv/shared/releases | head
   ```
2. Համոզվիր, որ `reporter`-ը իրոք `devteam` խմբում է (խմբի անդամությունը կիրառվում է նոր login-ից. `id reporter`-ը ցույց կտա)։
3. Ստուգիր deployment-ի պրոցեսի umask-ը.
   ```bash
   sudo -u app umask
   ```
   Եթե արդյունքը `0022` է, ապա նոր ֆայլերը կստանան `-rw-r--r--` (խումբը չի կարողանա գրել)։ Եթե ֆայլերի `group` դաշտը `app`-ն է, ապա գրացանակի վրա SGID բիթը բացակայում է։

### Լուծում

```bash
# 1. Գրացանակին տալ ընդհանուր խումբ և SGID
sudo chgrp devteam /srv/shared/releases
sudo chmod 2775 /srv/shared/releases

# 2. Ուղղել արդեն գոյություն ունեցող ֆայլերի խումբը և գրելու իրավունքը
sudo chgrp -R devteam /srv/shared/releases
sudo chmod -R g+rwX /srv/shared/releases

# 3. Ծառայության umask-ը դարձնել 002 (systemd)
sudo systemctl edit app.service
# [Service]
# UMask=0002
sudo systemctl daemon-reload
sudo systemctl restart app.service
```

`chmod g+rwX`-ի `X`-ը կարևոր է. այն `x` բիթը ավելացնում է միայն գրացանակներին (և արդեն executable ֆայլերին), ոչ թե բոլոր հասարակ ֆայլերին։

### Կանխարգելում

- Shared գրացանակները ստեղծիր infrastructure-ի կոդով (Ansible/`cloud-init`)՝ SGID-ը և խումբը սկզբից դրված, ոչ թե ձեռքով ուղղումով։
- Ծառայությունների umask-ը սահմանիր systemd unit-ի `UMask=`-ով, որ ֆայլերը լռելյայն չստանան `644`։
- SUID/SGID ֆայլերի ցանկը գրանցիր baseline-ով և պարբերաբար համեմատիր `find / -perm -4000`-ի արդյունքի հետ. նոր SUID ֆայլը կարող է compromise-ի ազդանշան լինել։
- `/tmp`-ի և կոնտեյներների volume-ների վրա պահիր sticky-ն, իսկ հնարավորության դեպքում՝ նաև `nosuid,noexec` mount option-ները։

## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչ է անում SGID բիթը գրացանակի վրա, և ինչո՞ւ է դա կարևոր shared folder-ում (mid-level)

Գրացանակի SGID-ն այդ գրացանակում ստեղծվող նոր ֆայլերին ու ենթագրացանակներին տալիս է ոչ թե ստեղծողի primary group-ը, այլ գրացանակի group-ը։ Առանց դրա՝ մի քանի օգտատերեր, գրելով նույն գրացանակում, ստեղծում են տարբեր group-ներով ֆայլեր, և միմյանց ֆայլերը չեն կարողանում կարդալ կամ թարմացնել։ SGID + `umask 002`-ը ստանդարտ պատասխանն է shared folder-ի համար, և այն սովորաբար լրացվում է sticky bit-ով, եթե գրացանակում գրելու իրավունք ունի ավելի լայն շրջանակ։

### Ինչո՞ւ է SUID root binary-ը համարվում privilege escalation-ի ռիսկ, և ինչպե՞ս կմեղմեիր այն production-ում (senior-level)

SUID root binary-ի ցանկացած անվտանգության թերություն (buffer overflow, անապահով ժամանակավոր ֆայլ, չմաքրված `PATH`-ով արտաքին ծրագրի կանչ) վերածվում է root-ի իրավունքների ձեռքբերման, քանի որ պրոցեսն աշխատում է EUID=0-ով։ Մեղմումը բազմաշերտ է. (1) SUID root ցանկը նվազագույնի հասցնել և աուդիտ անել `find / -perm -4000 -type f`-ով baseline-ի համեմատ (AIDE, `auditd`)։ (2) Ամբողջական root-ի փոխարեն օգտագործել նեղ մեխանիզմներ՝ Linux capabilities (`setcap`), sudo-ի կանոններ կամ polkit։ (3) Մեկուսացնել systemd-ի sandboxing-ով (`NoNewPrivileges=yes`, `ProtectSystem=strict`, `CapabilityBoundingSet=`) և AppArmor/SELinux-ով։ (4) `/tmp`, `/home` և կոնտեյներների volume-ները mount անել `nosuid,nodev,noexec`-ով։ (5) Պահել `fs.suid_dumpable=0`, որ SUID պրոցեսի core dump-ը չարտահոսի գաղտնիքները։ Այս ամենը կրկնվող գործընթաց է, ոչ թե մեկանգամյա կարգավորում։

## Ինքնաստուգում

1. Ինչ տարբերություն կա փոքրատառ `s`-ի և մեծատառ `S`-ի միջև `ls -l`-ի ելքում
2. Ինչո՞ւ SUID shell սկրիպտի վրա չի աշխատում
3. Ինչպե՞ս կստուգես, թե գրացանակում նոր ֆայլը կժառանգի՞ արդյոք ընդհանուր խումբը
4. Ի՞նչ է անում sticky bit-ը, և ո՞ր գրացանակների վրա է այն սովորաբար դրված
5. Ինչո՞ւ SGID-ն միայնակ բավարար չէ shared folder-ի համար (ինչ դեր ունի umask-ը)
6. Ինչպե՞ս կգտնես համակարգում բոլոր SUID և SGID ֆայլերը

## Հաջորդ քայլեր

Այս թեման permissions-ի մոդելի խորացումն է. շարունակիր [Filesystem](filesystem.md)-ի հետ (`chmod`, `chown`, `umask`), ապա անցիր **ACL**-ներին (`setfacl`/`getfacl`), որոնք SGID-ից ավելի նուրբ լուծում են տալիս բազմաթիվ օգտատերերի և ծառայությունների համար, և **Linux capabilities**-ին՝ SUID-ի ավելի անվտանգ այլընտրանքին (տես նաև [Processes](processes.md))։


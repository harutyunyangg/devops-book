# Linux

Linux-ը DevOps-ի հիմքն է. production workload-ների մեծ մասը գործում է Linux միջավայրում։ Այս էջում Linux թեման ամբողջությամբ հավաքված է մեկ տեղում՝ բաժիններով. համակարգը դիտարկում ենք որպես պրոցեսների, ֆայլերի, ցանցային socket-ների և ծառայությունների ամբողջություն։

## Էջի բաժիններ

- [Processes](#processes) — ծրագրերի գործարկում, դիտարկում և անվտանգ դադարեցում։
- [Bash Scripting](#bash-scripting) — սկրիպտեր, ավտոմատացում, subshell vs source, idempotency։
- [Cgroups](#cgroups) — ռեսուրսների (CPU, memory, I/O) խմբավորում և սահմանափակում systemd-ի ու կոնտեյներների համար։
- [Networking](#networking) — ինտերֆեյսներ, IP կարգավորում, routing և Netplan։
- [Firewall](#firewall) — Ubuntu-ի firewall (UFW և nftables)։
- [Filesystem](#filesystem) — ցուցակում, գրացանակային կառուցվածք և permissions-ի հիմունքներ։
- [Special Bits (SUID/SGID/Sticky)](#special-bits) — հատուկ permission բիթեր, shared folder-ներ և privilege escalation-ի ռիսկեր։
- systemd — շուտով։

Այս էջը կարդալու ձևը. սկսիր առաջին բաժնից և անցիր հաջորդին միայն այն ժամանակ, երբ գործարկել ես տվյալ բաժնի lab-ը սեփական VM-ում կամ կոնտեյներում։

## Processes {#processes}

### Տեսություն

**Program**-ը սկավառակի վրա գտնվող executable ֆայլ է։ Երբ kernel-ը այն գործարկում է, առաջանում է **process**՝ իր հիշողությամբ, PID-ով (process ID), բաց ֆայլերով և իրավունքներով։ Մի program-ը կարող է ունենալ շատ process-ներ։

Յուրաքանչյուր process ունի ծնող՝ parent process։ Process-ի PPID-ը (parent PID) օգնում է հասկանալ, թե ով է այն գործարկել։ Process-ը կարող է ունենալ մեկ կամ ավելի threads, որոնք կիսում են նույն process-ի հիշողությունը։

Linux-ում process-ի վիճակներից օգտակարներն են.

- `R` — running կամ runnable,
- `S` — interruptible sleep՝ սպասում է իրադարձության,
- `D` — uninterruptible sleep, սովորաբար I/O սպասում,
- `T` — stopped,
- `Z` — zombie. ավարտվել է, բայց ծնողը դեռ չի վերցրել exit status-ը։

#### Ինչպե՞ս տարբերել CPU-ի և I/O-ի բեռը

**Load Average**-ը ցույց է տալիս ակտիվ process-ների միջին քանակը 1, 5 և 15 րոպեների ընթացքում (օրինակ՝ `load average: 8.45, 7.90, 6.20`)։ Այն միայն CPU-ի զբաղվածությունը չի չափում. հաշվում են և՛ `R` (running/runnable), և՛ `D` (uninterruptible sleep, սովորաբար I/O-ի սպասում) վիճակի process-ները։ Բարձր Load-ի պատճառը գտնելու համար նայիր `top`-ի `%Cpu` տողին.

| Ցուցանիշ | Իմաստ |
| --- | --- |
| `us` | User time. ձեր process-ների (user-space) հաշվարկային ժամանակ |
| `sy` | System time. kernel-ի (system-space) գործողությունների ժամանակ |
| `ni` | Nice time. ցածր priority task-երի user time |
| `id` | Idle. իրական պարապ ժամանակ |
| `wa` | I/O wait. CPU-ն սպասում է I/O-ի ավարտին |
| `hi` / `si` | Hardware / software interrupt-ների ժամանակ |
| `st` | Steal. virtualized միջավայրում hypervisor-ի ժամանակ |

Որոշումը կայացնելու ալգորիթմը.

- «`us` + `sy`»-ը բարձր (=80%+) և «`wa`»-ն ցածր → համակարգը **CPU-bound** է. բեռը տալիս են `R` process-ները.
- «`us` + `sy`»-ը ցածր և «`wa`»-ն բարձր (20%+) → համակարգը **I/O-bound** է. CPU-ն սպասում է I/O-ի, Load Average-ը աճել է `D` process-ների պատճառով.

**us vs sy, ring-եր և space-ներ.** `us`-ը ձեր ծրագրի «մաքուր» աշխատանքն է, իսկ `sy`-ը՝ kernel-ի աշխատանքը՝ ձեր ծրագրի խնդրանքով։ CPU-ի ring/privilege levels-ը (Ring 0-3) ապարատային պաշտպանության մեխանիզմ է. Ring 0-ում աշխատ է kernel-ը (`sy`), Ring 3-ում՝ ձեր process-ները (`us`), Ring 1-2-ը ժամանակակից Linux-ում գրեթե չեն օգտագործվում։ Երբ process-ին անհրաժեշտ է ֆայլի մուտք, system call-ի միջոցով CPU-ն Ring 3-ից անցնում է Ring 0։ «Space»-ը վերաբերում է հիշողությանը՝ User Space-ը՝ ձեր process-ները, Kernel Space-ը՝ միջուկի կառուցվածքը.

**Ախտորոշիչ գործիքներ.**

- `top` / `htop` — ամենահեշտ քայլ, Load Average-ն ու `%Cpu`-ն և process list-ը.
- `vmstat 1` — Virtual Memory Statistics. համակարգի ընդհանուր պատկեր (processes, CPU, memory, I/O, swap)՝ ամեն վայրկյան
- `mpstat -P ALL 1` — Multi-Processor Statistics. CPU-ի յուրաքանչյուր core-ի ծանրաբեռնվածությունը.
- `iostat -x 1` — Input/Output Statistics. device-ի %util-ը և await-ը.
- `iotop` — I/O Top. որի process-ները ծանրաբեռնում են սկավառակը.
- `ps -eo pid,stat,wchan:30,comm` — D-վիճակի (I/O-ին սպասող) process-ների ցուցակ.

`iostat`-ի `%util`-ը 100%-ին մոտ լինելը խոսում է սկավառակի գերբեռնվածության մասին, իսկ բարձր `await`-ը՝ դանդաղ I/O-ի նշան է. D-վիճակի process-ներն են Load Average-ի մեջ՝ ահա ցածր CPU-ի դեպքում բարձր Load-ի պատճառը։

### Հիմնական հրամաններ

| Հրաման | Նպատակ | Զգուշացում |
| --- | --- | --- |
| `ps aux` | Process-ների snapshot | Շատ output կարող է տալ |
| `ps -o pid,ppid,stat,cmd -p <PID>` | Մեկ process-ի մանրամասներ | Անվտանգ, միայն կարդում է |
| `pgrep -a nginx` | Գտնել process-ը անունով | Անվտանգ, միայն կարդում է |
| `top` | CPU/RAM-ի live դիտարկում | `q`՝ դուրս գալու համար |
| `vmstat 1` | Ընդհանուր պատկեր՝ processes, memory, CPU, I/O, swap | Անվտանգ, միայն կարդում է |
| `mpstat -P ALL 1` | Յուրաքանչյուր core-ի CPU բեռը | sysstat փաթեթից |
| `iostat -x 1` | Սկավառակի `%util`-ը և `await`-ը | sysstat փաթեթից, կարող է root պահանջել |
| `iotop -o` | Որ process-ի I/O բեռն է մեծ | Root է պահանջում (`sudo`) |
| `ps -eo pid,stat,wchan:30,comm` | D-վիճակի process-ների ցուցակ | Անվտանգ, միայն կարդում է |
| `ionice -c 3 -p <PID>` | Process-ի I/O-ն՝ idle class | Ազդում է I/O priority-ի վրա |
| `kill -TERM <PID>` | Խնդրել process-ին նորմալ ավարտել | Նախընտրելի առաջին քայլ |
| `kill -KILL <PID>` | Kernel-ով ակնթարթորեն կանգնեցնել process-ը | Կարող է կորցնել չգրված տվյալներ |

`SIGTERM`-ը process-ին հնարավորություն է տալիս փակել ֆայլերը և ավարտել հարցումները։ `SIGKILL`-ը չի կարող բռնվել կամ անտեսվել, այդ պատճառով այն վերջին տարբերակն է։

### Փորձարկում (Lab)

Այս փորձը գործարկիր սովորական Linux VM-ում կամ development container-ում։ Մի օգտագործիր production server։

```bash
sleep 300 &
echo $!
```

`&`-ը հրամանը տեղափոխում է background, իսկ `$!`-ը տպում է վերջին background process-ի PID-ը։ Այնուհետև տեղադրիր PID-ը՝

```bash
ps -o pid,ppid,stat,etime,cmd -p <PID>
kill -TERM <PID>
ps -p <PID>
```

Սպասվող արդյունքը. առաջին `ps`-ում կտեսնես `sleep 300`, իսկ `SIGTERM`-ից հետո երկրորդը header-ից բացի ոչինչ չպետք է տպի։

!!! warning "Մի՛ կրկնիր PID-երը"
    Երբեք մի գործարկիր `kill` հրամանը հին copy-paste արված PID-ով. ավարտված PID-ը կարող է վերօգտագործվել մեկ այլ process-ի համար։ Նախ նորից ստուգիր command-ը `ps`-ով։

#### Lab 2. I/O-ի բեռի ախտորոշում

Միջավայր՝ սովորական Linux VM կամ container, որտեղ տեղադրել ես `sysstat`-ը (տալիս է `mpstat`/`iostat`) և `iotop`-ը։ Մի գործարկիր production server-ում.

```bash
sudo apt install -y sysstat iotop    # Debian/Ubuntu
dd if=/dev/zero of=/tmp/test.img oflag=direct bs=1M count=2048 &
```

`dd`-ը մեծ ֆայլ է գրում սկավառակ՝ ստեղծելով I/O-ի բեռ։ Այնուհետև մեկ ուրիշ տերմինալում.

```bash
top
vmstat 1
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /^D/ {print}'
iostat -x 1
```

Սպասվող արդյունքը՝ `top`-ում `wa`-ն բարձր է, `iostat`-ում՝ `%util`-ը մոտ 100% է, իսկ `ps`-ում `dd`-ի process-ը `D` վիճակում է։ Ավարտելուց հետո կանգնեցրու՝ `kill %1`, ապա ջնջիր ֆայլը՝ `rm /tmp/test.img`։

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Web ծառայությունը դանդաղել է, CPU-ն 100% է, իսկ load balancer-ի health check-երը սկսել են ձախողվել։

#### Ախտորոշում

1. `top`-ով գտիր ամենաշատ CPU օգտագործող PID-ը։
2. `ps -o pid,ppid,stat,etime,cmd -p <PID>`-ով հաստատիր, որ դա ճիշտ ծառայությունն է։
3. Ստուգիր process tree-ը և ծառայության logs-ը. բարձր CPU-ն կարող է լինել infinite loop, traffic spike կամ անհաջող deploy-ի հետևանք։
4. Արձանագրիր ժամանակը, PID-ը, command-ը և CPU ցուցանիշը incident note-ում՝ մինչև փոփոխություն անելը։

#### Լուծում

Եթե ծառայությունը systemd-ով է կառավարվում, նախընտրիր վերահսկվող restart-ը.

```bash
sudo systemctl restart <service>
sudo systemctl status <service>
```

Սա ավելի լավ է, քան պատահական worker PID-ներ kill անելը, քանի որ systemd-ն գիտի ծառայության lifecycle-ը։ Միայն եթե վերահսկվող կանգնեցումը չի աշխատում, ուսումնասիրիր կոնկրետ PID-ը և կիրառիր `SIGTERM`, ապա անհրաժեշտության դեպքում `SIGKILL`։

#### Կանխարգելում

- CPU saturation-ի համար դնել alert և dashboard։
- Deploy-ից առաջ ունենալ smoke test և արագ rollback։
- Սահմանել resource limits container-ների կամ systemd unit-երի համար։
- Runbook-ում գրել՝ որ ծառայությունն ինչպես է վերագործարկվում։

#### Իրավիճակ 2. Բարձր Load Average, բայց ցածր CPU՝ I/O-bound

##### Ախտանիշ

Տվյալների բազա և վեբ-հավելված ունեցող 8-core սերվերում՝ `load average: 8.45, 7.90, 6.20`։ `top`-ում՝ `5.2 us, 3.1 sy, 18.5 id, 72.1 wa`։ Ծառայությունը դանդաղ է աշխատում, հարցումների պատասխանները երկարում են։

##### Ախտորոշում

1. `top`-ի `wa` = 72.1%, իսկ `us + sy` = 8.3%՝ ուրեմն դա CPU-ի խնդիր չէ՝ CPU-ն սպասում է I/O-ի ավարտին։

2. `vmstat 1`-ում `b` (blocked) = 5, `bi`/`bo`-ն բարձր են (2845/1230)՝ I/O-ի ակտիվությունը մեծ է։
3. `mpstat -P ALL 1`-ում բոլոր 8 cores-ի `iowait`-ը 70-73% է՝ բեռը հավասարաչափ բաշխված է, ոչ թե մեկ core-ում։
4. `iotop -o`-ում տեսանելի է՝ `mysql` կարդում է 1842 M/s, `php-fpm` 567 M/s, `postgres` 436 M/s, իսկ `dd` գրում է 890 M/s՝ պատճառը պարզ է։

##### Լուծում

Կարճաժամկետ՝ իջեցնել backup-ի I/O-ն.

```bash
ionice -c 3 -p <dd PID>          # dd-ի I/O-ն` idle class
```

Երկարաժամկետ՝ backup-ը տեղափոխել ոչ պիկ ժամի կամ առանձին մեքենա, MySQL-ի ծանր հարցումներին ինդեքս ավելացնել, անհրաժեշտության դեպքում օգտագործել ավելի արագ սկավառակ (NVMe)։

##### Կանխարգելում

- I/O wait-ի և disk `%util`-ի համար alert դնել։
- Բոլոր մեծ jobs-ը (backup, import) գործարկել `ionice`-ով ցածր priority։
- Սկավառակի տիպը (HDD/SSD/NVMe) հաշվի առնել capacity planning-ում։

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ տարբերություն կա `SIGTERM` և `SIGKILL` միջև

`SIGTERM`-ը process-ին ուղարկում է նորմալ ավարտելու հարցում, և այն կարող է մաքրել resource-ները։ `SIGKILL`-ը kernel-ի պարտադիր կանգ է, process-ը չի կարող մշակել այն, ուստի օգտագործվում է միայն վերջին քայլով։

#### Ի՞նչ է zombie process-ը

Ավարտված child process է, որի exit status-ը ծնողը դեռ չի կարդացել։ Այն գրեթե CPU/RAM չի օգտագործում, բայց պահում է process table-ի գրառում։ Խնդրի արմատը սովորաբար parent process-ի սխալ վարքն է։

#### Ինչպե՞ս կհաստատես, որ բարձր Load Average-ը I/O-ի, ոչ թե CPU-ի պատճառով է

`top`-ի `%Cpu` տողում `wa`-ն բարձր է (օր. 70%), իսկ `us + sy`-ը՝ ցածր՝ նշանակում է՝ CPU-ն սպասում է I/O-ի ավարտին։ Հաստատելու համար՝ `vmstat 1`-ի `b` սյունակը (blocked process-ներ), `iostat -x 1`-ի `%util`-ն ու `await`-ը, և `iotop`-ը՝ որը ցույց է տալիս կոնկրետ `I/O`-ծանր process-ը։ Load-ն այստեղ բարձրանում է D-վիճակի process-ների հաշվին՝ CPU-ն պարապ է, բայց system-ը՝ ոչ։

### Ինքնաստուգում

1. Ինչպե՞ս կգտնես process-ի parent-ը։
2. Ի՞նչ տեղեկություն կտա `STAT` սյունակը։
3. Եթե ծառայությունը կառավարվում է systemd-ով, ո՞րն է առաջին restart հրամանը և ինչո՞ւ։
4. Ի՞նչ ապացույցներ կպահպանես բարձր CPU incident-ի ժամանակ՝ փոփոխություն անելուց առաջ։
5. Ի՞նչ է ցույց տալիս `top`-ի `wa` սյունակը, և երբ է այդ ցուցանիշը մտահոգիչ։
6. Ի՞նչ հրամանով կգտնես D-վիճակի process-ները, և ի՞նչ է դա նշանակում Load Average-ի համար։
7. Ի՞նչ կարող են ասել `iostat -x 1`-ի `%util`-ը և `await`-ը սկավառակի վիճակի մասին։

### Հաջորդ քայլեր

Շարունակիր [Filesystem](#filesystem) բաժնով, ապա անցիր systemd-ի բաժնին, որը շուտով կավելանա այս էջում։

## Bash Scripting {#bash-scripting}

### Տեսություն

**Bash** (Bourne Again SHell) -ը Linux-ի (և macOS-ի) ամենատարածված command-line ինտերպրետատորն է (shell)։ **Bash script**-ը պարզապես տեքստային ֆայլ է, որը պարունակում է հրամանների հաջորդականություն, որոնք կարող են գործարկվել միասին ավտոմատացնելու կրկնվող գործողությունները.

Bash-ը DevOps-ի հիմնական գործիքներից է. նախքան Ansible-ի, Terraform-ի կամ CI/CD pipeline-ների անցումը մենք հաճախ ամեն ինչ սկսում ենք սկրիպտից։ Այն թույլ է տալիս.

- **ավտոմատացնել** կրկնվող գործողությունները (backup, մոնիտորինգ),
- **կառավարել համակարգը** (ծրագրերի տեղադրում, կարգավորումներ, օգտատերեր),
- **ինտեգրել տարբեր Linux հրամաններ** մեկ աշխատանքային հոսքում,
- աշխատել **ավելի արագ**, քան GUI-ով, հատկապես սերվերներում.

#### Սկրիպտի կառուցվածքն ու գործարկումը

Սկրիպտի առաջին տողում գրվում է **shebang** (`` `#!/bin/bash` ``), որը ցույց է տալիս, թե որ ինտերպրետատորով պետք է գործարկվի ֆայլը։ `#`-ով սկսվող տողերը մեկնաբանություններ են և չեն կատարվում.

```bash
#!/bin/bash
# Սա մեկնաբանություն է
echo "Բարև, աշխարհ"
name="Արմեն"
echo "Բարև, $name"
```

Ֆայլը գործարկելու համար նախ դարձնում ենք executable, ապա կանչում.

```bash
chmod +x my_first_script.sh   # դարձնել գործարկելի
./my_first_script.sh          # գործարկել
# կամ
bash my_first_script.sh
```

#### Հիմնական սինտաքս

**Փոփոխականներ.** Bash-ում փոփոխականը սահմանվում է առանց բացատների `=`-ի շուրջ, իսկ արժեքը օգտագործվում է `$name` կամ `${name}` ձևով.

```bash
name="Աննա"                    # տողային փոփոխական
age=25                         # թիվ
echo "Իմ անունը $name է, ես $age տարեկան եմ"
new_age=$((age + 5))           # հաշվողական գործողություն
```

Փոփոխականների տեսակներ.

- **Տեղական**: `local var="value"` (ֆունկցիայի ներսում),
- **Միջավայրի**: `export PATH="/usr/local/bin:$PATH"` — ժառանգվում է child process-ների կողմից,
- **Ընթերցում օգտատիրոջից**: `read -p "անուն" username`,
- **Հրամանի արդյունք**: `current_date=$(date)`.

##### Ավտոմատ արտահանում `set -a` / `set +a`

Սովորաբար փոփոխականը child process-ին փոխանցելու համար գրում ենք հստակ `export VAR=value`։ **`set -a`** (alias`set -o allexport`) դա դարձնում է ավտոմատ. այդ պահից ստեղծված կամ փոփոխված յուրաքանչյուր փոփոխական (և ֆունկցիա) ստանում է export հատկանիշ և արտահանվում է child process-ների համար։ **`set +a`** (կամ `set +o allexport`) անջատում է այդ ռեժիմը. դրանից հետո փոփոխականներն այլևս չեն արտահանվում, քանի դեռ հստակ չգրենք `export` (bash-ում `+`-ի տեսքով օպցիան անջատվում է)։

```bash
#!/bin/bash
set -euo pipefail   # անվտանգ ռեժիմ
set -a              # միացնում ենք ավտո-արտահանումը
MY_ENV_VAR="value"  # ինքնաբերաբար export կլինի
set +a              # անջատում ենք
OTHER="text"        # սա արդեն export չի լինի

env | grep -E 'MY_ENV_VAR|OTHER'   # կտպվի միայն MY_ENV_VAR
```

`set -a`-ն օգտակար է, երբ բազմաթիվ փոփոխականներ պետք է փոխանցվեն child process-ին (օրինակ. CI/CD job-ին կամ `docker run`-ին) առանց յուրաքանչյուրին առանձին `export` գրելու։ Այն պարունակում և ամփոփում է զրույցի բոլոր երեք set օպցիաները.

| Հրաման | Նշանակություն |
| --- | --- |
| `set -euo pipefail` | Խիստ ռեժիմ. կանգնել սխալի դեպքում, զգուշանալ չսահմանված փոփոխականներից, հաշվի առնել pipe-ի բոլոր սխալները |
| `set -a` | Միացնել փոփոխականների ավտոմատ արտահանումը |
| `set +a` | Անջատել ավտոմատ արտահանումը |

!!! warning "Անվտանգություն"
    `set -a`-ի միացված վիճակում գրված բոլոր փոփոխականները, այդ թվում գաղտնիքները (tokens, passwords), արտահանվում են յուրաքանչյուր child process-ի։ Օգտագործումից հետո միշտ անջատիր այն `set +a`-ով, որպեսզի պատահական փոփոխականների կամ գաղտնիքների արտահանումը չշարունակվի script-ի մնացած մասում։

**Պայմանական արտահայտություններ (`if/elif/else`).** Թվերի օպերատորներն են `-eq` (հավասար), `-ne` (ոչ հավասար), `-gt` (մեծ), `-lt` (փոքր), `-ge` (մեծ կամ հավասար), `-le` (փոքր կամ հավասար)։ Տողերի համար `=`, `!=`, `-z` (դատարկ), `-n` (ոչ դատարկ)։ Ֆայլերի ստուգումներ `-f` (ֆայլ), `-d` (պանակ), `-e` (գոյություն ունի), `-r` (կարելի է կարդալ), `-w` (կարելի է գրել), `-x` (գործարկելի)։

```bash
if [ -f "file.txt" ]; then
    echo "Ֆայլը կա"
elif [ -d "folder" ]; then
    echo "Պանակը կա"
else
    echo "Ոչ մեկը չկա"
fi
```

**Ցիկլեր.**

```bash
for i in {1..5}; do
    echo "Համար $i"
done

for file in *.txt; do
    echo "Մշակում եմ $file"
    cat "$file"
done

counter=1
while [ $counter -le 5 ]; do
    echo "Քայլ $counter"
    ((counter++))
done

counter=1
until [ $counter -gt 5 ]; do
    echo "Counter $counter"
    ((counter++))
done
```

Ֆայլի տողերով շրջելը `while IFS= read -r line; do ...; done < "file.txt"` — `IFS=`-ը և `-r`-ը պահպանում են բացատներն ու backslash-ները.

**Զանգվածներ.**

```bash
fruits=("խնձոր" "բանան" "նարինջ")
echo "${fruits[0]}"        # խնձոր
echo "${fruits[@]}"        # բոլոր անդամները
echo "${#fruits[@]}"       # քանակը
fruits+=("կիվի")           # ավելացնել
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done
```

**Ֆունկցիաներ.**

```bash
greet() {        # կամ function greet() {
    local name=$1
    echo "Բարև, $name"
}
greet "Արմեն"
```

Նշում. ֆունկցիայի `return`-ը վերադարձնում է 0-255 exit status, ոչ թե տող. տող վերադարձնելու համար `echo`-ով արտածիր և արժեքը բռնիր `$(...)`-ով.

**Արգումենտներ.**

```bash
echo "Սկրիպտի անունը $0"
echo "Առաջին արգումենտ $1"
echo "Բոլոր արգումենտները $@"
echo "Քանակը $#"
for arg in "$@"; do
    echo "Արգումենտ $arg"
done
```

**Case, `&&`/`||`, here-doc.**

**`case` — ընտրություն ըստ արժեքի.** Ստուգում է `$1`-ը (սկրիպտի առաջին արգումենտը) և կատարում համապատասխան ճյուղը. `*)` ճյուղը. default-ն է (այն, ինչ չի տեղավորվում)։ Ամեն ճյուղ վերջանում է `;;`-ով, իսկ կառույցը փակվում է `esac`-ով (`case`-ի հակառակը).

```bash
case $1 in
  start)    echo "Մեկնարկում...";;
  stop)     echo "Դադարեցնում...";;
  restart)  echo "Վերագործարկում...";;
  *)        echo "Օգտագործում: $0 {start|stop|restart}";;
esac
```

**`&&` (AND) և `||` (OR).** `[ ... ]`-ը (test) ստուգում է պայմանը և վերադարձնում `0` (ճիշտ) կամ ոչ-0 (սխալ), և Bash-ը որոշումը կայացնում է **այդ exit code-ի վրա**.
- `A && B` — եթե A-ն վերադարձրեց 0-ն (հաջող), կատարվի B-ը,
- `A || B` — եթե A-ն վերադարձրեց ոչ-0-ն (ձախողում), կատարվի B-ը.

```bash
[ -f file.txt ] && echo "Ֆայլը կա"     # պայմանը ճիշտ  է → echo-ն կատարվում է
[ -f file.txt ] || echo "Ֆայլը չկա"   # պայմանը սխալ է → echo-ն կատարվում է
```

**Here-doc (`<< EOF`).** Ինչ-որ հրամանին (հաճախ `cat`-ին) բազմատող տեքստ է փոխանցվում որպես input. `EOF`-ը դելիմիտերն (բաժանարարն) է. կարող ես ցանկացած անուն ընտրել, բայց սկզբի և վերջի «բանալի»-ն պետք է նույնը լինի:

```bash
cat << EOF
Սա բազմատող
տեքստ է
EOF
```

#### Process execution model. subshell vs source

Bash script-ը աշխատում է կամ **նոր ենթապրոցեսում (subshell)**, կամ **նույն shell-ի կոնտեքստում (source)**, և հենց սա է որոշում, թե script-ի փոփոխությունները կպահպանվեն, թե ոչ.

| Գործարկման եղանակ | Հրաման | Ինչ է կատարվում | Փոփոխականները | Directory |
| --- | --- | --- | --- | --- |
| Subshell | `./script.sh` կամ `bash script.sh` | Նոր child process (subshell) | ❌ Կորչում են | ❌ Չի պահպանվում |
| Source | `source script.sh` կամ `. script.sh` | Ընթացիկ shell-ում | ✅ Պահպանվում են | ✅ Պահպանվում է |

Subshell-ը ժառանգում է ծնողի environment-ը (փոփոխականներ, directory, file descriptors), բայց դրանից հետո աշխատում է մեկուսացված script-ի ավարտից հետո դրա ներսում արված ցանկացած `cd`, `export`, `alias` կամ `set` ավարտվում է նրա հետ, իսկ ծնող shell-ը մնում է անփոփոխ։ Դրա պատճառով `./script.sh`-ի մեջ արված `cd`-ը ազդում է միայն script-ի ներսում. դուրս գալուց հետո մնում ես սկզբնական directory-ում.

Source-ը (`source script.sh` կամ `. script.sh`) կատարվում է ընթացիկ shell-ում առանց նոր պրոցեսի, ուստի `cd`-ն, `export`-ը, alias-ները և ֆունկցիաները պահպանվում են session-ում. այսպես են բեռնվում `.bashrc`-ը և `.profile`-ը, ակտիվանում է Python virtual environment-ը (`source venv/bin/activate`), և կարգավորվում են environment փոփոխականները.

**Ինչպես համոզվել, թե որտեղ է աշխատում script-ը.** `$$`-ը ցույց է տալիս ընթացիկ shell-ի PID-ը, `$PPID`-ը ծնողի PID-ը, իսկ `$SHLVL`-ը shell-ի ներդրման մակարդակը (1 = հիմնական, 2 = առաջին subshell)։ `./script.sh`-ով գործարկելիս SHLVL-ը մեծանում է 1-ով, source-ով մնում է նույնը.

```bash
#!/bin/bash
echo "PID $$, ծնող $PPID, SHLVL $SHLVL"
ps -f      # պրոցեսների ծառը
```

#### Exit codes և `set -e` / `set -u`

Յուրաքանչյուր հրաման կամ պրոցես վերադարձնում է **exit code** (0-255 ամբողջ թիվ).

| Exit code | Իմաստ |
| --- | --- |
| `0` | ✅ Հաջողություն. 0-ն միակ հաջողության կոդը |
| `1`-`255` | ❌ Սխալ «ոչ զրոյական exit code» |

`$?`-ը պահում է վերջին հրամանի exit code-ը `echo $?`։ `if`-ը ինքն է ստուգում exit code-ը `if command; then ...`. Ընդհանուր արժեքներ `1` (ընդհանուր սխալ), `2` (սխալ օգտագործում), `126` (ոչ executable), `127` (հրամանը չի գտնվել)։ Ինքդ կարող ես վերադարձնել `exit 0` — հաջողություն, `exit 1` — սխալ.

- `set -e` — դադարեցնում է սկրիպտը, եթե որևէ հրաման վերադարձնի ոչ զրոյական exit code (կանխում է աղետալի շարունակությունը, օրինակ cd-ի ձախողումից հետո `rm -rf *`)։
- `set -u` — սխալ չսահմանված փոփոխականի օգտագործման դեպքում (բռնում է տառասխալները օրինակ `$nam`-ը, երբ սահմանել ես `$name`)։
- `set -euo pipefail` — ամենաանվտանգ համադրությունը. `pipefail`-ը ցույց է տալիս pipe-ի ձախողումը, նույնիսկ եթե վերջին հրամանը հաջողվել է.
- `set -x` — debug. ցուցադրում է կատարվող հրամանները.

**Անվտանգ directory-ի փոփոխություն.**

```bash
cd /var/log/application || { echo "Չեմ կարողանում մտնել" >&2; exit 1; }
```

`||`-ը «OR» է եթե `cd`-ն ձախողվի, կատարվում է `{ ... }`-ի կոդը. `echo`-ն հաղորդագրությունն ուղարկում է **STDERR**-ին (`>&2`), իսկ `exit 1`-ը սկրիպտը դադարեցնում է սխալի կոդով. սխալի հաղորդագրություններն այդպես առանձնանում են նորմալ ելքից (STDOUT)։

#### Իդեմպոտենտություն (Idempotency)

**Idempotency**. գործողության հատկությունն է, երբ նույն գործողությունը մեկ կամ բազմիցս կատարելուց տալիս է նույն արդյունքը. առանց կողմնակի ազդեցության։ Այն DevOps-ի հիմնասյուներից է. ավտոմատացման գործիքները (Ansible, Terraform) idempotent են, ուստի կարելի է 100 անգամ գործարկել միջավայրը չփչացնելով։ Cron-ի վերագործարկումն ու disaster recovery-ն ել անվտանգ են դառնում.

| Գործ | ❌ Non-idempotent | ✅ Idempotent |
| --- | --- | --- |
| Directory ստեղծել | `mkdir /path` (սխալ եթե կա) | `mkdir -p /path` |
| User ստեղծել | `useradd name` (սխալ եթե կա) | `id name \|\| useradd name` |
| Ֆայլում տող ավելացնել | `echo "line" >> f` (կրկնվում է) | `grep -q "line" f \|\| echo "line" >> f` |
| Package տեղադրել | `apt-get install -y pkg` | `dpkg -l pkg \|\| apt-get install -y pkg` |
| Service start | `systemctl start svc` | `systemctl is-active svc \|\| systemctl start svc` |

Իդեմպոտենտ script-ի կառուցվածքը նախ ստուգիր՝ արդյոք վիճակն արդեն ցանկալին է, ապա միայն եթե ոչ կատարիր փոփոխությունը, և հաստատիր արդյունքը.

```bash
#!/bin/bash
set -euo pipefail

if id "myuser" &> /dev/null; then
    echo "User-ն արդեն կա. skip"
else
    sudo useradd -m myuser
    echo "User-ը ստեղծված է"
fi

if [ -d "/opt/myapp" ]; then
    echo "Directory կա. skip"
else
    mkdir -p /opt/myapp
fi

grep -q "myapp.local" /etc/hosts 2> /dev/null || echo "127.0.0.1 myapp.local" >> /etc/hosts
```

Իդեմպոտենտ գրելաձևի idioms՝ `mkdir -p`, `id user || useradd`, `getent group g || groupadd g`, `[ -f file ] || touch file`, `grep -q "line" file || echo "line" >> file`, `[ -L link ] || ln -s target link`, `dpkg -l pkg || apt-get install -y pkg`, `[ -d repo ] || git clone url repo. Իդեմպոտենտությունը ստուգվում է` script-ը 2-3 անգամ գործարկելով՝ output-ը պետք է նույնը լինի.

### Հիմնական հրամաններ

| Հրաման | Նպատակ |
| --- | --- |
| `echo` | Տողի արտածում |
| `read -p "prompt" var` | Օգտատիրոջից մուտքի ընթերցում |
| `test` / `[ ... ]` | Պայմանի ստուգում (թիվ, տող, ֆայլ) |
| `$(...)` | Հրամանի արդյունքը փոփոխականում |
| `set -e` / `-u` / `-x` | Սխալի/debug-ի ռեժիմներ |
| `grep` | Տեքստում որոնում |
| `sed` | Տեքստի փոխարինում/խմբագրում |
| `awk` | Տեքստի մշակում սյունակներով |
| `cut` | Տողի մասերի առանձնացում |
| `sort` / `uniq` | Տեսակավորում / կրկնվողների հեռացում |
| `wc` | Տողերի/բառերի/նիշերի քանակ |
| `ps -f` | Պրոցեսների ծառը (PID/PPID) |
| `$$` / `$PPID` / `$SHLVL` | Shell-ի նույնականացում |

### Փորձարկում (Lab)

Այս փորձերը գործարկիր սովորական Linux VM-ում կամ development container-ում ոչ production-ում, և առանց production credentials-ի.
#### Lab 1. subshell vs source

Ստեղծիր `where_am_i.sh`.

```bash
#!/bin/bash
echo "1. Ես $(whoami)-ն եմ, աշխատում եմ $(pwd)-ում"
cd /etc
echo "2. Հիմա ես $(pwd)-ում եմ"
echo "PID $$, SHLVL $SHLVL"
```

Գործարկիր երկու եղանակով.

```bash
$ pwd
/home/user

$ ./where_am_i.sh
1. Ես user-ն եմ, աշխատում եմ /home/user-ում
2. Հիմա ես /etc-ում եմ
PID 5678, SHLVL 2

$ pwd
/home/user        # ❌ չի փոխվել (subshell)

$ source where_am_i.sh
1. Ես user-ն եմ, աշխատում եմ /home/user-ում
2. Հիմա ես /etc-ում եմ
PID 1234, SHLVL 1

$ pwd
/etc              # ✅ փոխվել է (source)
```

Սպասվող արդյունք subshell-ում `cd`-ն կորչում է, PID/SHLVL-ը փոխվում են. source-ում ամենը պահպանվում է, PID-ը նույնը.

#### Lab 2: Exit codes և `set -euo pipefail`

Ինտերակտիվ shell-ում ստուգիր exit codes-ը.

```bash
$ true;  echo $?        # 0 - հաջողություն
$ false; echo $?        # 1 - սխալ
$ ls /no_such_dir; echo $?   # 2 - սխալ օգտագործում
```

Ստեղծիր script, որն օգտագործում է `set -euo pipefail` և միտումնավոր սխալ է պարունակում.

```bash
#!/bin/bash
set -euo pipefail
cd /root/non_existing_dir || { echo "Cannot cd" >&2; exit 1; }
echo "Սա չի տպվի"      # script-ը կանգնում է cd-ի ձախողումից
```

Սպասվող արդյունք. script-ը ավարտվում է 1-ով, `cd`-ի ձախողումից հետո. `echo`-ն չի աշխատում. Ավելացրու նաև `echo "$undefined_var"` և տես `set -u`-ի unbound variable սխալը.

#### Lab 3: Idempotent setup script

Գրիր `setup.sh`, որը կարելի է բազմիցս գործարկել նույն արդյունքով.

```bash
#!/bin/bash
set -euo pipefail
mkdir -p /tmp/demo_env
[ -f /tmp/demo_env/info.txt ] || echo "first" > /tmp/demo_env/info.txt
grep -q "127.0.0.1 demo.local" /etc/hosts || echo "127.0.0.1 demo.local" >> /etc/hosts
echo "Done"
```

Գործարկիր երկու անգամ.

```bash
./setup.sh; echo $?; cat /tmp/demo_env/info.txt
./setup.sh; echo $?; cat /tmp/demo_env/info.txt
```

Սպասվող արդյունք երկու run-ն էլ ավարտվում է 0-ով, իսկ `info.txt`-ի պարունակությունը չի կրկնվում script-ը idempotent է.

#### Lab 4: Ավտոմատ արտահանում `set -a` / `set +a`

Ստուգիր, թե որ փոփոխականներն է տեսնում child shell-ը `env`-ի միջոցով.

```bash
set -a                # միացնում ենք ավտո-արտահանումը
MY_AUTO="hello"
set +a                # անջատում ենք
MY_PLAIN="bye"        # սա չի արտահանվելու

sh -c 'echo "MY_AUTO=$MY_AUTO MY_PLAIN=${MY_PLAIN:-EMPTY}"'
```

Սպասվող արդյունք. `MY_AUTO=hello MY_PLAIN=EMPTY` (`set -a`-ի տակ սահմանված փոփոխականը հասանելի է child-ին, իսկ `set +a`-ից հետոյինը՝ ոչ)։ Ստուգիր նաև, որ `set -a`-ի միացված վիճակում ընթացիկ օպցիաները ցույց տվող `$-`-ում հայտնվում է `a`-ն.

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Production ինժեները գրել է backup/log-rotation script, որը պետք է ամեն գիշեր cron-ից գնա `/var/log/application`-ը, արխիվացնի log ֆայլերը և փոխանցի `/backup/logs`-ին։ Փորձարկման ժամանակ նկատվում է արխիվացված ֆայլերը հայտնվում են սխալ directory-ում, և script-ը չի գտնում log ֆայլերը, քանի որ աշխատում է սխալ directory-ում.

#### Ախտորոշում

1. `ps -f`-ով և `echo $$`, `echo $SHLVL`-ով ստուգիր script-ն աշխատում է subshell-ում, իսկ `cd`-ն script-ի ավարտից հետո կորչում է,
2. Ստուգիր script-ը կախված է ընթացիկ working directory-ից (relative paths, մենակ `cd`),
3. Հաշվի առ cron-ը script-ը գործարկում է minimal environment-ով առանց `.bashrc`, բայց interactive shell-ում դու PATH-ն ու relative path-ները սովոր ես տեսնել.

#### Լուծում

Սկրիպտը վերածիր absolute paths-ով գործարկման եղանակից անկախ.

```bash
#!/bin/bash
set -euo pipefail
LOG_DIR="/var/log/application"
BACKUP_DIR="/backup/logs"
mkdir -p "$BACKUP_DIR"
cd "$LOG_DIR" || { echo "Cannot change dir" >&2; exit 1; }
tar -czf "$BACKUP_DIR/backup_$(date +%Y%m%d).tar.gz" *.log
echo "Backup ready" | logger -t backup_script
```

Crontab-ում նշել ամբողջական ճանապարհը.

```
0 2 * * * /usr/local/bin/backup_logs.sh
```

Ստուգիր արդյունքը.

```bash
ls -la /backup/logs
tar -tzf /backup/logs/backup_*.tar.gz | head -20
journalctl -t backup_script --since "1 hour ago"
```

#### Կանխարգելում

- Բոլոր scripts-ներում օգտագործիր absolute paths. անկախ գործարկման միջավայրից (cron, systemd, CI/CD),
- Սկզբում `set -euo pipefail`,
- Դիր idempotent միջոց `mkdir -p` և այլն,
- Cron/systemd-ի համար unit-ում նշիր WorkingDirectory-ը, script-ի աշխատանքը գրանցիր syslog-ում.

### Հարցազրույցի հարցեր և պատասխաններ

#### Ինչու՞ է `./script.sh`-ով գործարկելիս directory-ի փոփոխությունը չպահպանվում, և ինչպե՞ս լուծել

`./script.sh`-ն նոր subshell (child process) է ստեղծում, որտեղ կատարվում է `cd`-ը. subshell-ի ավարտից հետո փոփոխությունները կորչում են, և ծնող shell-ը մնում է նախկին directory-ում. Պահպանելու համար `source script.sh` կամ `. script.sh`-ը որը կատարում է ընթացիկ shell-ում. Production-ում սակայն, նախընտրում են absolute paths-ով գործարկման եղանակից անկախ.

#### Ինչպե՞ս ստուգել script-ը subshell-ում է, թե source-ով

`$$`-ը համեմատիր `$PPID`-ի հետ. source-ով `$$`-ը ծնողի PID-ն է նույն shell-ը. subshell-ում `$$`-ը նոր PID է, և `$SHLVL`-ը 1-ով մեծ է. Բացի դա `ps -f`-ի PID/PPID/CMD սյունակներն են ցույց տալիս որտեղ է աշխատում script-ը.

#### Ի՞նչ ռիսկեր կան subshell-ի և source-ի հետ ավտոմատացված միջավայրերում

Ենթադրությունները որ script-ն աշխատում է որոշակի directory-ում կամ environment-ով. cron-ը minimal environment-ով է առանց `.bashrc`-ի. source-ը կարող է անսպասելիորեն փոխել CI/CD pipeline-ի shell-ի միջավայրը. Լուծում absolute paths, PATH-ի սահմանում script-ի սկզբում, `set -euo pipefail`, WorkingDirectory unit-ում, logging, և pipeline-ում subshell-ի նախընտրում եթե script-ը չպետք է ազդի shell-ի վրա.

#### Ի՞նչ է idempotency, և ինչո՞ւ այն կարևոր DevOps-ում

Գործողությունը idempotent է, եթե կրկնակի կատարումը տալիս է նույն արդյունքը առանց կողմնակի ազդեցության. Այն թույլ է տալիս անվտանգորեն կրկնել cron-ը, CI/CD-ն, recovery-ն առանց միջավայրը փչացնելու. ոչ idempotent-ը վտանգավոր. Օրինակ `mkdir -p`-ի և `mkdir`-ի տարբերությունը.

#### Ի՞նչ է անում `set -a`-ն, և ի՞նչ ռիսկ ունի

`set -a`-ն (այլ `set -o allexport`) ստեղծված կամ փոփոխված բոլոր փոփոխականներն ու ֆունկցիաները ավտոմատ կերպով արտահանում է child process-ներին, այնպես որ պետք չէ յուրաքանչյուրին առանձին `export` գրել։ Ռիսկն այն է, որ գաղտնիքները (tokens, passwords) ևս արտահանվում են բոլոր child process-ներին, իսկ պատահական փոփոխականներն արտահոսում են։ Դրա պատճառով օգտագործումից հետո անպայման անջատիր այն `set +a`-ով, և ցանկալի է` օգտագործել այն միայն փոքր, խմբավորված scope-ում։

### Ինքնաստուգում

1. Ի՞նչ է shebang-ը և ինչո՞ւ այն դրվում է սկրիպտի առաջին տողում
2. Ի՞նչ տարբերություն է subshell-ի և source-ի միջև. ե՞րբ կկորցնես `cd`-ի փոփոխությունը
3. Ի՞նչ է ցույց տալիս `$?`-ը, և ե՞րբ է «ոչ զրոյական» exit code-ը համարվում սխալ
4. Ի՞նչ է անում `set -euo pipefail`-ը և ինչո՞ւ է այն կարևոր
5. Ինչպե՞ս կդարձնես directory-ի ստեղծումը idempotent
6. Ի՞նչ է անում `set -a`-ն, և ինչո՞ւ պետք է ավարտից հետո `set +a`-ով անջատել

### Հաջորդ քայլեր

Շարունակիր [Filesystem](#filesystem) բաժնով, ապա անցիր systemd-ի բաժնին, որը շուտով կավելանա այս էջում: կիրառիր ձեր գրած scripts-ում `set -euo pipefail`, absolute paths և idempotent մեթոդներ (տես նաև [Linux Processes](#processes))։

## Cgroups {#cgroups}

### Տեսություն

**Cgroups** (Control Groups) -ը Linux-ի միջուկի (kernel) հնարավորություն է, որը թույլ է տալիս պրոցեսները խմբավորել և կառավարել, թե խմբերից յուրաքանչյուրը որքան համակարգային ռեսուրս (պրոցեսորի ժամանակ, հիշողություն, I/O, պրոցեսների քանակ) կարող է օգտագործել։ Այն ապահովում է ռեսուրսների մեկուսացում և սահմանափակում, ինչը կարևոր է համակարգի կայունության և անվտանգության համար. օրինակ՝ անկառավարելի պրոցեսը չի կարող «ուտել» ամբողջ հիշողությունը և ճգնաժամի մեջ գցել մյուս ծառայությունները։

#### Հիմնական հասկացություններ

- **Cgroup** — պրոցեսների խումբ, որի վրա կիրառվում են ռեսուրսների որոշակի սահմանափակումներ։
- **Controller / Subsystem** — միջուկի բաղադրիչ, որը կառավարում է կոնկրետ ռեսուրս (օրինակ, `memory`, `cpu`, `io`, `pids`)։
- **Hierarchy (հիերարխիա)** — cgroups-ների ծառային կառուցվածք, որտեղ սահմանափակումները ժառանգվում են ծնողից (parent)՝ զավակներին (children)։

#### Cgroups-ի երկու տարբերակները

Linux-ում գոյություն ունի cgroups-ի երկու տարբերակ.

- **Cgroups v1 (հին)** — յուրաքանչյուր ռեսուրս (CPU, հիշողություն և այլն) ունի իր առանձին հիերարխիան։ Տարբեր հիերարխիաներում ռեսուրսների համաձայնեցված կառավարումը կարող է բարդացնել աշխատանքը։
- **Cgroups v2 (նոր, նախընտրելի)** — օգտագործում է մեկ միասնական հիերարխիա բոլոր controller-ների համար, ինչը հեշտացնում է կառավարումը։ Սա ակտիվ զարգացվող և առաջարկվող տարբերակն է. ժամանակակից Ubuntu-ի (22.04 և ավելի նոր) և ամենաթարմ Docker-ի վրա cgroups v2-ն է տիրապետողը։

#### Ինչպե՞ս ստուգել, թե որ տարբերակն է ակտիվ

```bash
mount | grep cgroup
```

Եթե տեսնում եք `cgroup2 on /sys/fs/cgroup ...` տող, ապա համակարգը գործարկում է cgroups v2-ով. այս դեպքում `/sys/fs/cgroup/`-ում գտնվում է միակ՝ v2-ի միասնական հիերարխիան (`cgroup.controllers`, `cgroup.subtree_control` և այլն)։ Եթե նշված է `cgroup on /sys/fs/cgroup/...` (առանց `2`-ի), ուրեմն ակտիվ են v1-ի հիերարխիաները (`memory`, `cpu` և այլ առանձին)։ Ժամանակակից Ubuntu-ում խորհուրդ է տրվում աշխատել v2-ի հետ, և ստորև նկարագրված բոլոր օրինակները նախատեսված են cgroups v2-ի համար։
#### Cgroups-ի դերը systemd-ում և կոնտեյներներում

Ժամանակակից Linux համակարգերում cgroups-ը սերտորեն ինտեգրված է **systemd**-ի հետ.

- Systemd-ն յուրաքանչյուր **service** (ծառայության) համար ավտոմատ ստեղծում է առանձին cgroup, ինչը թույլ է տալիս դիտել նրա ռեսուրսների ծախսը և ծառայությունը կանգնեցնելիս անվտանգ դադարեցնել դրա բոլոր պրոցեսները (մեկ հրամանով, առանց «մոռացված» child process-ների)։
- Systemd-ը կազմակերպում է cgroups-ները **slices**-ների մեջ. `system.slice` (համակարգային ծառայություններ), `user.slice` (օգտատիրոջ սեսիաներ), `machine.slice` (վիրտուալ մեքենաներ, կոնտեյներներ)։ Այս slice-ների վրա կարելի է դնել ընդհանուր սահմանափակումներ, իսկ ծառայությունը կարող է նշանակվել կոնկրետ slice-ի մեջ։
- Կոնտեյներների տեխնոլոգիաները (օրինակ՝ Docker, LXD) ևս հիմնված են cgroups-ի վրա՝ յուրաքանչյուր կոնտեյների համար ռեսուրսների սահմանափակում և մեկուսացում ապահովելու համար։ Namespace-ները մեկուսացնում են «ինչ է տեսնում» կոնտեյները, իսկ cgroups-ը սահմանափակում է «ինչքան կարող է օգտագործել»։

#### Cgroups-ով կառավարվող հիմնական ռեսուրսները

| Ռեսուրս | Նկարագրություն | systemd-ի հատկություն |
| --- | --- | --- |
| CPU | Պրոցեսորի ժամանակի սահմանափակում (քվոտա) կամ հարաբերական քաշ | `CPUQuota`, `CPUWeight` |
| Հիշողություն (Memory) | Հիշողության օգտագործման կոշտ կամ փափուկ սահման | `MemoryMax`, `MemoryHigh` |
| I/O | Սկավառակի կարդալու/գրելու արագության և քաշի սահմանափակում | `IOWeight` |
| PIDs | Cgroup-ում ստեղծվող պրոցեսների քանակի սահմանափակում (պաշտպանություն «fork bomb»-ից) | `TasksMax` |
| cpuset | Որ CPU միջուկները կարող են օգտագործել | `AllowedCPUs` |

Cgroups-ի միջերեսը Linux-ում հասանելի է որպես վիրտուալ ֆայլային համակարգ `/sys/fs/cgroup/` հասցեում. այնտեղ կարելի է ուղղակիորեն ստեղծել cgroups և կարգավորել սահմանափակումները։ Սակայն Ubuntu-ում խորհուրդ է տրվում սահմանափակումները կառավարել **systemd**-ի միջոցով (ստորև նկարագրված), որն ավտոմատ սպասարկում է cgroups-ի կառուցվածքը և ապահովում հետևողական վարքագիծ ծառայությունների հետ աշխատելիս։
### Հիմնական հրամաններ

| Հրաման | Նպատակ | Զգուշացում |
| --- | --- | --- |
| `mount \| grep cgroup` | Ստուգել, թե cgroups-ի որ տարբերակն է ակտիվ (v2, թե v1) | Անվտանգ, միայն կարդում է |
| `sudo systemctl edit <service>` | Բացել ծառայության override ֆայլը՝ ռեսուրսների սահմանափակումներ ավելացնելու համար | Խմբագրումը պետք է ճիշտ սինտաքսով լինի. սխալը կարող է ծառայությունը չգործարկել |
| `sudo systemctl daemon-reload` | Վերաբեռնել systemd-ի կոնֆիգուրացիան unit-երի փոփոխությունից հետո | Անհրաժեշտ է unit-ի փոփոխությունից հետո, հակառակ դեպքում փոփոխությունը չի կիրառվի |
| `systemctl show <service>` | Ցույց է տալիս ծառայության բոլոր cgroups-հատկությունները | Անվտանգ, միայն կարդում է |
| `systemctl show <service> --property=CPUQuota,MemoryMax,IOWeight` | Ցուցադրել կոնկրետ սահմանափակումները | Անվտանգ, միայն կարդում է |
| `sudo systemd-run --scope -p MemoryMax=... <cmd>` | Գործարկել մեկ հրաման ժամանակավոր սահմանափակումներով | `--scope`-ով հրամանը կապվում է ձեր սեսիայի հետ. զգույշ եղեք, որ ճիշտ հրաման եք գործարկում |
| `systemd-cgtop` | Իրական ժամանակում ցույց է տալիս cgroups-ների ռեսուրսների օգտագործումը | Ինտերակտիվ գործիք. ելքին՝ `q` |
| `cat /sys/fs/cgroup/.../memory.current` | Ուղղակիորեն կարդալ cgroup-ի ընթացիկ հիշողության օգտագործումը | Անվտանգ, միայն կարդում է |
| `sudo systemctl set-property <unit> <property>=<value>` | Սահմանափակումների դինամիկ փոփոխություն առանց ֆայլի խմբագրման | Փոփոխությունը կիրառվում է անմիջապես. զգույշ եղեք արժեքների հետ |

### Փորձարկում (Lab)

Այս փորձը նախատեսված է սովորական Linux VM-ի (օրինակ՝ Ubuntu 22.04+ cgroups v2-ով) վրա. չօգտագործեք production սերվեր և իրական գաղտնիքներ։ Թիրախը՝ սահմանափակումներ դնել, դրանք ավելացնել և հետևելն է.

#### Քայլ 1. Ստուգել cgroups v2-ը

```bash
mount | grep cgroup
# Սպասվող. cgroup2 on /sys/fs/cgroup type cgroup2 ...
```

#### Քայլ 2. Ծառայության համար ռեսուրսների սահմանափակում (systemd)

Ստեղծեք փորձնական unit-ի override-ը՝ սահմանափակումներ ավելացնելու համար.

```bash
sudo systemctl edit cgroup-lab.service
```

Եթե `cgroup-lab.service` unit-ը դեռ գոյություն չունի, `systemctl edit`-ը միայն override-ի դրվագ է ստեղծում. ամենաճիշտը սկզբում ստեղծել պարզ test service-ը՝ ավելի ուշ ջնջելու պարզությամբ՝ օրինակ՝ `systemd-run`-ով (տես Քայլ 3)։ Այնուամենայնիվ, override-ի պարունակությունը հետևյալն է.

```ini
[Service]
CPUQuota=50%
MemoryMax=512M
MemoryHigh=400M
IOWeight=50
```

Ապա `daemon-reload` և (անհրաժեշտության դեպքում) restart, և ստուգեք.

```bash
sudo systemctl daemon-reload
sudo systemctl restart cgroup-lab.service
systemctl show cgroup-lab.service --property=CPUQuota,MemoryMax,MemoryHigh,IOWeight
```

Սպասվող արդյունք. նշված հատկությունները ցուցադրվում են կիրառված արժեքներով, իսկ ծառայությունն աշխատում է այդ սահմանների ներքո։ Ուշադրություն՝ `CPUQuota=50%`-ը նշանակում է մեկ CPU միջուկի 50%-ը (այսինքն՝ միջուկի կեսը)։ 100% = մեկ ամբողջ միջուկ, 200% = երկու միջուկ, և այլն։
#### Քայլ 3. Ժամանակավոր սահմանափակում մեկ հրամանի համար (`systemd-run`)

Եթե ցանկանում եք արագ գործարկել մեկ հրաման՝ սահմանափակումներով, առանց unit ֆայլ խմբագրելու.

```bash
sudo systemd-run --scope -p MemoryMax=512M -p CPUQuota=50% sleep 30
```

Սպասվող արդյունք. հրամանը (`sleep 30`) գործարկվում է ժամանակավոր cgroup-ի ներսում՝ տվյալ սահմանափակումներով, և այն երևում է հաջորդ քայլի մոնիտորինգում։ Ուշադրություն՝ միշտ նշեք անմեղ հրաման (օրինակ՝ `sleep`), ոչ թե կործանարար բան՝ սահմանափակումների ազդեցությունը ստուգելու համար։

#### Քայլ 4. Մոնիտորինգ

```bash
systemd-cgtop
# և կոնկրետ cgroup-ի ընթացիկ հիշողությունը.
cat /sys/fs/cgroup/system.slice/cgroup-lab.service/memory.current
```

Սպասվող արդյունք. `systemd-cgtop`-ում երևում է ձեր unit-ի/հրամանի cgroup-ը՝ ընթացիկ CPU/RAM օգտագործմամբ, իսկ `memory.current`-ը ցույց է տալիս իրական ընթացիկ արժեքը (բայթերով)։
#### Քայլ 5. Դինամիկ փոփոխություն

```bash
sudo systemctl set-property --runtime cgroup-lab.service CPUQuota=25%
systemctl show cgroup-lab.service --property=CPUQuota
```

Սպասվող արդյունք. քվոտան փոխվում է անմիջապես՝ առանց ֆայլի խմբագրման։ `--runtime`-ով փոփոխությունը ժամանակավոր է և կկորչի վերագործարկումից հետո. սա օգտակար է արագ փորձերի համար։ Ավարտելուց հետո մաքրեք փորձնական unit-ը՝ `sudo systemctl stop cgroup-lab.service`-ով, անհրաժեշտության դեպքում՝ `sudo systemctl reset-failed cgroup-lab.service`-ով, իսկ `systemctl edit`-ով ստեղծած override-ը ջնջեք `/etc/systemd/system/cgroup-lab.service.d/` պանակից՝ այնպես, որ test-ից ոչինչ չմնա համակարգում.

!!! note "MemoryMax vs MemoryHigh"
    - `MemoryMax` — **կոշտ** սահման (hard limit). այն գերազանցելու դեպքում kernel-ի OOM killer-ը սպանում է cgroup-ի պրոցեսը/ծառայությունը՝ դրանով հետ պահելով համակարգի ամբողջական քանդումից։
    - `MemoryHigh` — **փափուկ** սահման (soft limit). այն գերազանցելու դեպքում kernel-ը սկսում է «ճնշել» (throttle) ծառայության հիշողությունը՝ ակտիվորեն ազատելով (reclaim), և հիշողության ճնշման (memory pressure) պայմաններում կարող է սպանել կամ swap նետել։ Օգտագործեք այն՝ ծառայությունը զսպելու համար՝ առանց անհապաղ termination-ի, որպեսզի այն կայուն աշխատի սահմանի ներսում՝ չնայած հիշողության աճին։
### Իրական DevOps իրավիճակ

#### Ախտանիշ

Նույն host-ի վրա աշխատում են մի քանի ծառայություն/կոնտեյներ։ Դրանցից մեկում (օրինակ՝ հավելված՝ հիշողության արտահոսքով / memory leak) սկսում է անդադար աճել RAM-ի օգտագործումը. շուտով host-ի ամբողջ հիշողությունը սպառվում է, մյուս ծառայությունների արձագանքման ժամանակը կտրուկ աճում է, հայտնվում են OOM-ի նշաններ, և հատկապես «անմեղ» ծառայությունը սկսում է crash անել կամ ավտոմատ վերագործարկվել (restart loop)։

#### Ախտորոշում

1. Տեսեք, թե որ cgroup-ն է սպառում հիշողությունը.

   ```bash
   systemd-cgtop
   ```

2. Մեկ-մեկ ստուգեք կասկածելի slice/ծառայությունների cgroups-ի ընթացիկ և սահմանային արժեքները.

   ```bash
   systemctl show <service> --property=MemoryCurrent,MemoryMax
   cat /sys/fs/cgroup/system.slice/<service>/memory.current
   ```

3. Հաստատեք, որ աճի աղբյուրը տվյալ ծառայությունն է (leak-ը), այլ ոչ թե համակարգային այլ պրոցես. `top`/`ps`-ով գտեք առավել RAM-ագույն պրոցեսը և համոզվեք, որ այն պատկանում է այն slice-ին, որը մեծանում է `systemd-cgtop`-ում։

#### Լուծում

Հիմնական լուծումը՝ սահմանափակումներ դնել, որպեսզի անսարք ծառայությունը, նույնիսկ leak-ի դեպքում, չկարողանա «ուտել» host-ի բոլոր ռեսուրսները.

```bash
sudo systemctl edit <service>
```

```ini
[Service]
MemoryHigh=1G
MemoryMax=1.5G
TasksMax=512
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart <service>
```

Արդյունք. եթե ծրագիրը գերազանցի 1G-ը, kernel-ը սկզբում «ճնշում» (throttle) է անում՝ `MemoryHigh`-ի շնորհիվ, իսկ 1.5G-ից հետո OOM killer-ը սպանում է միայն այս ծառայությունը՝ առանց մյուսների վրա ազդելու։ Այս կերպ սահմանափակվում է «բլաստ շառավիղը» (blast radius)։ Զգույշ եղեք՝ սահմանը չափազանց ցածր չդնեք՝ նախ հաշվարկեք ծառայության իրական baseline հիշողությունը այլապես նորմալ աշխատող ծառայությունը կսկսի crash անել, երբ էլ հատի սահմանը՝ պատճառելով նոր տիպի խնդիր։

#### Կանխարգելում

- Բոլոր ծառայությունների և կոնտեյներների համար դրեք `MemoryMax`/`MemoryHigh`, իսկ անհրաժեշտության դեպքում՝ `CPUQuota` և `TasksMax` (fork bomb-ից պաշտպանության համար)։
- Մոնիտորինգ դրեք cgroups-ի ռեսուրսների օգտագործման վրա և alert՝ սահմանի 80%-ին հասնելիս՝ մոտալուտ սպառման մասին նախազգուշացնելու համար։
- Capacity planning. հաշվարկեք յուրաքանչյուր ծառայության baseline-ը, և վերապահեք host-ի ռեսուրսներից headroom՝ spike-ների համար. համոզվեք, որ բոլոր սահմանների գումարը չի ձգտում ֆիզիկական ռեսուրսներին։
- Թարմացրեք runbook-ը՝ նկարագրելով, թե որ slice/cgroup-ն ինչ սահման ունի, ինչպես է ախտորոշվում ու վերականգնվում ռեսուրսների գերածախսը։

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ է cgroup-ը և ի՞նչ խնդիր է այն լուծում (mid-level)

Cgroup-ը (Control Group) Linux-ի kernel-ի մեխանիզմ է, որը պրոցեսները խմբավորում և սահմանափակում է, թե խմբերից յուրաքանչյուրը որքան ռեսուրս (CPU, հիշողություն, I/O, պրոցեսների քանակ) կարող է օգտագործել։ Այս կերպ մեկ անսարք կամ չարամիտ պրոցես չի կարող սպառել host-ի բոլոր ռեսուրսները և խաթարել մյուս ծառայությունները. սա ռեսուրսների մեկուսացման և կայունության հիմքն է՝ և՛ systemd-ում, և՛ կոնտեյներներում (Docker, LXD)։

#### Ի՞նչ տարբերություն կա cgroups v1-ի և v2-ի միջև, և ինչո՞ւ է v2-ը նախընտրելի (senior-level)

Հիմնական տարբերությունը կառուցվածքում է. v1-ը յուրաքանչյուր controller-ի (memory, cpu և այլն) համար ունի առանձին, անկախ հիերարխիա, որի պատճառով տարբեր ռեսուրսների սահմանափակումները դժվար է համակարգել մեկ cgroup-ի վրա։ v2-ն օգտագործում է մեկ միասնական հիերարխիա բոլոր controller-ների համար. դա հեշտացնում է կառավարումը, ապահովում է հետևողական վարքագիծ և «no internal processes» մոդելը (cgroup-ն ունի կա՛մ պրոցեսներ, կա՛մ ենթահիերարխիա՝ ոչ երկուսն էլ), և ծագում է ժամանակակից workflow-ների համար (systemd, Kubernetes, վերջին Docker)։ Դրա համար էլ v2-ը դարձել է նախընտրելի և լռելյայն տարբերակը ժամանակակից Ubuntu-ում նոր ռեսուրսների սահմանափակումների մեծ մասի համար։

### Ինքնաստուգում

1. Ի՞նչ է ցույց տալիս `mount | grep cgroup`-ի `cgroup2` տողի առկայությունը, և ո՞րն է տարբերությունը v1-ի և v2-ի ակտիվ լինելու միջև։
2. Ի՞նչ տարբերություն կա `MemoryMax`-ի և `MemoryHigh`-ի միջև, և ի՞նչ է կատարվում դրանցից յուրաքանչյուրի գերազանցման դեպքում։
3. Ի՞նչ է նշանակում `CPUQuota=200%`-ը, և ինչպե՞ս է այն կապված CPU միջուկների քանակի հետ։
4. Ի՞նչ slice-ներ կան systemd-ում (`system.slice`, `user.slice`, `machine.slice`) և ի՞նչ դեր ունեն ռեսուրսների խմբավորման մեջ։
5. Ի՞նչ հրամանով կգործարկես մեկ հրաման՝ ժամանակավոր հիշողության սահմանմամբ՝ առանց unit ֆայլ ստեղծելու, և ի՞նչ է տալիս `--scope`-ը։
6. Ի՞նչ ռեսուրսներ կարող է սահմանափակել cgroup-ը, և ինչպե՞ս է `TasksMax`-ը պաշտպանում «fork bomb» հարձակումից։
7. Ինչպե՞ս կախտորոշես, թե որ cgroup/ծառայությունն է սպառում host-ի հիշողությունը, և ի՞նչ alert-եր կդնես կանխարգելման համար։
8. Ի՞նչ դեր ունի `/sys/fs/cgroup/`-ը, և ինչո՞ւ է խորհուրդ տրվում cgroups-ով աշխատել systemd-ի միջոցով՝ այլ ոչ թե ուղղակի ֆայլային մանիպուլյացիայով։

### Հաջորդ քայլեր

Շարունակիր systemd թեմայով. cgroups-ը համակարգի ռեսուրսների կառավարման «ներսի» մեխանիզմն է, իսկ systemd-ը՝ այն միջերեսը, որով DevOps-ն ամենից հաճախ է աշխատում դրա հետ։ Այս գիտելիքը ուղիղ կապ ունի նաև [Docker և Kubernetes](containers.md) բաժնի հետ, որտեղ cgroups-ը կոնտեյներների ռեսուրսների սահմանափակման հիմքն է։

## Networking {#networking}

### Տեսություն

Ցանցային ինտերֆեյսը (**network interface**) այն «կետն» է, որով օպերացիոն համակարգը կապվում է ցանցի հետ. այն կարող է լինել ֆիզիկական սարք (լարային Ethernet քարտ, Wi-Fi ադապտեր) կամ վիրտուալ (loopback, VPN, բրիջ)։ Linux-ում սարքերի մասին տվյալներ ստանալու ու կառավարելու հիմնական միջոցներն են `iproute2`-ի `ip` հրամանը և `/sys/class/net/` վիրտուալ ֆայլային համակարգը (sysfs)։

#### Ինտերֆեյսի անունների «կանխատեսելի» սկզբունքը

Ժամանակակից Ubuntu-ներում (մոտավորապես 16.04-ից, իսկ 18.04-ից Netplan-ի անցումով) ցանցային ինտերֆեյսների անունները փոխվել են. ավանդական `eth0`, `wlan0` անունների փոխարեն կիրառվում է **Predictable Network Interface Names** («կանխատեսելի ինտերֆեյսի անվանում») սկզբունքը։ Այս մոտեցումը `udev`-ի միջոցով անունը կապում է սարքի **ֆիզիկական դիրքի** և (կամ) **MAC հասցեի** հետ, որպեսզի անունը կայուն լինի և չփոխվի բեռնման հայտնաբերման հերթականությունից կախված. հին մոտեցմամբ `eth0`-ն կարող էր «սայթաքել» մյուս սարքին, երբ սարքերը հայտնվում էին տարբեր կարգով, ինչը վտանգում էր firewall-ի կանոններն ու կարգավորումները։

Անվանումներն այժմ ունեն որոշակի կառուցվածք, որը թույլ է տալիս հասկանալ, թե ինչ սարքի հետ գործ ունենք.

| Նախածանց | Նշանակություն | Օրինակ |
| --- | --- | --- |
| `lo` | Loopback. համակարգի «ներքին» ցանցը, միշտ `127.0.0.1` / `::1` | `lo` |
| `enX...` | Ethernet (լարային ցանց) | `enp3s0`, `enp0s3` |
| `wlX...` | Wireless LAN (Wi-Fi) | `wlp2s0`, `wlo1` |
| `wwX...` | Wireless WAN (բջջային մոդեմ) | `wwan0` |
| `ens33` | Ethernet, որը միացված է PCI **s**lot `33`-ին | `ens33` |

Անվան մնացած մասը ցույց է տալիս սարքի ֆիզիկական դիրքը.

- `p` — PCI **bus** համարը, օր. `enp3s0`-ում `3`-ը PCI bus-ն է.
- `s` — PCI **slot** համարը, օր. `ens33`-ում `33`-րդ սլոտը.
- `d` — PCI **device/function** համարը (օր. `enp0s3f1` նշանակում է function 1)։
- `x` — անունը ծագում է **MAC** հասցեից, օր. `enx00:0c:29:ab:cd:ef`-ի կրճատում (հարմար, երբ սարքը չունի հստակ PCI դիրք)։
- `o` — onboard ինտերֆեյս (`en0`), երբ firmware-ի ACPI-ն տալիս է նշանակված անուն։

Հենց դա էլ վերծանում է `ens33`-ը. `en` = Ethernet, `s` = PCI-Express սլոտ, `33` = 33-րդ սլոտ. այսինքն՝ լարային ցանցային քարտ՝ 33-րդ PCI սլոտում։ Ճշգրիտ համարները կախված են մայր տախտակի/VM-ի ապարատային կոնֆիգուրացիայից, ուստի երկու «նույն» մեքենաների անունները կարող են տարբերվել. ձեր մեքենայում ինտերֆեյսը կարող է կոչվել `enp0s3`, `eth0` կամ այլ անունով՝ կախված ապարատից.

#### Ինտերֆեյսի անունները և տվյալները տեսնելը

Ձեր համակարգի ինտերֆեյսների անունները տեսնելու ամենահեշտ ձևը հետևյալ հրամաններից որևէ մեկի օգտագործումն է.

```bash
ip a
# կամ
ip addr show
```

Դրանք ցուցադրում են բոլոր ակտիվ ցանցային ինտերֆեյսները՝ անուններով, IP հասցեներով, MAC հասցեներով և վիճակով։ Մեկ այլ տարբերակ՝ դիտել `/sys/class/net/` պանակը.

```bash
ls /sys/class/net
```

Այս տարբերակը ցույց է տալիս միայն ինտերֆեյսների անունները (օր. `lo enp0s3 wlo1`). դա հարմար է սկրիպտներում՝ թույլ տալով աշխատել ցուցակի հետ՝ առանց հավելյալ tool-ների տեղադրման։

#### Ի՞նչ է `/sys/class/net/`-ը

`/sys/class/net/`-ը **sysfs**-ի պանակ է՝ Linux-ի վիրտուալ ֆայլային համակարգ, որը kernel-ը «ցույց է տալիս» որպես ֆայլեր ու պանակներ՝ առանց իրական սկավառակի վրա գրելու. դրանք իրական ֆայլեր չեն, այլ kernel-ի կողմից տրամադրվող ինտերֆեյս՝ համակարգի և սարքերի մասին տվյալներ ստանալու համար։ Այստեղ կա առանձին ենթապանակ՝ յուրաքանչյուր ինտերֆեյսի անունով, օրինակ.

```bash
ls /sys/class/net/enp0s3/
```

Այնտեղ կարող եք գտնել հետևյալ կարևոր ֆայլերը.

| Ֆայլ | Բովանդակություն |
| --- | --- |
| `address` | MAC հասցեն (ֆիզիկական հասցե) |
| `mtu` | Maximum Transmission Unit (մեկ փաթեթի առավելագույն չափը բայթերով) |
| `operstate` | Ինտերֆեյսի գործառնական վիճակը (`up`/`down`/`unknown`) |
| `speed` | Միացման արագությունը (Mbit/s-ով) |
| `duplex` | Կապի եղանակը (`full`/`half`) |
| `flags` | Ինտերֆեյսի դրոշները (կարգավիճակը kernel-ի տեսանկյունից) |
| `statistics/` | Վիճակագրություն. ստացված/ուղարկված բայթեր, փաթեթներ, սխալներ և այլն |

Օրինակներ.

```bash
cat /sys/class/net/enp0s3/address            # MAC
cat /sys/class/net/enp0s3/operstate          # up
cat /sys/class/net/enp0s3/speed              # 1000 (1 Գբիթ/վրկ)
cat /sys/class/net/enp0s3/duplex             # full
cat /sys/class/net/enp0s3/statistics/rx_bytes
cat /sys/class/net/enp0s3/statistics/tx_bytes
```

Բոլոր ինտերֆեյսների MAC հասցեները ցույց տալու սկրիպտային եղանակը.

```bash
for iface in /sys/class/net/*; do echo "$(basename "$iface"): $(cat "$iface/address")"; done
```

`/sys/class/net/`-ը (ինչպես ընդհանրապես sysfs-ը) տվյալների աղբյուր է, որով ծրագրերն ու administrator-ները ճշգրիտ տվյալներ են ստանում առանց հավելյալ գործիքների. այն հատկապես օգտակար է bash/Python սկրիպտներում և ավտոմատացման մեջ՝ ցանցի կարգավիճակը ստուգելու համար։ Ի տարբերություն `ip a`-ի՝ sysfs-ը ցույց է տալիս միայն ինտերֆեյսներին վերաբերող ելքը, մինչդեռ `nmcli`-ն (NetworkManager-ի հրամանը) ցույց է տալիս ցանցի կարգավորումներն ու միացումները այլ մակարդակում։

### Հիմնական հրամաններ

| Հրաման | Նպատակ |
| --- | --- |
| `ip a` / `ip addr show` | Ցույց տալ բոլոր ինտերֆեյսները՝ IP, MAC, վիճակով |
| `ip link show` | Ցույց տալ ինտերֆեյսների link-ի մակարդակի տվյալները (MAC, MTU, վիճակ) |
| `ls /sys/class/net` | Թվարկել ինտերֆեյսների անունները |
| `ip -s link show <iface>` | Ցույց տալ RX/TX բայթերը, փաթեթները, սխալներն ու կորցրած փաթեթները |
| `sudo ip link set <iface> up/down` | Միացնել/անջատել ինտերֆեյսը |
| `sudo ip addr add <ip>/<prefix> dev <iface>` | Ինտերֆեյսին IP հասցե նշանակել |
| `sudo ip addr del <ip>/<prefix> dev <iface>` | Ինտերֆեյսից IP հասցե հեռացնել |
| `sudo dhclient <iface>` | DHCP-ով այդ ինտերֆեյսի համար հասցե ստանալ |
| `cat /sys/class/net/<iface>/speed` | Ցույց տալ միացման արագությունը (Mbit/s) |
| `sudo netplan apply` | Կիրառել `/etc/netplan/`-ի YAML ֆայլերի փոփոխությունները (Ubuntu Server) |
| `lspci \| grep -i ethernet` | Ցույց տալ PCI Ethernet սարքերը՝ համոզվելու, որ քարտը հայտնաբերված է |

!!! note "`ip` հրամանի ազդեցության ժամկետը"
    `ip`-ով կատարված փոփոխությունները (օր. `ip addr add`, `ip link set`) ազդում են **միայն ընթացիկ աշխատող kernel**-ի վրա և **մշտական չեն**. դրանք կկորչեն վերագործարկումից կամ ցանցային ծառայության վերագործարկումից հետո։ Մշտական կարգավորումների համար Ubuntu Server-ում օգտագործվում է **Netplan**-ը։

#### «`sudo ip addr` մշտական է՞»՝ ip vs Netplan

Ոչ, `sudo ip addr`-ը մշտական չէ. այն ժամանակավոր փոփոխություն է, որը կվերանա վերագործարկումից հետո։ Ubuntu Server-ում մշտական կարգավորումների համար նախատեսված է **Netplan** գործիքը (Ubuntu 18.04 և ավելի նոր)՝ որն աշխատում է `/etc/netplan/` պանակում գտնվող YAML ֆայլերով և այն կիրառում systemd-networkd-ի կամ NetworkManager-ի միջոցով՝ կախված renderer-ից.

| Հատկանիշ | `sudo ip addr add ...` | `netplan apply` |
| --- | --- | --- |
| Ազդեցության ժամանակը | Ակնթարթային, անմիջապես կիրառվում է | Կիրառվում է հրամանի աշխատեցումից հետո |
| Մշտականություն | Ժամանակավոր (վերագործարկումից հետո կորչում է) | Մշտական (պահպանվում է բոլոր վերագործարկումներից հետո) |
| Կիրառման եղանակը | Փոփոխում է միայն ընթացիկ kernel-ի վիճակը | Կարդում և կիրառում է `/etc/netplan/`-ի YAML ֆայլերը |

Ճիշտ աշխատելակարգը. փնտրեք ձեր Netplan ֆայլը `/etc/netplan/`-ում (օր. `00-installer-config.yaml` կամ `50-cloud-init.yaml`), խմբագրեք այն `sudo`-ով, Հետո կիրառեք.

```bash
ls /etc/netplan/
sudo nano /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

Ստատիկ IP-ի YAML կառուցվածքը՝ օր. `ens33`-ի համար.

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.1.100/24   # Ձեր ցանկալի IP-ն
      routes:
        - to: default
          via: 192.168.1.1    # Ձեր ցանցի gateway-ը
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]   # DNS սերվերներ
```

!!! warning "Netplan-ը զգայուն է YAML-ի նահանջների նկատմամբ"
    Netplan-ի փոփոխությունից հետո չափազանց զգույշ եղեք, քանի որ YAML-ը շատ զգայուն է indent-ի (նահանջ) և `-` գծիկների նկատմամբ. սխալ `sudo netplan apply`-ից հետո ցանցը կարող է «կտրվել», հատկապես SSH-ով աշխատելիս։ Խորհուրդ՝ նախ «չոր» ստուգեք `sudo netplan generate`-ով, և apply-ից առաջ backup SSH session պահեք՝ վերականգնման համար, եթե ինչ-որ բան սխալ գնա.

Ամփոփելով. `ip`-ը լավ է արագ թեստավորման ու ժամանակավոր փոփոխության համար, իսկ մշտական կարգավորումների համար Ubuntu Server-ում միշտ օգտագործեք Netplan-ը։

#### Ubuntu Desktop. NetworkManager-ը և static IP-ն

Մինչ Ubuntu Server-ը լռելյայն օգտագործում է systemd-networkd-ը (Netplan-ի `renderer: networkd`), Ubuntu Desktop-ը ցանցի կառավարումը լռելյայն հանձնում է **NetworkManager**-ին։ Netplan-ի YAML ֆայլում այդ նշվում է `renderer: NetworkManager` տողով. renderer-ը որոշում է, թե որ գործիքն է իրականում «քշում» սարքերը` systemd-networkd-ը, թե NetworkManager-ը.

##### Ինչու Desktop-ի `/etc/netplan/`-ում երկու ֆայլ է կարևոր

| Ֆայլ | Դերը |
| --- | --- |
| `00-installer-config.yaml` | Ստեղծվում է Ubuntu-ի տեղադրման ժամանակ. հիմնական ցանցային կոնֆիգուրացիան (հաճախ DHCP) |
| `01-network-manager-all.yaml` | Ասում է Netplan-ին` ցանցի կառավարումը հանձնել NetworkManager-ին |

Netplan-ը կարդում և միացնում (merge) է `/etc/netplan/`-ի բոլոր YAML ֆայլերը **ֆայլանունի համարակիալ կարգով**, և ավելի ուշ ֆայլի արժեքը գերակշռում է ավելի վաղին։ Քանի որ `01-network-manager-all.yaml`-ը մշակվում է `00-installer-config.yaml`-ից հետո, նրա `renderer: NetworkManager`-ը «ծածկում» է installer-ի կարգավորումը։ Սովորաբար այս ֆայլը պարունակում է.

```yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
```

Սա նշանակում է, որ Desktop-ում IP-ն սովորաբար կարգավորում եք Settings > Network GUI-ով կամ `nmcli`-ով, այլ ոչ թե ուղղակիորեն YAML-ը խմբագրելով, քանի որ NetworkManager-ն է, որ կառավարում է սարքերը.

##### `renderer`-ի երկու արժեքը

| renderer | Ինչ է անում |
| --- | --- |
| `networkd` (լռելյայն) | Ինտերֆեյսները «քշում» է systemd-networkd-ը. հարմար է Server-ի և ֆայլ-հենված կոնֆիգուրացիայի համար |
| `NetworkManager` | Ինտերֆեյսները «քշում» է NetworkManager-ը. հարմար է Desktop-ի GUI/nmcli-ի համար |

##### Static IP-ն Desktop-ում. երկու եղանակ

**1. NetworkManager-ի միջոցով (խորհուրդ է տրվում Desktop-ի համար).** Քանի որ renderer-ը NetworkManager է, ամենատարածված ու կայուն եղանակը `nmcli`-ն է` առանց YAML-ին դիպչելու.

```bash
# Թվարկել միացումները (նշենք ձեր միացման անունը)
nmcli connection show

# Static IP նշանակել (անվանը փոխարինեք ձեր իրականով)
sudo nmcli connection modify "Wired connection 1" \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  ipv4.method manual

# Վերագործարկել միացումը, որ փոփոխությունը կիրառվի
sudo nmcli connection down "Wired connection 1" && sudo nmcli connection up "Wired connection 1"
```

- `ipv4.method manual`-ը միացումը DHCP-ից փոխադրում է «ձեռքով» (static) ռեժիմի.
- `ipv4.dns`-ը ընդունում է DNS-ների ցուցակը ստորակետով բաժանված.
- `nmcli connection down`/`up`-ը անջատում ու նորից միացնում է միացումը, որ փոփոխությունը ակտիվանա.

**2. Netplan YAML-ով (server-ոյի ոճ).** Եթե ուզում եք ամեն ինչ կառավարել YAML-ով, ստեղծեք ավելի բարձր համարով ֆայլ, որը կմշակվի ավելի ուշ` օր. `99-static-ip.yaml`.

```yaml
network:
  version: 2
  renderer: networkd   # Ինտերֆեյսը «հեռացնում» է NetworkManager-ից
  ethernets:
    enp0s3:            # Գտեք ճշգրիտ անունը `ip link show`-ով
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

!!! warning "Desktop-ում renderer-ը փոխելու հետևանքը"
    Եթե Desktop-ում renderer-ը դարձնեք `networkd`, այդ ինտերֆեյսը «դուրս է գալիս» NetworkManager-ի վերահսկողությունից, և GUI-ի Settings > Network-ի (և `nmcli`-ի) փոփոխություններն դրա համար այլևս չեն աշխատի։ Նախ փորձեք `sudo netplan try`-ով` 120 վայրկյան auto-revert պատուհանով` որպեսզի սխալի դեպքում կապը ավտոմատ իսկապես վերականգնվի.

##### `netplan try` vs `netplan apply`

| Հրաման | Վարքը |
| --- | --- |
| `sudo netplan generate` | «Չոր» ստուգում. վավերացնում է YAML-ը և «ծնում» backend-ի կոնֆիգուրացիան` առանց կիրառելու |
| `sudo netplan try` | Կիրառում է ժամանակավորապես (լռելյայն 120 վայրկյան), հարցնում հաստատում, չհաստատելու դեպքում ավտոմատ վերադարձնում (auto-revert) |
| `sudo netplan apply` | Կիրառում է անմիջապես ու մշտապես` առանց auto-revert-ի |

SSH-ով աշխատելիս `sudo netplan try`-ն ավելի անվտանգ է. եթե նոր կոնֆիգուրացիան «կտրում» է կապը, հրամանը ժամանակի լրացումից հետո ինքնուրույն վերադարձնում է հինը, և դուք չեք մնում «դուրս» սերվերից. ի հակառակն `netplan apply`-ը կիրառում է վերջնական, ուստի դրա դեպքում backup SSH session-ը պարտադիր է.

#### Ինտերֆեյսի անունը փոխելը

!!! warning "Զգույշ. անվան փոփոխությունը կարող է խաթարել ծառայությունները"
    Ինտերֆեյսի անունը փոխելը կարող է ազդել firewall-ի կանոնների, DHCP/DNS կարգավորումների և այն ամենի վրա, ինչը հղվում է հին անունով։ Արեք դա միայն անհրաժեշտության դեպքում և վերագործարկումից առաջ ստուգեք կարգավորումները՝ ներառյալ firewall-ի կանոններն ու այլ ծառայությունների աշխատանքը։

**1. Վերադարձ ավանդական `eth0`/`eth1` անուններին (ամենատարածված եղանակը)**

Սա ժամանակավորապես անջատում է նոր «կանխատեսելի» անվանման կանոնը.

```bash
sudo nano /etc/default/grub
```

`GRUB_CMDLINE_LINUX` տողում ավելացրեք պարամետրերը (փոխարինեք `...`-ը, եթե կան այլ պարամետրեր).

```
GRUB_CMDLINE_LINUX="... net.ifnames=0 biosdevname=0"
```

Թարմացրեք GRUB-ը, Հետո վերագործարկեք.

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot
```

UEFI համակարգերի դեպքում կոնֆիգուսի այրումը կարող է պահանջել այլ ուղի (օր. `/boot/efi/EFI/ubuntu/grub.cfg`)։ Արդյունքում ինտերֆեյսները կրկին կկոչվեն `eth0`, `eth1` և այլն։ Մի մոռացեք, որ «կանխատեսելի» անունները վերադարձնելու համար հետագայում պետք է հեռացնել այդ պարամետրերը և նորից թարմացնել GRUB-ը:

**2. Մշտական անվան փոփոխություն udev-ի կանոնով**

Այս մեթոդը թույլ է տալիս ինտերֆեյսին տալ ցանկացած անուն՝ կապելով այն MAC հասցեին, և ստեղծում է մշտական կանոն.

```bash
ip link show              # պարզեք MAC հասցեն
sudo nano /etc/udev/rules.d/70-persistent-net.rules
```

Ֆայլում ավելացրեք տողը (փոխարինեք MAC-հասցեն և ցանկալի անունը).

```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="xx:xx:xx:xx:xx:xx", NAME="my_interface"
```

Վերաբեռնեք udev-ի կանոնները և կիրառեք դրանք.

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo reboot
```

!!! note "`net.ifnames=0` vs udev կանոն"
    GRUB-ի `net.ifnames=0 biosdevname=0`-ը անջատում է ամբողջ «կանխատեսելի» համակարգը՝ բոլոր ինտերֆեյսներին վերադարձնելով `eth0`-ի ոճ, մինչդեռ udev-ի կանոնը թիրախավորում է մեկ կոնկրետ սարք MAC հասցեով։ Եթե մեքենայում կա մեկից ավելի ցանցային քարտ, udev-ով ավելի նպատակային եք աշխատում՝ չփոխելով մյուսների անունները։

### Փորձարկում (Lab)

Միջավայր. սովորական Ubuntu Linux VM (օր. VirtualBox/VMware, որտեղ ժամանակակից ադապտերները հաճախ ստանում են `ens33` կամ `enp0s3`) կամ ձեր test server-ը՝ առանց production credentials-ի։ Lab-ը բաղկացած է դիտման, ախտորոշման և ժամանակավոր փոփոխության քայլերից.

```bash
# 1. Թվարկել ինտերֆեյսների անունները
ls /sys/class/net

# 2. Մանրամասն տվյալներ՝ IP-ով
ip a

# 3. Ընտրեք ձեր Ethernet-ի անունը (օր. eth0/enp0s3/ens33) և դիտեք
ip link show <iface>
ip -s link show <iface>

# 4. Տվյալներ sysfs-ից
cat /sys/class/net/<iface>/address
cat /sys/class/net/<iface>/operstate
cat /sys/class/net/<iface>/speed
cat /sys/class/net/<iface>/duplex

# 5. Բոլոր ինտերֆեյսների MAC հասցեները
for iface in /sys/class/net/*; do echo "$(basename "$iface"): $(cat "$iface/address")"; done
```

Սպասվող արդյունքը. կտեսնեք ձեր ինտերֆեյսների անունները (`lo`, `enp0s3`/`ens33` և այլն), MAC հասցեները, ընթացիկ վիճակը (`up`/`down`) և արագությունը (100, 1000 և այլն)՝ ըստ hardware-ի, առանց որևէ permanent փոփոխության։ Այս քայլերից ոչ մեկը համակարգը չի փոխում. դրանք զուտ դիտողական են և անվտանգ.

Ժամանակավոր փորձ. ինտերֆեյսն անջատեք ու միացրեք, որպեսզի տեսնեք, թե ինչ է կատարվում (զգույշ. այս պահին IP-ն կկորի, ուստի արեք տեղական console-ից, ոչ SSH-ից).

```bash
sudo ip link set <iface> down
ip link show <iface>     # state DOWN
sudo ip link set <iface> up
ip link show <iface>     # state UP
```

!!! note "Best Practices"
    - Միշտ օգտագործեք `ip`-ը, ոչ թե հնացած `ifconfig`-ը. ժամանակակից Ubuntu-ում `net-tools` փաթեթն առանձին է տեղադրվում, և լռելյայն `ifconfig`-ը բացակայում է.
    - Permanent փոփոխությունների համար մշտապես օգտագործեք `/etc/netplan/`-ի YAML-ը, ոչ թե `ip addr`-ի ակնթարթային արդյունքը.
    - SSH-ով աշխատելիս, ցանցի ցանկացած փոփոխությունից (netplan apply, ip link down) առաջ backup SSH session պահեք, և apply-ից առաջ ստուգեք, թե արդյոք նոր IP-ն/անունը ճիշտ է. հեշտ է «կտրել» ձեր միակ կապը.
    - Firewall-ի կանոն գրելիս մի ունեցեք չափազանց կոշտ կախվածություն մեկ ինտերֆեյսի անվան վրա. անվան ցանկացած փոփոխություն կդադարեցնի կանոնը.
    - Սկրիպտներում նախընտրեք sysfs-ը (`/sys/class/net`), քան `ip`-ի output-ի parser-ը. sysfs-ի ֆայլերը կայուն են և կանխատեսելի.

**Netplan / NetworkManager lab (test VM, ոչ production).** Այստեղ `netplan try`-ն ավելի անվտանգ է, քան `netplan apply`-ը, քանի որ սխալ կոնֆիգուրացիայի դեպքում 120 վայրկյանում ավտոմատ վերադարձնում է (auto-revert) հին վիճակը։ Տնային test VM-ում կարող եք փորձել static IP դնել `nmcli`-ով և հաստատել `ip a`-ով.

```bash
# 1. Որ renderer-ն է ակտիվ (տես /etc/netplan/-ի ֆայլերը)
cat /etc/netplan/*.yaml
nmcli device status

# 2. Թվարկել միացումները NetworkManager-ի դեպքում
nmcli connection show

# 3. «Չոր» ստուգում՝ YAML-ը վավերացնելու համար (չի կիրառում)
sudo netplan generate

# 4. Անվտանգ կիրառում. 120 վայրկյան auto-revert պատուհան
sudo netplan try
```

Խորհուրդ. այս lab-ի հրամաններն «շոշափելի» փոփոխություն չեն անում, բայց `netplan try`/`apply`-ը կիրառում են ցանցային կոնֆիգուրացիա. արեք միայն console-ից կամ `netplan try`-ով, որպեսզի վրիպումը չ«կտրի» ձեր միակ SSH կապը։

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Ubuntu Server-ի վրա, վերագործարկումից հետո, վեբ հավելվածը դառնում է անհասանելի. բրաուզերը չի միանում, իսկ SSH-ը (հնարավորության դեպքում) աշխատում է։ Administrator-ը նախորդ գիշեր ավելացրել է firewall-ի կանոն, որը հղվում էր `ens33`-ին. բայց այս boot-ի ժամանակ ցանցային քարտը հայտնաբերվել է այլ PCI slot-ում, անվան փոխվելով, և մշտական IP-ն, որին հղվում էր firewall-ը, հատկացվել է մյուս ինտերֆեյսին կամ ամբողջությամբ կորել է.

#### Ախտորոշում

1. Թվարկել ինտերֆեյսները՝ որոշելու, թե ինչ է հասանելի.
   ```bash
   ip a
   ls /sys/class/net
   ```
2. Գտնել, թե որ ինտերֆեյսն ունի ցանկալի IP-ն, և որ անունով է այն «քշվում» boot-ից հետո.
   ```bash
   ip addr show | grep -E "^[0-9]+:|inet "
   cat /sys/class/net/*/operstate
   ```
3. Համոզվել, որ `/etc/netplan/`-ի YAML-ում ճիշտ անուն է օգտագործված, ինչը համապատասխանում է իրականին.
   ```bash
   cat /etc/netplan/*.yaml
   sudo netplan get
   ```
4. Ստուգել firewall-ի, DNS-ի և DHCP-ի կարգավորումների հղումները հին անունով/IP-ով.
   ```bash
   sudo ufw status verbose
   journalctl -u systemd-networkd --no-pager | tail -n 40
   ```

#### Լուծում

Կախված պատճառից.

- Եթե խնդիրը մշտական IP-ի կորստից է. տվեք կայուն անուն udev-ի կանոնով՝ կապված MAC-ին (տե՛ս վերևում), համոզվեք, որ Netplan-ի YAML-ում ճիշտ անունն եք օգտագործում, Հետո՝ `sudo netplan apply`.
- Եթե իսկապես անհրաժեշտ է հին, կայուն ոճի անուն. անջատեք «կանխատեսելի» անունները GRUB-ով (`net.ifnames=0 biosdevname=0`), թարմացրեք GRUB-ը և վերագործարկեք.
- Համոզվեք, որ firewall-ի կանոնն այժմ հղվում է նոր անվանը/հասցեին, և թեստավորեք կանոնը մինչև «կտրել» եք SSH-ը.
- Վերականգնելուց հետո ծառայությունը նորից հասանելի է դառնում, իսկ ինտերֆեյսի անունը կայուն է բոլոր հետագա boot-երի ընթացքում.

#### Կանխարգելում

- Ցանցի permanent կոնֆիգուրացիան միշտ պահեք `/etc/netplan/`-ում, ոչ թե `ip`-ի ակնթարթային փոփոխությունների վրա.
- Կայունություն երաշխավորելու համար օգտագործեք MAC-ի վրա հենված udev կանոն, երբ անվան կայունությունը կարևոր է (օր. NIC teaming, firewall policy).
- Firewall-ի կանոններում նախապատվությունը տվեք interface-ի մակարդակին (`ufw allow in on <iface>`) և պարբերաբար sync արեք կարգավորման ֆայլերը boot-ի փաստացի վիճակի հետ.
- Փոփոխություններից առաջ միշտ ունենաք out-of-band կամ backup SSH access, և ցանցային փոփոխություններից հետո գրանցեք ակնկալվող և փաստացի վիճակը runbook-ում.

#### Ախտանիշ (երկրորդ. Desktop-ում NetworkManager-ի «չկիրառվող» static IP)

Ubuntu Desktop-ի օգտատերը ցանկացավ static IP դնել և ուղղակիորեն խմբագրեց `/etc/netplan/00-installer-config.yaml`-ը. `sudo netplan apply`-ից հետո IP-ն չփոխվեց այնպես, ինչպես սպասվում էր. իսկ հետագայում, GUI-ի Settings > Network-ից IP փոխելուց հետո, կապը սկսեց երբեմն «կտրվել»։ Պատճառը. Desktop-ում ակտիվ renderer-ը NetworkManager-ն է (`01-network-manager-all.yaml`), և YAML-ով արված systemd-networkd-ի ոճի կարգավորումը չի համապատասխանում NetworkManager-ի «հսկողությանը»` երկու գործիք փորձում են կառավարել նույն ինտերֆեյսը։

#### Ախտորոշում

1. Որ renderer-ն է ակտիվ. ստուգեք `/etc/netplan/`-ի ֆայլերը.
   ```bash
   cat /etc/netplan/*.yaml
   ```
2. Ով է իրականում «քշում» սարքը.
   ```bash
   nmcli device status
   ```
3. Համոզվեք, որ Netplan-ի «ծնած» backend-ը համապատասխանում է այն գործիքին, որն իրականում վերահսկում է ինտերֆեյսը.

#### Լուծում

Խորհուրդ է տրվում Desktop-ում static IP դնել NetworkManager-ի միջոցով (`nmcli connection modify ... ipv4.method manual`, ոչ թե `systemd-networkd`-ի ոճի YAML-ով), եթե հստակ կարիք չկա `renderer: networkd`-ի համար. այդպես GUI-ն ու `nmcli`-ը շարունակում են ազդել միացման վրա։ Եթե ամեն դեպքում պետք է YAML-ը, «ծեծեք» սարքը մեկ գործիքի հսկողության տակ՝ ավելի բարձր համարով ֆայլում (օր. `99-static-ip.yaml`), և կիրառեք `sudo netplan try`-ով` 120 վայրկյան auto-revert պատուհանով, որպեսզի սխալը չ«կտրի» կապը ընդմիշտ.

#### Կանխարգելում

- Desktop-ի վրա նախընտրեք NetworkManager-ը (`nmcli`/GUI), իսկ `renderer: networkd`-ը թողեք Server-ի համար, որտեղ ֆայլ-հենված կոնֆիգուրացիան գերակշռող է.
- Մինչև վերջնական `netplan apply`-ը միշտ օգտագործեք `sudo netplan try`-ը (auto-revert) կամ backup SSH/console session.
- Պարբերաբար համոզվեք, որ յուրաքանչյուր ինտերֆեյս կառավարում է միայն մեկ գործիք.

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ է նշանակում `ens33` ինտերֆեյսի անունը, և որտե՞ղ կպարզես քո ինտերֆեյսների անունները (mid-level)

`ens33`-ը «կանխատեսելի» անվանման սկզբունքով (Predictable Network Interface Names) ստեղծված Ethernet-ի անուն է. `en` = Ethernet, `s` = PCI slot, `33` = 33-րդ սլոտ. այսինքն՝ լարային ցանցային քարտ՝ 33-րդ PCI սլոտում։ Տեսնելու համար, թե ինչ ինտերֆեյսներ կան, օգտագործում եմ `ip a` (կամ `ip link show`), ինչպես նաև `ls /sys/class/net`, որը հարմար է սկրիպտից ցուցակը վերցնելու համար. sysfs-ի `/sys/class/net/<iface>/`-ում կա MAC (`address`), վիճակ (`operstate`), արագություն (`speed`), MTU և `statistics/`, ինչը թույլ է տալիս ճշգրիտ տվյալներ ստանալ առանց հավելյալ գործիքների.

#### Ինչո՞ւ է `sudo ip addr add`-ը ժամանակավոր, և ինչ եղանակ կօգտագործես Ubuntu Server-ում մշտական IP-ի համար (senior-level)

`ip` հրամանը փոփոխում է միայն ընթացիկ աշխատող kernel-ի ցանցային կոնֆիգուրացիան. այն ոչ մի ֆայլում չի պահպանվում, ուստի վերագործարկումից հետո, երբ kernel-ը բեռնվում է զրոյից, այն անհետանում է։ Ubuntu Server-ում (18.04+) մշտական կարգավորումների համար օգտագործում եմ **Netplan**-ը. կարգավորում եմ `/etc/netplan/`-ի YAML ֆայլերը (օր. `00-installer-config.yaml`) IP-ն, gateway-ի և DNS-ի հետ, այնուհետև «չոր» ստուգում `netplan generate`-ով և կիրառում `sudo netplan apply`-ով։ Այս եղանակով կոնֆիգուրացիան պահպանվում է բոլոր boot-երի ընթացքում, իսկ `ip`-ը օգտագործում եմ միայն ակնթարթային թեստավորման ու ախտորոշման համար. այս երկուսը լրացնում են միմյանց՝ տարբեր նպատակներով.

#### Ինչպե՞ս կապահովես, որ ինտերֆեյսի անունը կայուն մնա՝ անկախ boot-ի հերթականությունից (senior-level հավելյալ)

Կայունության հիմքը «կանխատեսելի» անվանումն է կամ udev-ի MAC-ի վրա հենված կանոնը. ամենահուսալին PCI slot-ի վրա հիմնված անունն է, որը տալիս է «կանխատեսելի» սկզբունքը, իսկ անհրաժեշտության դեպքում՝ `/etc/udev/rules.d/70-persistent-net.rules` կանոնը, որում `ATTR{address}` (MAC) կապվում է ցանկալի `NAME`-ի հետ։ Երբ ցանցային քարտը զբաղեցնում է ֆիքսված PCI slot (VM-ում հաճախ այդպես է), անունը կայուն է. իսկ ֆիզիկական տեղափոխության դեպքում՝ MAC-ի վրա հենված udev կանոնը տալիս է երաշխիք, որ firewall-ն ու Netplan-ը միշտ կճանաչեն ինտերֆեյսը՝ առանց անվան պատճառով կապի կորստի.

#### Ինչո՞ւ Ubuntu Desktop-ում `renderer: NetworkManager`-ն է լռելյայն, և ինչ հետևանք ունի այն `networkd`-ի փոխելը (mid-level)

Ubuntu Desktop-ը լռելյայն տրամադրում է `/etc/netplan/01-network-manager-all.yaml` ֆայլը `renderer: NetworkManager`-ով. այս renderer-ը ասում է Netplan-ին, որ ցանցի կառավարումը հանձնի NetworkManager-ին, որպեսզի օգտատերը IP-ն կարգավորի GUI-ից (Settings > Network) կամ `nmcli`-ով։ Քանի որ Netplan-ը ֆայլերը միացնում է ֆայլանունի համարակիալ կարգով, և ավելի ուշ ֆայլը գերակշռում է, `01-network-manager-all.yaml`-ը մշակվում է `00-installer-config.yaml`-ից հետո և «ծածկում» է նրա renderer-ը։ Եթե Desktop-ում renderer-ը փոխեմ `networkd`, այդ ինտերֆեյսի կառավարումը անցնում է systemd-networkd-ին, և NetworkManager-ի (GUI/nmcli) փոփոխություններն դրա համար այլևս չեն ազդում. երկու գործիք կարող են միաժամանակ «վիճել» մեկ սարքի համար, ուստի Desktop-ում ավելի ապահով է մնալ `NetworkManager`-ի մոտ, քան փոխել renderer-ը.

### Ինքնաստուգում

1. Ի՞նչ է ցույց տալիս `ls /sys/class/net`-ը, և ինչո՞վ է այն տարբերվում `ip a`-ից։
2. Վերծանի՛ր `enp3s0` և `ens33` անունները. որո՞նք են `en`-ի, `p`-ի, `s`-ի և համարների նշանակությունները։
3. Ի՞նչ տվյալներ կարելի է կարդալ `/sys/class/net/<iface>/`-ի `address`, `operstate`, `speed`, `duplex` ֆայլերից։
4. Ինչո՞ւ է `sudo ip addr add`-ը ժամանակավոր, և ի՞նչ tool-ով ես Ubuntu Server-ում մշտական IP կարգավորում։
5. Ի՞նչ է անում `sudo netplan apply`-ը, և ինչո՞ւ է YAML-ի նահանջների մասին զգուշացումը կարևոր SSH-ով աշխատելիս։
6. Ի՞նչ տարբերություն կա GRUB-ի `net.ifnames=0 biosdevname=0`-ի և udev-ի MAC-ի վրա հենված կանոնի միջև՝ ինտերֆեյսի անունը փոխելիս։
7. Ինչպե՞ս կգտնես խնդիրը, երբ վերագործարկումից հետո firewall-ը դադարում է «տեսնել» քո ինտերֆեյսը սխալ անվան/հասցեի պատճառով (ախտորոշման քայլերը)։
8. Ինչո՞ւ է Desktop-ի `/etc/netplan/01-network-manager-all.yaml` ֆայլի `renderer: NetworkManager`-ը «ծածկում» `00-installer-config.yaml`-ը, և ինչ դեր ունի ֆայլերի մշակման կարգը։
9. Ինչպե՞ս static IP դնել Ubuntu Desktop-ում `nmcli`-ով, և որ հրամանը կիրառել մինչև վերջնական `netplan apply`-ը, որ SSH-ով կապը չկտրվի (auto-revert)։

### Հաջորդ քայլեր

Այս էջում նաև խորը բաժին կա Desktop-ի static IP-ի մասին (Netplan-ի renderer-ը, NetworkManager-ը, `nmcli`-ը, `netplan try`/`apply`-ն)։ Շարունակիր ցանցի «վերևի» շերտերով. [Bash Scripting](#bash-scripting)-ով սովորիր ցանցի դիտարկումն ու ախտորոշումը ավտոմատացնել՝ օգտագործելով այստեղ նկարագրված sysfs-ի կայուն ֆայլերը կամ `ip`-ը, իսկ systemd-ով ծառայությունների կառավարման համար տես [Processes](#processes)։ Ցանցի խորը թեմաների (TCP/IP, DNS, firewalls, TLS) համար անցիր Networks բաժինը, որտեղ քննարկվում են ցանցային արձանագրություններն ու անվտանգությունը ամբողջ ցանցային շղթայի երկայնքով.

## Firewall {#firewall}

### Տեսություն

**Firewall-ը** (հայերեն՝ «պատնեշ») համակարգի կամ ցանցի բաղադրիչ է, որը վերահսկում է, թե ինչ մուտքային (incoming) և ելքային (outgoing) կապեր են թույլատրվում։ Սերվերի վրա դա սովորաբար **host-based firewall** է, որն աշխատում է Linux-ի միջուկի (kernel) ներսում՝ դեռ ծրագրերին հասնելուց առաջ ֆիլտրելով ցանցային փաթեթները։

#### Ubuntu-ի լռելյայն firewall-ը՝ UFW

Ubuntu-ի վերջին տարբերակներում (ներառյալ 22.04 / 24.04 LTS) լռելյայն firewall-կառավարման գործիքը **UFW**-ն է (**U**ncomplicated **F**irewall՝ «ոչ բարդ firewall»)։ Կարևոր է հասկանալ, որ UFW-ն **frontend** է (միջերես), ոչ թե առանձին firewall. այն պարզ հրամաններով կառավարում է kernel-ի իրական firewall-մեխանիզմը, որպեսզի օգտատերը ստիպված չլինի ուղղակի աշխատել բարդ iptables-ի հետ։

Երկու կարևոր փաստ UFW-ի մասին.

- **Նախապես տեղադրված է** (available by default) Ubuntu-ի բոլոր տեղադրումներում (8.04-ից մինչև այսօր)։
- **Լռելյայն անջատված է** (inactive). Տեղադրելուց հետո բոլոր մուտքային կապերը բաց են, մինչև այն ձեռքով միացնենք։ Սա զգուշավոր քայլ է, որ նոր համակարգը «կողպված» չմնա դեռևս կարգաբերված չլինելու պատճառով. միացնում ենք `sudo ufw enable` հրամանով:

#### UFW-ի հենքը՝ nftables

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

#### Հիմքում ընկած համակարգը՝ Netfilter, hooks, tables, chains, rules

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

#### Ի՞նչ է ցույց տալիս `type`-ը

Base chain-ը պարտադիր ունի `type` դաշտը, որն ասում է, թե ինչով է զբաղվում շղթան.

| `type` | Դերը | Ինչ verdict-ներ է թույլատրվում |
| --- | --- | --- |
| `filter` | Զտում (`accept`/`drop`/`reject`) | Ամենատարածվածը՝ INPUT/OUTPUT/FORWARD-ի համար |
| `nat` | Հասցեների փոխարկում | `snat`, `dnat`, `masquerade` (միայն նախատեսված hook-երում) |
| `route` | Փաթեթի վերաուղղորդում (հազվադեպ) | `dup`, `fwd` |

`type`-ը աշխատում է նաև `priority`-ի հետ, ինչը որոշում է կատարման հերթականությունը նույն hook-ի վրա. օրինակ՝ NAT-ն աշխատում է filter-ից առաջ՝ որովհետև նախ պետք է փոխել հասցեն, հետո զտել։ Regular chain-երը `type` չունեն՝ քանի որ ժառանգում են իրենց կանչող base chain-ի համատեքստը։

#### Sets. մեծ ցուցակների արդյունավետ կառավարում

nftables-ի ամենաուժեղ հնարավորություններից մեկը **sets**-ն է (հավաքածուներ)՝ IP-ների, պորտերի և այլ արժեքների ցուցակ, որը kernel-ում պահվում և որոնվում է որպես մեկ կառուցվածք։ Սա հատկապես օգտակար է մեծ ցուցակների դեպքում՝ օրինակ՝ Cloudflare-ի IP-ների։ Մեկ set-ով մենք գրում ենք **մեկ** կանոն՝ փոխանակ յուրաքանչյուր subnet-ի համար առանձին տող գրելու.

```bash
# set-ի սահմանում և ավելացում
sudo nft add set inet filter cloudflare_v4 { type ipv4_addr; flags interval; }
sudo nft add element inet filter cloudflare_v4 { 104.16.0.0/13, 172.64.0.0/13, 162.158.0.0/15 }
# մեկ կանոն` ամբողջ set-ի համար
sudo nft add rule inet filter input ip saddr @cloudflare_v4 tcp dport { 80, 443 } accept
```

#### ct. connection tracking (կապերի հետևում)

`ct`-ն **connection tracking**-ի հապավումն է՝ kernel-ի մեխանիզմ, որը «հիշում է» կապերի վիճակը։ Այն թույլ է տալիս firewall-ին տարբերել՝ «սա վտանգավոր անծանոթն է» (new/invalid) և «սա մեր հին բարեկամն է» (established/related)։ Հիմնական վիճակները.

- `new` — նոր կապ (օրինակ՝ առաջին SYN-փաթեթը) ;
- `established` — արդեն հաստատված կապ՝ տրաֆիկի գերակշիռ մասը (օրինակ՝ սերվերի պատասխան փաթեթները) ;
- `related` — առնչվող կապ (FTP-ի տվյալային կապ, ICMP սխալ) ;
- `invalid` — անվավեր/կեղծ փաթեթ՝ հաճախ հաքերային ;

Գրեթե յուրաքանչյուր firewall-ի առաջին կանոնը `ct state { established, related } accept` է՝ արդեն հաստատված կապերը չմշակելու և արագություն ապահովելու համար։ `invalid`-ները սովորաբար անմիջապես `drop` են արվում։ Connection tracking-ի աղյուսակը կարելի է դիտել՝ `sudo conntrack -L` կամ `cat /proc/net/nf_conntrack` հրամանով։

#### Docker-ը և firewall-ը

Docker-ը ինքնուրույն է կառավարում firewall-ի կանոնները՝ iptables/nftables-ի միջոցով (հիմնականում NAT-ը և FORWARD-ը)։ Սա կարևոր նրբություն է.

- Docker-ը ստեղծում է իր `DOCKER`, `DOCKER-FORWARD`, `DOCKER-USER` շղթաները և՝ `FORWARD`-ի `policy drop`-ի ֆոնին՝ ավելացնում է ներքին ցանցերի միջև տրաֆիկի թույլտվություններ։
- Ձեր host firewall-ը (UFW/nftables) չի խանգարում Docker-ի ներքին ցանցերին, բայց վերահսկում է դեպի **host-ի 0.0.0.0** պորտերը եկող մուտքային տրաֆիկը (օրինակ՝ proxy-ի 80/443-ը)։
- Զգուշացում՝ «# Warning: ... do not touch!» տողը նշանակում է, որ այդ աղյուսակը կառավարում է Docker-ը/iptables-nft-ը. ձեռքով կարգավորվող են միայն ձեր աղյուսակները. իսկ Docker-ի աղյուսակները ջնջելը կկանգնեցնի Docker-ի աշխատանքը՝ և ջնջվածն էլ նորից կստեղծվի գործարկման ժամանակ:

#### Cloudflare-ի հետ աշխատանքը. իրական IP-ն «թաքցնելու» սխեման

Պրոդաքշն WordPress/nginx սերվերները հաճախ ծածկվում են **Cloudflare**-ով՝ DDoS-ից պաշտպանության համար։ Այս սխեմայում.

- Հաճախորդի հարցումը գնում է Cloudflare, այլ ոչ թե ուղիղ ձեր սերվերի IP-ին.
- Cloudflare-ը զտում է այն և հարցումն ուղարկում ձեր սերվերի իրական IP-ին՝ **Cloudflare-ի IP-ներից** (ցուցակը՝ `https://www.cloudflare.com/ips-v4` և `ips-v6`),
- Ձեր firewall-ը կարող է սահմանել, որ 80/443 պորտերը ընդունվեն **միայն** Cloudflare-ի IP-ներից՝ այսպես հաքերը չի կարող ուղիղ միանալ ձեր իրական IP-ին (կհայտնվի firewall-ի DROP-ի առաջ),
- SSH-ը (22) պահվում է բաց՝ ձեր կառավարման կապը չկորցնելու համար.

Ստուգման ժամանակ `nmap`-ը ցույց է տալիս, որ 22-ը `open` է (հասանելի է), իսկ 80-ը/443-ը՝ `filtered` (ոչ-Cloudflare աղբյուրից միանալու դեպքում)՝ հենց այդ «միայն Cloudflare» քաղաքականության արդյունքն է։

### Հիմնական հրամաններ

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

### Փորձարկում (Lab)

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

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Աշխատող սերվերում (Ubuntu, Docker + nginx) որոշվեց 80/443 պորտերը թողնել միայն Cloudflare-ի համար։ Սակայն `sudo nmap -sS -Pn -p 22,80,443 <server_ip>`-ի արդյունքում 22-ը `open` է, 80/443-ը՝ նույնպես `open` (ոչ `filtered`), և մյուս պորտերը ևս երևում են արտաքինից՝ չնայած Cloudflare-ի «միայն» կանոններին, որոնք պետք է ցույց տային `filtered`:

#### Ախտորոշում

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

#### Լուծում

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

#### Կանխարգելում

- Պահեք աշխատող կանոնակազմը `/etc/nftables.conf`-ում և `systemctl enable nftables`-ով ապահովեք boot-ի ժամանակ վերականգնումը.
- Բոլոր փոփոխություններից առաջ՝ dry-run (`nft -c`) և backup SSH/կոնսոլ.
- Պարբերաբար, որոշ կայուն IP-ից՝ nmap-ի սկան՝ որպես ռեգրեսիոն թեստ՝ ակնկալելով՝ 22-ը՝ `open`, իսկ 80/443-ը՝ `filtered` ոչ-Cloudflare-ից:
- Թարմացրեք runbook-ը՝ նկարագրելով՝ որ աղյուսակի/շղթայի վրա է հիմնված պետությունը, և ինչպես զատել nftables-ի և Docker-ի աղյուսակները:

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ է UFW-ն, և ո՞ր firewall-ն է իրականում աշխատում Ubuntu-ի նոր տարբերակներում (mid-level)

UFW-ն (Uncomplicated Firewall, «ոչ բարդ firewall») Ubuntu-ի լռելյայն firewall-ի **կառավարման frontend** գործիքն է՝ նախապես տեղադրված, բայց լռելյայն `inactive` վիճակում՝ մինչև `sudo ufw enable`-ը։ Իրական firewall-մեխանիզմը, որը ֆիլտրում է փաթեթները, kernel-ի **Netfilter**-ն է՝ ժամանակակից Ubuntu-ում (20.04-ից) կարգավորվող՝ **nftables**-ի միջոցով։ UFW-ն օգտագործում է `iptables-nft` համատեղելիության շերտը, ուստի նույն կանոնները երևում են և՛ `iptables -L`-ում, և՛ `nft list ruleset`-ում՝ տարբեր ձևաչափով։ Իսկապես ակտիվը՝ nftables-ն է (kernel-ի մակարդակ), մինչդեռ UFW-ն ու iptables-ը միայն կառավարման միջերեսներ են։

#### Ինչպե՞ս եք տարբերակում «open» և «filtered» պորտը, և ի՞նչ է այն ասում ձեր firewall-ի մասին (senior-level)

Nmap-ի համար՝ «open» նշանակում է, որ պորտը պատասխանում է SYN-ACK-ով (հասանելի է), «filtered»՝ որ firewall-ը DROP է անում (կամ պատասխան չի գալիս), ուստի պորտի վիճակն անորոշ է, բայց մուտքը մերժված է։ Այս երկուսի տարբերությունն ախտորոշիչ արժեք ունի. օրինակ՝ եթե 80/443-ը `filtered` են ձեր (ոչ-Cloudflare) IP-ից՝ ապա «միայն Cloudflare» քաղաքականությունն աշխատում է. իսկ եթե դրանք `open` են՝ ապա scope-ի սահմանափակումը թերի է (օրինակ՝ դատարկ `policy accept` base chain-ը nftables-ում, որը շրջանցում է ձեր `inet filter`-ը)։ Senior-ը կսկսի `sudo nft list ruleset`-ից՝ փնտրելով նման աղյուսակներ, և՝ միայն ապա՝ ջնջել service-ին չպատկանող, կասկածելի base chain-երը.

### Ինքնաստուգում

1. Ի՞նչ է UFW-ն, և ինչո՞ւ է այն լռելյայն `inactive` Ubuntu-ում՝ չնայած, որ «firewall-ը» տեղադրված է։
2. Ի՞նչ back-end է օգտագործում UFW-ն Ubuntu-ի 20.04+-ում՝ nftables, թե iptables, և ի՞նչ դեր ունի `iptables-nft`-ը.
3. Նշեք Netfilter-ի 5 hooks-ը և ինչի համար են դրանցից յուրաքանչյուրը (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING)։
4. Ի՞նչ տարբերություն կա Base chain-ի և Regular chain-ի միջև nftables-ում, և ի՞նչ դեր ունի `jump`-ը.
5. Ի՞նչ է `ct state { established, related }`-ը, և ինչո՞ւ են այն դնում input-ի ամենասկզբում։
6. Ի՞նչ է nftables-ի set-ը, և ինչո՞ւ է մեկ set ավելի արդյունավետ, քան Cloudflare-ի IP-ների համար առանձին կանոնների երկար ցուցակը։
7. Ինչո՞ւ է `sudo nft flush ruleset`-ը «վտանգավոր», և ինչպե՞ս է կոնֆիգը վերականգնվում boot-ից հետո (`systemctl enable nftables`)։
8. Ի՞նչ է ցույց տալիս nmap-ի `filtered` վիճակը, և ինչպե՞ս է այն կապված «միայն Cloudflare» firewall-քաղաքականության հետ.

### Հաջորդ քայլեր

Շարունակիր Networks բաժնով՝ հետագա՝ TCP/IP, routing, DNS և TLS-ի մասին. firewall-ը ցանցային շղթայի «վերջին դարպասն» է դեպի սերվեր։ Ավելի խոր՝ [Linux-ի ցանցային կարգավորումներ](#networking)՝ ինտերֆեյսների ու routing-ի համար։ Քանի որ Docker-ն ինքնուրույն է կառավարում iptables/nftables-ը, ֆայլը սերտորեն կապված է [Docker և Kubernetes](containers.md) բաժնի հետ՝ արժե կարդալ՝ նախքան կոնտեյներների ցանցային պորտերի մեկուսացումը պրակտիկայում կիրառելը։

## Filesystem {#filesystem}

### Տեսություն

Linux-ում ամեն ինչ ֆայլ է կամ գրացանակ (directory)։ Ֆայլերը, գրացանակները, symlink-ները, socket-ները, նույնիսկ սարքերը դասավորված են մեկ միասնական ծառ՝ սկսած արմատից `/`։ Համակարգում կողմնորոշվելու և ախտորոշելու համար ամենահաճախ օգտագործվող հրամանն է `ls`-ը, որը ցուցակում է ֆայլերն ու գրացանակները։

Գրացանակի վրա `ls`-ը նորմայում ցույց է տալիս նրա **ներսի պարունակությունը**, այսինքն ֆայլերն ու ենթագրացանակները, ոչ թե գրացանակի մասին ինֆորմացիան։ Սա հաճախ շփոթեցնող է, հատկապես այն դեպքերում, երբ պետք է գրացանակի սեփական հատկությունները։ Երկու կարևոր դրոշակ է լուծում այս շփոթը՝ `-ld` և `-R`:

- **`-l`** (`--format=long` «երկար ձևաչափ»)՝ յուրաքանչյուր ֆայլի համար ցույց է տալիս մանրամասն տող՝ ֆայլի տիպը և թույլտվությունները (permissions), սեփականատիրոջը (owner), խումբը (group), չափը, փոփոխման ամսաթիվը և անունը։ Առաջին նիշը ցույց է տալիս տիպը՝ `d` գրացանակ, `-` հասարակ ֆայլ, `l` symbolic link, և այլն։
- **`-d`** (`--directory` «գրացանակ»)՝ ասում է `ls`-ին՝ ցույց տուր հենց գրացանակը որպես սովորական ֆայլ, ոչ թե նրա պարունակությունը։ GNU coreutils-ի պաշտոնական փաստաթղթերի համաձայն `-d`-ը «ցուցակում է գրացանակների անուններն այնպես, ինչպես այլ տիպերի ֆայլերի, ոչ թե դրանց պարունակությունը» և, իմպից, չի հետևում command line-ում տրված symbolic links-ներին։
- **`-R`** (`--recursive` «ռեկուրսիվ»)՝ ցուցակում է բոլոր գրացանակների պարունակությունը ռեկուրսիվ կերպով, այսինքն ամփոփում է ամբողջ ծառան՝ բոլոր ներդրված ենթագրացանակներով մինչև ամենախոր մակարդակը:

Երբ միավորում ենք `-l`-ն ու `-d`-ն (`ls -ld`), ստանում ենք գրացանակի մասին **միայն մեկ մանրամասն տող**, որում առաջին նիշը `d`-ն է (գրացանակ), ապա permissions-ը, owner-ը, group-ը, չափը, ամսաթիվը և անունը։ Սա օգտագործվում է գրացանակի մասին ինֆորմացիա ստանալու համար՝ առանց նրա ներսում մտնելու կամ պարունակությունը բացելու։

### Հիմնական հրամաններ

| Հրաման | Նպատակ |
| --- | --- |
| `ls` | Ցուցակել գրացանակի պարունակությունը (ոչ թե ինքնը) |
| `ls -l` | Երկար ֆորմատ. մանրամասն info յուրաքանչյուր ֆայլի համար (permissions, owner, group, չափ, ժամանակ) |
| `ls -d` | Ցուցակել գրացանակն ինքնը՝ ինչպես սովորական ֆայլ |
| `ls -ld` | Գրացանակի մանրամասն info-ն (մեկ տող)` առանց պարունակությունը բացելու |
| `ls -R` | Ռեկուրսիվ ցուցակում. ամբողջ գրացանակային ծառը՝ բոլոր ենթագրացանակներով |
| `ls -ldR` | Յուրաքանչյուր գրացանակի մասին մանրամասն տող` փոխարինում է նրա պարունակությունը` և շրջում ամբողջ ծառան |
| `ls -la` | Բոլոր ֆայլերը, ներառյալ թաքնված `.`-ով սկսվողները, երկար ֆորմատով |
| `ls -lh` | Երկար ֆորմատ, չափերը մարդու համար ընթեռնելի (K, M, G) |

### Փորձարկում (Lab)

Միջավայր՝ ցանկացած Linux VM կամ Ubuntu container:

```bash
# 1. Ստեղծել թեստային ծառա
mkdir -p project/src project/tests
touch project/README.md project/src/main.py project/tests/test_main.py

# 2. Գրացանակի պարունակությունը (ոչ ինքնը)
ls -l project

# 3. Գրացանակի մասին մեկ մանրամասն տող՝ առանց ներս մտնելու
ls -ld project

# 4. Ամբողջ ծառան ռեկուրսիվ
ls -R project

# 5. Համեմատել. ա) պարունակությունը ծնողի վրա, բ) ինքնը՝ -ld-ով
ls -ldR project | head -20
```

Սպասվող արդյունք.

```text
$ ls -l project
total 8
-rw-r--r-- 1 user user    0 Sep  9 10:00 README.md
drwxr-xr-x 2 user user 4096 Sep  9 10:00 src
drwxr-xr-x 2 user user 4096 Sep  9 10:00 tests

$ ls -ld project
drwxr-xr-x 4 user user 4096 Sep  9 10:00 project

$ ls -R project
project:
README.md  src  tests

project/src:
main.py

project/tests:
test_main.py
```

Ուշադրություն. `ls -ld project`-ում տեսնում եք **մեկ** տող `d`-ով, ոչ թե ֆայլերի ցուցակ. այս տարբերությունն է `-l`-ի և `-ld`-ի հիմքը։ Բացի այդ, `ls -ldR`-ում ամեն գրացանակի մասին ցուցադրվում է նրա սեփական մանրամասն տողը, իսկ ներսը՝ առանձին բաժիններով:

!!! tip "Best Practices"
    - Գրացանակի owner-ը կամ permissions-ը ստուգելու համար օգտագործիր `ls -ld`, ոչ թե `ls -l` գրացանակի վրա. առաջինը ցույց է տալիս հենց գրացանակի տողը:
    - Ծառի կառուցվածքը արագ տեսնելու համար `ls -R`-ը, բայց մեծ գրացանակների վրա (`/usr`, `/var`) ելքը կարող է հսկայական լինել, ուստի սահմանափակիր `head`-ով կամ օգտագործիր `tree`, եթե այն տեղադրված է:
    - Թաքնված ֆայլերը տեսնելու համար `-a`-ն պարտադիր է, այլապես `.env`, `.git` և այլն չեն երևա:

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Դեպլոյ-սկրիպտը վերադարձնում է «permission denied» error-ը, երբ փորձում է գրել application-ի log գրացանակում։ Ընթացիկ օգտատիրոջը պետք է պարզի, թե ում սեփականատերը (owner) է գրացանակը և ինչ permissions ունի։

#### Ախտորոշում

1. Համոզվիր, որ գրացանակի անունը ճիշտ ես գրել.
   ```bash
   ls -d /var/log/application
   ```
2. Գրացանակի մասին մանրամասն info ստանալու համար օգտագործիր `-ld`, ոչ թե `-l`:
   ```bash
   ls -ld /var/log/application
   ```
3. Եթե գրացանակը symlink է, `-L`-ով ստուգիր, թե հղումը որը է ցույց տալիս.

#### Լուծում

```bash
# Տեսնել, թե ում է պատկանում և ինչ permissions ունի
ls -ld /var/log/application
# drwxr-xr-x 2 appuser appgroup 4096 Sep  9 10:00 /var/log/application

# Փոխել owner-ը, որ deployment user-ը կարողանա գրել
sudo chown deployer:appgroup /var/log/application
sudo chmod 2775 /var/log/application
```

#### Կանխարգելում

- Ծառայության գրացանակների owner-ի և permissions-ի մասին ենթադրություն մի արա. ստուգիր `ls -ld`-ով։ Մուտքը պահանջում է հատում բոլոր միջանկյալ մակարդակներում, ուստի ստուգիր նաև parent գրացանակների permissions:
- Դեպլոյ-ից հետո գրիր փոքր ստուգում, որը `ls -ld`-ով համեմատում է ակնկալվող owner-ի հետ:

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ տարբերություն է `ls -l`-ի և `ls -ld`-ի միջև գրացանակի վրա (mid-level)

`ls -l` գրացանակի վրա ցուցակում է նրա **պարունակությունը**՝ ներսի ֆայլերն ու ենթագրացանակները յուրաքանչյուրը սեփական մանրամասն տողով։ Իսկ `ls -ld`-ը ցույց է տալիս **հենց գրացանակի մասին մեկ մանրամասն տող**՝ owner-ը, group-ը, permissions-ը, չափը, առանց ներսը բացելու. առաջին նիշը `d`-ն է, որը ասում է, որ սա գրացանակ է. Օգտագործվում է գրացանակի հատկությունները ստուգելու համար՝ առանց պարունակությունը մեկնաբանելու.

#### Ո՞րն է `-R`-ի վտանգը մեծ ֆայլային համակարգերում և ինչպե՞ս այն մեղմացնել (senior-level)

`ls -R`-ը ռեկուրսիվ անցնում է բոլոր ենթագրացանակներով. մեծ կամ խորը ծառաների վրա (օր. `/usr`, `/var`, կոնտեյներ cache-ներ) ելքը կարող է հսկայական լինել, խճողել էկրանը կամ սպառել հիշողություն, իսկ symlink-ցիկլերի դեպքում կրկնել միևնույն պարունակությունը։ Բացի այդ, մեծ ծառաների վրա տարբեր մակարդակների բաժինների միջև տարբերությունը հեշտ չէ parser-ել, քանի որ դրանք առանձնանում են միայն վերնագրի տողերով. Մեղմացում. սահմանափակիր `head`-ով, խորությունը սահմանիր `find -maxdepth N`-ով, կամ օգտագործիր `tree`-ը. Ավտոմատացումներում ավելի կանխատեսելի է `find`-ը, որի ելքը (ոչ ֆորմատավորված paths) ավելի հեշտ պարսավորելի է, քան `ls -R`-ի բաժանված տեսքը.

#### Ի՞նչ է անում `ls -d`-ն, և ե՞րբ է այն օգտակար (ավելացված)

`-d`-ն ասում է `ls`-ին՝ գրացանակը ցուցակիր որպես սովորական ֆայլ, ոչ թե բացիր նրա պարունակությունը. Օգտակար է, երբ ստուգում ես՝ արդյոք գրացանակ գոյություն ունի, պետք է հաստատես, որ տվյալ անունը գրացանակ է (`d`-ով, ոչ թե `-`-ով), կամ պետք է համեմատել մի քանի գրացանակների permissions-ը առանց դրանց պարունակությունը թափելու.

### Ինքնաստուգում

1. Ի՞նչ ցույց կտա `ls -ld /tmp`-ն, և ինչո՞ւ առաջին նիշը `d`-ն է
2. Ո՞ր հրամանով կտեսնես գրացանակի owner-ին` առանց ներսը բացելու
3. Ի՞նչ է անում `-R`-ը, և ի՞նչ ռիսկ ունի մեծ ծառաների վրա
4. Ի՞նչ տարբերություն կա `ls -l`-ի և `ls -ld`-ի միջև գրացանակի վրա
5. Ի՞նչ կնշանակի, եթե `ls -ld somepath`-ը ցույց է տալիս `-`-ով սկսվող տող, ինչ եզրակացություն կանես, և ե՞րբ դա կարող է հանդիպել

### Հաջորդ քայլեր

Շարունակիր ֆայլային համակարգի թեմայով. ուսումնասիրիր permissions-ը (`chmod`, `umask`, sticky bit), symbolic links-ը և `find`/`du`/`df`-ը՝ ֆայլերի որոնման ու տարածության վերլուծության համար (տես նաև [Processes](#processes) և [Bash Scripting](#bash-scripting))։

Հատուկ permission բիթերին (SUID, SGID, sticky) անցիր [Special Bits](#special-bits) էջով. այնտեղ տես կապը `chmod`-ի թվային ձևաչափի հետ, shared folder-ի SGID + `umask` զույգը և SUID-ի անվտանգության ռիսկերը։

## Special Bits (SUID/SGID/Sticky) {#special-bits}

### Տեսություն

Linux-ում ֆայլերն ու գրացանակները (directories) ունեն 9 հիմնական թույլտվության բիթ՝ read (`r`), write (`w`), execute (`x`)՝ երեք խմբի համար. տեր (owner/user), խումբ (group), մնացածը (others)։ Այս 9 բիթերին գումարվում է ևս **3 հատուկ բիթ**, որոնք փոխում են համակարգի վարքը՝ **SUID** (Set User ID), **SGID** (Set Group ID) և **Sticky bit**։ Դրանք հանդիպում են երկու տեղում՝ `chmod`-ի թվային 4-նիշ ձևաչափում (առաջին նիշը) և `ls -l`-ի ելքի `x` դիրքերում՝ `s`, `S`, `t`, `T` տառերով։

Կարևոր հիմք. այս բիթերը **չեն ավելացնում** նոր թույլտվություն, այլ փոխում են **ում անունից** է կատարվում գործողությունը (SUID/SGID) կամ **ում ֆայլերը** կարելի է ջնջել (sticky)։ Դրանք permissions-ի մոդելի մաս են և չեն շրջանցում այն. արդյունավետ իրավունքները դեռ որոշվում են `r`/`w`/`x` բիթերով։

#### SUID (Set User ID)

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

#### SGID (Set Group ID)

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

#### Sticky bit

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

### Հիմնական հրամաններ

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

### Փորձարկում (Lab)

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

### Իրական DevOps իրավիճակ

#### Ախտանիշ

CI/CD deployment-ից հետո application-ը (սովորական `app` օգտատիրոջ տակ) կարողանում է ֆայլեր ստեղծել shared `/srv/shared/releases` գրացանակում, բայց report-ների generator-ը (մեկ այլ օգտատեր՝ `reporter`, նույն `devteam` խմբում) ստանում է «Permission denied», երբ փորձում է թարմացնել այնտեղի ֆայլերը։

#### Ախտորոշում

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

#### Լուծում

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

#### Կանխարգելում

- Shared գրացանակները ստեղծիր infrastructure-ի կոդով (Ansible/`cloud-init`)՝ SGID-ը և խումբը սկզբից դրված, ոչ թե ձեռքով ուղղումով։
- Ծառայությունների umask-ը սահմանիր systemd unit-ի `UMask=`-ով, որ ֆայլերը լռելյայն չստանան `644`։
- SUID/SGID ֆայլերի ցանկը գրանցիր baseline-ով և պարբերաբար համեմատիր `find / -perm -4000`-ի արդյունքի հետ. նոր SUID ֆայլը կարող է compromise-ի ազդանշան լինել։
- `/tmp`-ի և կոնտեյներների volume-ների վրա պահիր sticky-ն, իսկ հնարավորության դեպքում՝ նաև `nosuid,noexec` mount option-ները։

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ է անում SGID բիթը գրացանակի վրա, և ինչո՞ւ է դա կարևոր shared folder-ում (mid-level)

Գրացանակի SGID-ն այդ գրացանակում ստեղծվող նոր ֆայլերին ու ենթագրացանակներին տալիս է ոչ թե ստեղծողի primary group-ը, այլ գրացանակի group-ը։ Առանց դրա՝ մի քանի օգտատերեր, գրելով նույն գրացանակում, ստեղծում են տարբեր group-ներով ֆայլեր, և միմյանց ֆայլերը չեն կարողանում կարդալ կամ թարմացնել։ SGID + `umask 002`-ը ստանդարտ պատասխանն է shared folder-ի համար, և այն սովորաբար լրացվում է sticky bit-ով, եթե գրացանակում գրելու իրավունք ունի ավելի լայն շրջանակ։

#### Ինչո՞ւ է SUID root binary-ը համարվում privilege escalation-ի ռիսկ, և ինչպե՞ս կմեղմեիր այն production-ում (senior-level)

SUID root binary-ի ցանկացած անվտանգության թերություն (buffer overflow, անապահով ժամանակավոր ֆայլ, չմաքրված `PATH`-ով արտաքին ծրագրի կանչ) վերածվում է root-ի իրավունքների ձեռքբերման, քանի որ պրոցեսն աշխատում է EUID=0-ով։ Մեղմումը բազմաշերտ է. (1) SUID root ցանկը նվազագույնի հասցնել և աուդիտ անել `find / -perm -4000 -type f`-ով baseline-ի համեմատ (AIDE, `auditd`)։ (2) Ամբողջական root-ի փոխարեն օգտագործել նեղ մեխանիզմներ՝ Linux capabilities (`setcap`), sudo-ի կանոններ կամ polkit։ (3) Մեկուսացնել systemd-ի sandboxing-ով (`NoNewPrivileges=yes`, `ProtectSystem=strict`, `CapabilityBoundingSet=`) և AppArmor/SELinux-ով։ (4) `/tmp`, `/home` և կոնտեյներների volume-ները mount անել `nosuid,nodev,noexec`-ով։ (5) Պահել `fs.suid_dumpable=0`, որ SUID պրոցեսի core dump-ը չարտահոսի գաղտնիքները։ Այս ամենը կրկնվող գործընթաց է, ոչ թե մեկանգամյա կարգավորում։

### Ինքնաստուգում

1. Ինչ տարբերություն կա փոքրատառ `s`-ի և մեծատառ `S`-ի միջև `ls -l`-ի ելքում
2. Ինչո՞ւ SUID shell սկրիպտի վրա չի աշխատում
3. Ինչպե՞ս կստուգես, թե գրացանակում նոր ֆայլը կժառանգի՞ արդյոք ընդհանուր խումբը
4. Ի՞նչ է անում sticky bit-ը, և ո՞ր գրացանակների վրա է այն սովորաբար դրված
5. Ինչո՞ւ SGID-ն միայնակ բավարար չէ shared folder-ի համար (ինչ դեր ունի umask-ը)
6. Ինչպե՞ս կգտնես համակարգում բոլոր SUID և SGID ֆայլերը

### Հաջորդ քայլեր

Այս թեման permissions-ի մոդելի խորացումն է. շարունակիր [Filesystem](#filesystem)-ի հետ (`chmod`, `chown`, `umask`), ապա անցիր **ACL**-ներին (`setfacl`/`getfacl`), որոնք SGID-ից ավելի նուրբ լուծում են տալիս բազմաթիվ օգտատերերի և ծառայությունների համար, և **Linux capabilities**-ին՝ SUID-ի ավելի անվտանգ այլընտրանքին (տես նաև [Processes](#processes))։

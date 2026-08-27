# Linux Processes

## Տեսություն

**Program**-ը սկավառակի վրա գտնվող executable ֆայլ է։ Երբ kernel-ը այն գործարկում է, առաջանում է **process**՝ իր հիշողությամբ, PID-ով (process ID), բաց ֆայլերով և իրավունքներով։ Մի program-ը կարող է ունենալ շատ process-ներ։

Յուրաքանչյուր process ունի ծնող՝ parent process։ Process-ի PPID-ը (parent PID) օգնում է հասկանալ, թե ով է այն գործարկել։ Process-ը կարող է ունենալ մեկ կամ ավելի threads, որոնք կիսում են նույն process-ի հիշողությունը։

Linux-ում process-ի վիճակներից օգտակարներն են.

- `R` — running կամ runnable,
- `S` — interruptible sleep՝ սպասում է իրադարձության,
- `D` — uninterruptible sleep, սովորաբար I/O սպասում,
- `T` — stopped,
- `Z` — zombie. ավարտվել է, բայց ծնողը դեռ չի վերցրել exit status-ը։

### Ինչպե՞ս տարբերել CPU-ի և I/O-ի բեռը

**Load Average**-ը ցույց է տալիս ակտիվ process-ների միջին քանակը 1, 5 և 15 րոպեների ընթացքում (օրինակ` `load average: 8.45, 7.90, 6.20`)։ Այն միայն CPU-ի զբաղվածությունը չի չափում. հաշվում են և՛ `R` (running/runnable), և՛ `D` (uninterruptible sleep, սովորաբար I/O-ի սպասում) վիճակի process-ները։ Բարձր Load-ի պատճառը գտնելու համար նայիր `top`-ի `%Cpu` տողին.

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

**us vs sy, ring-եր և space-ներ.** `us`-ը ձեր ծրագրի «մաքուր» աշխատանքն է, իսկ `sy`-ը՝ kernel-ի աշխատանքը` ձեր ծրագրի խնդրանքով։ CPU-ի ring/privilege levels-ը (Ring 0-3) ապարատային պաշտպանության մեխանիզմ է. Ring 0-ում աշխատ է kernel-ը (`sy`), Ring 3-ում` ձեր process-ները (`us`), Ring 1-2-ը ժամանակակից Linux-ում գրեթե չեն օգտագործվում։ Երբ process-ին անհրաժեշտ է ֆայլի մուտք, system call-ի միջոցով CPU-ն Ring 3-ից անցնում է Ring 0։ «Space»-ը վերաբերում է հիշողությանը` User Space-ը` ձեր process-ները, Kernel Space-ը` միջուկի կառուցվածքը.

**Ախտորոշիչ գործիքներ.**

- `top` / `htop` — ամենահեշտ քայլ, Load Average-ն ու `%Cpu`-ն և process list-ը.
- `vmstat 1` — Virtual Memory Statistics. համակարգի ընդհանուր պատկեր (processes, CPU, memory, I/O, swap)` ամեն վայրկյան
- `mpstat -P ALL 1` — Multi-Processor Statistics. CPU-ի յուրաքանչյուր core-ի ծանրաբեռնվածությունը.
- `iostat -x 1` — Input/Output Statistics. device-ի %util-ը և await-ը.
- `iotop` — I/O Top. որի process-ները ծանրաբեռնում են սկավառակը.
- `ps -eo pid,stat,wchan:30,comm` — D-վիճակի (I/O-ին սպասող) process-ների ցուցակ.

`iostat`-ի `%util`-ը 100%-ին մոտ լինելը խոսում է սկավառակի գերբեռնվածության մասին, իսկ բարձր `await`-ը` դանդաղ I/O-ի նշան է. D-վիճակի process-ներն են Load Average-ի մեջ` ահա ցածր CPU-ի դեպքում բարձր Load-ի պատճառը։

## Հիմնական հրամաններ

| Հրաման | Նպատակ | Զգուշացում |
| --- | --- | --- |
| `ps aux` | Process-ների snapshot | Շատ output կարող է տալ |
| `ps -o pid,ppid,stat,cmd -p <PID>` | Մեկ process-ի մանրամասներ | Անվտանգ, միայն կարդում է |
| `pgrep -a nginx` | Գտնել process-ը անունով | Անվտանգ, միայն կարդում է |
| `top` | CPU/RAM-ի live դիտարկում | `q`՝ դուրս գալու համար |
| `vmstat 1` | Ընդհանուր պատկեր` processes, memory, CPU, I/O, swap | Անվտանգ, միայն կարդում է |
| `mpstat -P ALL 1` | Յուրաքանչյուր core-ի CPU բեռը | sysstat փաթեթից |
| `iostat -x 1` | Սկավառակի `%util`-ը և `await`-ը | sysstat փաթեթից, կարող է root պահանջել |
| `iotop -o` | Որ process-ի I/O բեռն է մեծ | Root է պահանջում (`sudo`) |
| `ps -eo pid,stat,wchan:30,comm` | D-վիճակի process-ների ցուցակ | Անվտանգ, միայն կարդում է |
| `ionice -c 3 -p <PID>` | Process-ի I/O-ն` idle class | Ազդում է I/O priority-ի վրա |
| `kill -TERM <PID>` | Խնդրել process-ին նորմալ ավարտել | Նախընտրելի առաջին քայլ |
| `kill -KILL <PID>` | Kernel-ով ակնթարթորեն կանգնեցնել process-ը | Կարող է կորցնել չգրված տվյալներ |

`SIGTERM`-ը process-ին հնարավորություն է տալիս փակել ֆայլերը և ավարտել հարցումները։ `SIGKILL`-ը չի կարող բռնվել կամ անտեսվել, այդ պատճառով այն վերջին տարբերակն է։

## Փորձարկում (Lab)

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

### Lab 2. I/O-ի բեռի ախտորոշում

Միջավայր` սովորական Linux VM կամ container, որտեղ տեղադրել ես `sysstat`-ը (տալիս է `mpstat`/`iostat`) և `iotop`-ը։ Մի գործարկիր production server-ում.

```bash
sudo apt install -y sysstat iotop    # Debian/Ubuntu
dd if=/dev/zero of=/tmp/test.img oflag=direct bs=1M count=2048 &
```

`dd`-ը մեծ ֆայլ է գրում սկավառակ` ստեղծելով I/O-ի բեռ։ Այնուհետև մեկ ուրիշ տերմինալում.

```bash
top
vmstat 1
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /^D/ {print}'
iostat -x 1
```

Սպասվող արդյունքը` `top`-ում `wa`-ն բարձր է, `iostat`-ում` `%util`-ը մոտ 100% է, իսկ `ps`-ում `dd`-ի process-ը `D` վիճակում է։ Ավարտելուց հետո կանգնեցրու` `kill %1`, ապա ջնջիր ֆայլը` `rm /tmp/test.img`։

## Իրական DevOps իրավիճակ

### Ախտանիշ

Web ծառայությունը դանդաղել է, CPU-ն 100% է, իսկ load balancer-ի health check-երը սկսել են ձախողվել։

### Ախտորոշում

1. `top`-ով գտիր ամենաշատ CPU օգտագործող PID-ը։
2. `ps -o pid,ppid,stat,etime,cmd -p <PID>`-ով հաստատիր, որ դա ճիշտ ծառայությունն է։
3. Ստուգիր process tree-ը և ծառայության logs-ը. բարձր CPU-ն կարող է լինել infinite loop, traffic spike կամ անհաջող deploy-ի հետևանք։
4. Արձանագրիր ժամանակը, PID-ը, command-ը և CPU ցուցանիշը incident note-ում՝ մինչև փոփոխություն անելը։

### Լուծում

Եթե ծառայությունը systemd-ով է կառավարվում, նախընտրիր վերահսկվող restart-ը.

```bash
sudo systemctl restart <service>
sudo systemctl status <service>
```

Սա ավելի լավ է, քան պատահական worker PID-ներ kill անելը, քանի որ systemd-ն գիտի ծառայության lifecycle-ը։ Միայն եթե վերահսկվող կանգնեցումը չի աշխատում, ուսումնասիրիր կոնկրետ PID-ը և կիրառիր `SIGTERM`, ապա անհրաժեշտության դեպքում `SIGKILL`։

### Կանխարգելում

- CPU saturation-ի համար դնել alert և dashboard։
- Deploy-ից առաջ ունենալ smoke test և արագ rollback։
- Սահմանել resource limits container-ների կամ systemd unit-երի համար։
- Runbook-ում գրել՝ որ ծառայությունն ինչպես է վերագործարկվում։

### Իրավիճակ 2. Բարձր Load Average, բայց ցածր CPU` I/O-bound

#### Ախտանիշ

Տվյալների բազա և վեբ-հավելված ունեցող 8-core սերվերում` `load average: 8.45, 7.90, 6.20`։ `top`-ում` `5.2 us, 3.1 sy, 18.5 id, 72.1 wa`։ Ծառայությունը դանդաղ է աշխատում, հարցումների պատասխանները երկարում են։

#### Ախտորոշում

1. `top`-ի `wa` = 72.1%, իսկ `us + sy` = 8.3%` ուրեմն դա CPU-ի խնդիր չէ` CPU-ն սպասում է I/O-ի ավարտին։

2. `vmstat 1`-ում `b` (blocked) = 5, `bi`/`bo`-ն բարձր են (2845/1230)` I/O-ի ակտիվությունը մեծ է։
3. `mpstat -P ALL 1`-ում բոլոր 8 cores-ի `iowait`-ը 70-73% է` բեռը հավասարաչափ բաշխված է, ոչ թե մեկ core-ում։
4. `iotop -o`-ում տեսանելի է` `mysql` կարդում է 1842 M/s, `php-fpm` 567 M/s, `postgres` 436 M/s, իսկ `dd` գրում է 890 M/s` պատճառը պարզ է։

#### Լուծում

Կարճաժամկետ` իջեցնել backup-ի I/O-ն.

```bash
ionice -c 3 -p <dd PID>          # dd-ի I/O-ն` idle class
```

Երկարաժամկետ` backup-ը տեղափոխել ոչ պիկ ժամի կամ առանձին մեքենա, MySQL-ի ծանր հարցումներին ինդեքս ավելացնել, անհրաժեշտության դեպքում օգտագործել ավելի արագ սկավառակ (NVMe)։

#### Կանխարգելում

- I/O wait-ի և disk `%util`-ի համար alert դնել։
- Բոլոր մեծ jobs-ը (backup, import) գործարկել `ionice`-ով ցածր priority։
- Սկավառակի տիպը (HDD/SSD/NVMe) հաշվի առնել capacity planning-ում։

## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչ տարբերություն կա `SIGTERM` և `SIGKILL` միջև

`SIGTERM`-ը process-ին ուղարկում է նորմալ ավարտելու հարցում, և այն կարող է մաքրել resource-ները։ `SIGKILL`-ը kernel-ի պարտադիր կանգ է, process-ը չի կարող մշակել այն, ուստի օգտագործվում է միայն վերջին քայլով։

### Ի՞նչ է zombie process-ը

Ավարտված child process է, որի exit status-ը ծնողը դեռ չի կարդացել։ Այն գրեթե CPU/RAM չի օգտագործում, բայց պահում է process table-ի գրառում։ Խնդրի արմատը սովորաբար parent process-ի սխալ վարքն է։

### Ինչպե՞ս կհաստատես, որ բարձր Load Average-ը I/O-ի, ոչ թե CPU-ի պատճառով է

`top`-ի `%Cpu` տողում `wa`-ն բարձր է (օր. 70%), իսկ `us + sy`-ը` ցածր` նշանակում է` CPU-ն սպասում է I/O-ի ավարտին։ Հաստատելու համար` `vmstat 1`-ի `b` սյունակը (blocked process-ներ), `iostat -x 1`-ի `%util`-ն ու `await`-ը, և `iotop`-ը` որը ցույց է տալիս կոնկրետ `I/O`-ծանր process-ը։ Load-ն այստեղ բարձրանում է D-վիճակի process-ների հաշվին` CPU-ն պարապ է, բայց system-ը` ոչ։

## Ինքնաստուգում

1. Ինչպե՞ս կգտնես process-ի parent-ը։
2. Ի՞նչ տեղեկություն կտա `STAT` սյունակը։
3. Եթե ծառայությունը կառավարվում է systemd-ով, ո՞րն է առաջին restart հրամանը և ինչո՞ւ։
4. Ի՞նչ ապացույցներ կպահպանես բարձր CPU incident-ի ժամանակ՝ փոփոխություն անելուց առաջ։
5. Ի՞նչ է ցույց տալիս `top`-ի `wa` սյունակը, և երբ է այդ ցուցանիշը մտահոգիչ։
6. Ի՞նչ հրամանով կգտնես D-վիճակի process-ները, և ի՞նչ է դա նշանակում Load Average-ի համար։
7. Ի՞նչ կարող են ասել `iostat -x 1`-ի `%util`-ը և `await`-ը սկավառակի վիճակի մասին։

## Հաջորդ քայլեր

Շարունակիր filesystem permissions (1.4) կամ systemd (1.5) թեմայով։


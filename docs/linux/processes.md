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

## Հիմնական հրամաններ

| Հրաման | Նպատակ | Զգուշացում |
| --- | --- | --- |
| `ps aux` | Process-ների snapshot | Շատ output կարող է տալ |
| `ps -o pid,ppid,stat,cmd -p <PID>` | Մեկ process-ի մանրամասներ | Անվտանգ, միայն կարդում է |
| `pgrep -a nginx` | Գտնել process-ը անունով | Անվտանգ, միայն կարդում է |
| `top` | CPU/RAM-ի live դիտարկում | `q`՝ դուրս գալու համար |
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

## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչ տարբերություն կա `SIGTERM` և `SIGKILL` միջև

`SIGTERM`-ը process-ին ուղարկում է նորմալ ավարտելու հարցում, և այն կարող է մաքրել resource-ները։ `SIGKILL`-ը kernel-ի պարտադիր կանգ է, process-ը չի կարող մշակել այն, ուստի օգտագործվում է միայն վերջին քայլով։

### Ի՞նչ է zombie process-ը

Ավարտված child process է, որի exit status-ը ծնողը դեռ չի կարդացել։ Այն գրեթե CPU/RAM չի օգտագործում, բայց պահում է process table-ի գրառում։ Խնդրի արմատը սովորաբար parent process-ի սխալ վարքն է։

## Ինքնաստուգում

1. Ինչպե՞ս կգտնես process-ի parent-ը։
2. Ի՞նչ տեղեկություն կտա `STAT` սյունակը։
3. Եթե ծառայությունը կառավարվում է systemd-ով, ո՞րն է առաջին restart հրամանը և ինչո՞ւ։
4. Ի՞նչ ապացույցներ կպահպանես բարձր CPU incident-ի ժամանակ՝ փոփոխություն անելուց առաջ։

## Հաջորդ քայլեր

Շարունակիր filesystem permissions (1.4) կամ systemd (1.5) թեմայով։

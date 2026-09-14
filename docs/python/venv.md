# Python Virtual Environments (venv)

## Տեսություն

**Virtual environment (venv)**-ը Python-ի փոքր, մեկուսացված միջավայր է. յուրաքանչյուր venv ունի իր `site-packages` պանակը, որտեղ տեղադրված են փաթեթները, և իր executable-ների ուղին։ Այն կառուցվում է արդեն տեղադրված Python-ի վրա, որը կոչվում է **base Python**։ venv-ը մեկուսացնում է տվյալ նախագծի փաթեթները ինչպես system Python-ից, այնպես էլ մյուս venv-ներից։ Ինչպես նշված է [Python-ի պաշտոնական փաստաթղթերում](https://docs.python.org/3/library/venv.html), յուրաքանչյուր venv ստեղծվում է base Python-ի վրա և լռությամբ մեկուսացված է base environment-ի փաթեթներից։ Այդ պատճառով մեկ app-ի թարմացումը չի ազդում մյուս հավելվածների վրա։

### Ինչպե՞ս է կառուցված venv-ը

Ubuntu/Linux-ի վրա `.venv/`-ի կառուցվածքը.

```
.venv/
├── bin/
│   ├── python             ← base Python-ի symlink (կամ պատճեն)
│   ├── pip                ← գործարկվող pip
│   └── activate           ← ակտիվացման script
├── lib/
│   └── python3.12/        ← ձեր Python-ի տարբերակով
│       └── site-packages/ ← փաթեթների պանակը. այստեղ է ծավալը
│           ├── pip/
│           ├── setuptools/
│           └── ...        ← ձեր տեղադրած փաթեթները
└── pyvenv.cfg             ← ~100 bytes. base Python-ի ուղին
```

(Windows-ի վրա `bin/`-ը կոչվում է `Scripts/`, իսկ `lib/python3.12/site-packages`-ը `Lib\site-packages`։)

`pyvenv.cfg`-ը venv-ի անձնագիրն է. այն պարունակում է `home` բանալին, որը ցույց է տալիս base Python-ի ուղին, և `include-system-site-packages = false` ցույց է տալիս, որ venv-ը մեկուսացված է system փաթեթներից։

```ini
home = /usr/bin
include-system-site-packages = false
version = 3.12.3
executable = /usr/bin/python3.12
command = /usr/bin/python3 -m venv /home/app/.venv
```

### Ի՞նչ է կրկնվում, և ի՞նչը ոչ

- **Կրկնվում է.** ամեն անգամ `python3 -m venv .venv` գործարկելիս `pip`-ը (մոտ 10-15 MB) և `setuptools`-ը (մոտ 5 MB) ensurepip-ի միջոցով ընկնում են այդ venv-ի `site-packages`-ում։ Այսինքն, ամեն venv կրում է նույն `pip`-ը և `setuptools`-ը։ Երեք venv-ում այդ ~15-20 MB-ը կրկնվում է երեք անգամ (մոտ 50 MB)։

- **Չի կրկնվում.** Python interpreter-ը (բուն `python3.x` binary-ը) մեկն է ամբողջ սերվերի վրա. venv-ի `bin/python`-ը Linux-ում symlink է դեպի base Python-ը, իսկ `pyvenv.cfg`-ն ցույց է տալիս դրա ուղին (օրինակ `/usr/bin/python3.12`)։ Իրական interpreter-ը տեղադրված է մեկ անգամ, և բոլոր venv-ները կիսում են այն։

### Ինչու՞ է կրկնությունն ընդունելի

1. **Մեկուսացումն ավելի կարևոր է, քան 20 MB-ը.** Եթե `pip`-ը կամ ընդհանուր `site-packages`-ը կիսվեր բոլորի միջև, մեկ app-ի թարմացումը կարող էր կոտրել մյուսներին (dependency hell)։ venv-ն երաշխավորում է, որ յուրաքանչյուր app ունի իր փաթեթների ճշգրիտ տարբերակները։
2. **Ծավալն աննշան է** ժամանակակից սերվերի համար (GB-ներ). 15-20 MB կրկնություն յուրաքանչյուր venv-ում ռեսուրսի խնդիր չէ։
3. **Ավելի պարզ է.** ամեն venv ինքնաբավ է և disposable, հեշտ ջնջվում ու նորից ստեղծվում. Ուստի venv-ը ներառվում է `.gitignore`-ում, իսկ version control-ում չի պահվում։

### Իսկ ինչպե՞ս է սա տարբերվում Docker-ում

Docker-ում կրկնությունն ավելի մեծ է. յուրաքանչյուր image սովորաբար ներառում է իր ամբողջական Python runtime-ը (`python:3.12` base image-ը ~100 MB+ է)։ Այսինքն, ամեն կոնտեյներ կրում է իր Python-ը ի տարբերություն venv-ի, որը կիսում է հոսթի base Python-ը. Բայց Docker-ի layer caching-ը մեղմում է ծավալը. read-only image layer-ները այդ թվում Python layer-ը հոսթի վրա մեկ անգամ են պահվում և կիսվում նույն base-ով աշխատող կոնտեյների միջև։ Այսպիսով, և՛ venv-ը, և՛ Docker-ն էլ միևնույն փոխզիջումն են, փոքր կրկնվող ծավալ մեկուսացման դիմաց։

### Key concepts

- **base Python** հոսթի վրա մեկ անգամ տեղադրված Python-ը, որի վրա կառուցվում է venv-ը.
- **site-packages** venv-ի ներսի պանակը, որտեղ pip-ը տեղադրում է փաթեթները.
- **ensurepip** Python-ի մոդուլը, որը venv-ի ստեղծման ժամանակ ներսում տեղադրում է pip-ը.
- **symlink** Linux-ում venv-ի python-ը մատնանշում է base Python-ին առանց այն պատճենելու.
- **disposable** venv-ը ժամանակավոր է հեշտ ջնջվում և նորից ստեղծվում.

## Հիմնական հրամաններ

| Հրաման | Նպատակ | Զգուշացում |
| --- | --- | --- |
| `python3 -m venv .venv` | Ստեղծում է venv `.venv/` պանակում | Բունդլավորում է pip/setuptools, այստեղից մոտ 15-20 MB կրկնություն |
| `source .venv/bin/activate` | Միացնում է venv-ը (Linux) | Ազդում է միայն ընթացիկ terminal-ի վրա |
| `.venv\Scripts\activate` | Միացնում է venv-ը (Windows) | — |
| `deactivate` | Դուրս է գալիս venv-ից | — |
| `which python` | Ցույց է տալիս ակտիվ python-ի ուղին | Ակտիվ venv-ում ցույց է տալիս `.venv/bin/python` |
| `cat .venv/pyvenv.cfg` | Ցույց է տալիս base Python-ի ուղին | Անվտանգ միայն կարդում է |
| `python -m pip list` | Թվարկում է venv-ում տեղադրված փաթեթները | — |
| `python -m pip install <pkg>` | Տեղադրում է փաթեթ venv-ի մեջ | Չի ազդում այլ venv-ների վրա |
| `python -m pip freeze > requirements.txt` | Ամրագրում է ճշգրիտ տարբերակների ցուցակ | — |
| `python -m venv --system-site-packages .venv` | Թույլ է տալիս տեսնել system փաթեթները | Թուլացնում է մեկուսացումը |

## Փորձարկում (Lab)

Միջավայր սովորական Ubuntu VM կամ container. Երկու venv կստեղծենք, որ տեսնենք ինչ է կրկնվում, իսկ ինչը ոչ.

```bash
# 1) Երկու անկախ venv նույն base Python-ի վրա
python3 -m venv /tmp/app1-venv
python3 -m venv /tmp/app2-venv

# 2) Կառուցվածքն ու base Python-ի ուղին
ls -la /tmp/app1-venv/bin/
cat /tmp/app1-venv/pyvenv.cfg

# 3) Ինչ է կրկնվում. pip-ը և setuptools-ը երկու venv-ում էլ կա
ls /tmp/app1-venv/lib/python3*/site-packages/
ls /tmp/app2-venv/lib/python3*/site-packages/

# 4) Ինչը չի կրկնվում. interpreter-ը symlink է դեպի base Python
ls -l /tmp/app1-venv/bin/python
readlink -f /tmp/app1-venv/bin/python
/tmp/app1-venv/bin/python -c "import sys; print(sys.prefix)"

# 5) Ծավալը
du -sh /tmp/app1-venv /tmp/app2-venv
```

Սպասվող արդյունք. երկու `site-packages`-ում նույն `pip`/`setuptools`-ն է, իսկ `bin/python`-ը երկուսում էլ մատնանշում է նույն base Python-ը. interpreter-ը չի կրկնվում. venv-ի ներսում `lib/python3*/site-packages/`-ը պետք է ստուգես ձեր Python-ի տարբերակով (`python3 --version`)-ով.

!!! note "Best Practices"
    - venv-ը համարիր disposable. ջնջել ու նորից ստեղծելն ավելի հուսալի է, քան վերանորոգել.
    - venv-ը չի version control-վում. `.venv/`-ը ավելացրու `.gitignore`-ում.
    - Տարբերակներն ամրագրիր. `python -m pip freeze > requirements.txt` (կամ `uv.lock`) վերարտադրելի build-ի համար.
    - Ծառայությունները գործարկիր venv-ի ամբողջական ուղիով, օրինակ `/srv/app/.venv/bin/python`:

## Իրական DevOps իրավիճակ

### Ախտանիշ

Production սերվերում երեք Python հավելված աշխատում է առանց venv-ի, բոլորը փաթեթներ են տեղադրում system `site-packages`-ում. Թիմերից մեկը մեկ հավելվածի համար թարմացնում է ընդհանուր փաթեթը (օրինակ `requests`), և հաջորդ օրը մյուս հավելվածը սկսում է ընկնել `ImportError`-ով, որովհետև փաթեթի API-ն փոխվել է, իսկ այդ հավելվածն ակնկալում էր հին տարբերակը (dependency hell).

### Ախտորոշում

1. Պարզիր թե ինչ տարբերակ է ակտիվ.
   ```bash
   python3 -m pip show requests
   python3 -m pip freeze | grep -i requests
   ```
2. Համոզվիր, որ նույն system `site-packages`-ից օգտվում են մի քանի app-ներ.
3. Եզրակացրու. մեկի թարմացումն ազդում է բոլորի վրա.

### Լուծում

Կարճաժամկետ rollback վերադարձրու նախկին տարբերակը.

```bash
python3 -m pip install requests==<նախորդ_տարբերակ>
```

Երկարաժամկետ յուրաքանչյուր app-ին տուր իր venv-ը.

```bash
python3 -m venv /srv/app1/.venv
/srv/app1/.venv/bin/python -m pip install -r /srv/app1/requirements.txt
# նույնը app2-ի և app3-ի համար
```

Այսուհետ ամեն app-ի թարմացումն ազդում է միայն իր venv-ի վրա. systemd unit-ի `ExecStart`-ում գրիր venv-ի `bin/python`-ը, օրինակ `/srv/app1/.venv/bin/python /srv/app1/main.py`.

### Կանխարգելում

- Յուրաքանչյուր app սեփական venv ամրագրված `requirements.txt`-ով (կամ `uv.lock`).
- CI-ում փորձարկիր venv-ով, իսկ deploy-ի ժամանակ վերստեղծիր venv-ը նույն ցուցակից (reproducible build).
- Ծառայությունները գործարկիր venv-ի ամբողջական ուղիով, ոչ թե PATH-ից.
- Ավելի խոր մեկուսացման համար փաթեթավորիր app-ը Docker image-ով (տես [Docker և Կոնտեյներներ](../docker-kubernetes/index.md)).

## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչ կա venv-ի ներսում, և ինչու՞ է pip-ը կրկնվում ամեն venv-ում (mid-level)

venv-ի ներսում կան `bin/` (Windows-ում `Scripts/`), `lib/pythonX.Y/site-packages/` (փաթեթները) և `pyvenv.cfg` (base Python-ի ուղին). `pip`-ն ու `setuptools`-ը կրկնվում են, որովհետև ensurepip-ը դրանք ավելացնում է յուրաքանչյուր `site-packages`-ում, որպեսզի ամեն venv ինքնավար լինի. մեկուսացումն այստեղ ավելի կարևոր է, քան 15-20 MB կրկնվող ծավալը.

### Ինչու՞ interpreter-ը չի կրկնվում, իսկ pip-ը կրկնվում է, և ինչո՞ւ է դա ընդունելի (senior-level)

Interpreter-ը մեկն է ամբողջ սերվերի վրա. venv-ը չի պատճենում Python-ը. Linux-ում `bin/python`-ը symlink է դեպի base Python-ը, իսկ `pyvenv.cfg`-ի `home`-ը ցույց է տալիս դրա ուղին. Փաթեթները յուրաքանչյուր venv-ում տարբեր են, որովհետև դրանք նախագծից նախագիծ պետք է տարբերվեն. venv-ը կրկնում է փաթեթի շերտը, բայց կիսում է interpreter-ի շերտը հոսթի մակարդակով. Docker-ում պատկերն հակառակն է. ամեն image-ը կրում է իր Python runtime-ը (base image ~100 MB+), այսինքն կրկնվում է ամբողջ interpreter-ը ևս. Բայց layer caching-ը read-only layer-ները մեկ անգամ է պահում հոսթի վրա, կիսելով նույն base-ով աշխատող կոնտեյների միջև.

## Ինքնաստուգում

1. Որ ֆայլերն են կրկնվում ամեն venv-ում, և որո՞նք ոչ.
2. Ի՞նչ է ցույց տալիս `pyvenv.cfg`-ի `home` բանալին, և ինչու է venv-ը կիսում base Python-ը.
3. Ինչու՞ չի կարելի օպտիմիզացնել ջնջելով կամ կիսելով venv-ների `pip`/`site-packages`-ը.
4. Ինչու՞ venv-ը չի ներառվում Git-ում, և ինչպե՞ս է այն վերստեղծվում նոր մեքենայում.
5. Ինչպե՞ս է venv-ի կրկնությունը համեմատվում Docker-ի, և ինչպե՞ս է layer caching-ը մեղմում այն.

## Հաջորդ քայլեր

Հիմա, երբ հասկանում ես, որ venv-ում կրկնությունը գիտակցված փոխզիջում է, խորացրու թեման [Docker և Կոնտեյներներ](../docker-kubernetes/index.md), որտեղ նույն մեկուսացման սկզբունքը լուծվում է image layer-ների միջոցով, կամ [Linux Processes](../linux/processes.md) որպեսզի տեսնես, թե ինչպես են process-ները դիտարկվում հոսթի վրա. Package management-ի հաջորդ քայլը `pip-tools`/`uv` տարբերակների ամրագրման և վերարտադրելի build-ի գործիքներն են.

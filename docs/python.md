# Python

Python-ը DevOps-ի ամենահաճախ օգտագործվող լեզուներից է. դրանով գրվում են ավտոմատացման սկրիպտեր, CI/CD գործիքներ, Ansible-ի մոդուլներ և ներքին API-ներ։ Այս էջում սկսում ենք միջավայրի մեկուսացումից, որը Python-ում ամենահաճախ խնդիրներն առաջացնող թեման է։

## Էջի բաժիններ

- [Virtual Environments (venv)](#venv) — ինչպես է կառուցված venv-ը, ինչն է կրկնվում ամեն միջավայրում, dependency hell-ի իրական իրավիճակ և կանխարգելում։
- [Python մեթոդներ և type hints](#python-methods) — `@staticmethod`/`@classmethod`/`@property`, bound method-ի մեխանիզմը, `_` կոնվենցիան, type hints-ի runtime չստուգումը։
- [SQL placeholders և injection-ից պաշտպանություն](#sql-placeholders) — DB-API placeholder-ների ոճերը, prepared statement-ը, `("Anna")` vs `("Anna",)` սխալը, shell injection և XSS։
- pip, requirements և package management — շուտով (`uv`/`pip-tools` տարբերակների ամրագրմամբ)։

## Virtual Environments (venv) {#venv}

### Տեսություն

**Virtual environment (venv)**-ը Python-ի փոքր, մեկուսացված միջավայր է. յուրաքանչյուր venv ունի իր `site-packages` պանակը, որտեղ տեղադրված են փաթեթները, և իր executable-ների ուղին։ Այն կառուցվում է արդեն տեղադրված Python-ի վրա, որը կոչվում է **base Python**։ venv-ը մեկուսացնում է տվյալ նախագծի փաթեթները ինչպես system Python-ից, այնպես էլ մյուս venv-ներից։ Ինչպես նշված է [Python-ի պաշտոնական փաստաթղթերում](https://docs.python.org/3/library/venv.html), յուրաքանչյուր venv ստեղծվում է base Python-ի վրա և լռությամբ մեկուսացված է base environment-ի փաթեթներից։ Այդ պատճառով մեկ app-ի թարմացումը չի ազդում մյուս հավելվածների վրա։

#### Ինչպե՞ս է կառուցված venv-ը

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

#### Ի՞նչ է կրկնվում, և ի՞նչը ոչ

- **Կրկնվում է.** ամեն անգամ `python3 -m venv .venv` գործարկելիս `pip`-ը (մոտ 10-15 MB) և `setuptools`-ը (մոտ 5 MB) ensurepip-ի միջոցով ընկնում են այդ venv-ի `site-packages`-ում։ Այսինքն, ամեն venv կրում է նույն `pip`-ը և `setuptools`-ը։ Երեք venv-ում այդ ~15-20 MB-ը կրկնվում է երեք անգամ (մոտ 50 MB)։

- **Չի կրկնվում.** Python interpreter-ը (բուն `python3.x` binary-ը) մեկն է ամբողջ սերվերի վրա. venv-ի `bin/python`-ը Linux-ում symlink է դեպի base Python-ը, իսկ `pyvenv.cfg`-ն ցույց է տալիս դրա ուղին (օրինակ `/usr/bin/python3.12`)։ Իրական interpreter-ը տեղադրված է մեկ անգամ, և բոլոր venv-ները կիսում են այն։

#### Ինչու՞ է կրկնությունն ընդունելի

1. **Մեկուսացումն ավելի կարևոր է, քան 20 MB-ը.** Եթե `pip`-ը կամ ընդհանուր `site-packages`-ը կիսվեր բոլորի միջև, մեկ app-ի թարմացումը կարող էր կոտրել մյուսներին (dependency hell)։ venv-ն երաշխավորում է, որ յուրաքանչյուր app ունի իր փաթեթների ճշգրիտ տարբերակները։
2. **Ծավալն աննշան է** ժամանակակից սերվերի համար (GB-ներ). 15-20 MB կրկնություն յուրաքանչյուր venv-ում ռեսուրսի խնդիր չէ։
3. **Ավելի պարզ է.** ամեն venv ինքնաբավ է և disposable, հեշտ ջնջվում ու նորից ստեղծվում. Ուստի venv-ը ներառվում է `.gitignore`-ում, իսկ version control-ում չի պահվում։

#### Իսկ ինչպե՞ս է սա տարբերվում Docker-ում

Docker-ում կրկնությունն ավելի մեծ է. յուրաքանչյուր image սովորաբար ներառում է իր ամբողջական Python runtime-ը (`python:3.12` base image-ը ~100 MB+ է)։ Այսինքն, ամեն կոնտեյներ կրում է իր Python-ը ի տարբերություն venv-ի, որը կիսում է հոսթի base Python-ը. Բայց Docker-ի layer caching-ը մեղմում է ծավալը. read-only image layer-ները այդ թվում Python layer-ը հոսթի վրա մեկ անգամ են պահվում և կիսվում նույն base-ով աշխատող կոնտեյների միջև։ Այսպիսով, և՛ venv-ը, և՛ Docker-ն էլ միևնույն փոխզիջումն են, փոքր կրկնվող ծավալ մեկուսացման դիմաց։

#### Key concepts

- **base Python** հոսթի վրա մեկ անգամ տեղադրված Python-ը, որի վրա կառուցվում է venv-ը.
- **site-packages** venv-ի ներսի պանակը, որտեղ pip-ը տեղադրում է փաթեթները.
- **ensurepip** Python-ի մոդուլը, որը venv-ի ստեղծման ժամանակ ներսում տեղադրում է pip-ը.
- **symlink** Linux-ում venv-ի python-ը մատնանշում է base Python-ին առանց այն պատճենելու.
- **disposable** venv-ը ժամանակավոր է հեշտ ջնջվում և նորից ստեղծվում.

### Հիմնական հրամաններ

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

### Փորձարկում (Lab)

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

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Production սերվերում երեք Python հավելված աշխատում է առանց venv-ի, բոլորը փաթեթներ են տեղադրում system `site-packages`-ում. Թիմերից մեկը մեկ հավելվածի համար թարմացնում է ընդհանուր փաթեթը (օրինակ `requests`), և հաջորդ օրը մյուս հավելվածը սկսում է ընկնել `ImportError`-ով, որովհետև փաթեթի API-ն փոխվել է, իսկ այդ հավելվածն ակնկալում էր հին տարբերակը (dependency hell).

#### Ախտորոշում

1. Պարզիր թե ինչ տարբերակ է ակտիվ.
   ```bash
   python3 -m pip show requests
   python3 -m pip freeze | grep -i requests
   ```
2. Համոզվիր, որ նույն system `site-packages`-ից օգտվում են մի քանի app-ներ.
3. Եզրակացրու. մեկի թարմացումն ազդում է բոլորի վրա.

#### Լուծում

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

#### Կանխարգելում

- Յուրաքանչյուր app սեփական venv ամրագրված `requirements.txt`-ով (կամ `uv.lock`).
- CI-ում փորձարկիր venv-ով, իսկ deploy-ի ժամանակ վերստեղծիր venv-ը նույն ցուցակից (reproducible build).
- Ծառայությունները գործարկիր venv-ի ամբողջական ուղիով, ոչ թե PATH-ից.
- Ավելի խոր մեկուսացման համար փաթեթավորիր app-ը Docker image-ով (տես [Docker և Կոնտեյներներ](containers.md)).

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ կա venv-ի ներսում, և ինչու՞ է pip-ը կրկնվում ամեն venv-ում (mid-level)

venv-ի ներսում կան `bin/` (Windows-ում `Scripts/`), `lib/pythonX.Y/site-packages/` (փաթեթները) և `pyvenv.cfg` (base Python-ի ուղին). `pip`-ն ու `setuptools`-ը կրկնվում են, որովհետև ensurepip-ը դրանք ավելացնում է յուրաքանչյուր `site-packages`-ում, որպեսզի ամեն venv ինքնավար լինի. մեկուսացումն այստեղ ավելի կարևոր է, քան 15-20 MB կրկնվող ծավալը.

#### Ինչու՞ interpreter-ը չի կրկնվում, իսկ pip-ը կրկնվում է, և ինչո՞ւ է դա ընդունելի (senior-level)

Interpreter-ը մեկն է ամբողջ սերվերի վրա. venv-ը չի պատճենում Python-ը. Linux-ում `bin/python`-ը symlink է դեպի base Python-ը, իսկ `pyvenv.cfg`-ի `home`-ը ցույց է տալիս դրա ուղին. Փաթեթները յուրաքանչյուր venv-ում տարբեր են, որովհետև դրանք նախագծից նախագիծ պետք է տարբերվեն. venv-ը կրկնում է փաթեթի շերտը, բայց կիսում է interpreter-ի շերտը հոսթի մակարդակով. Docker-ում պատկերն հակառակն է. ամեն image-ը կրում է իր Python runtime-ը (base image ~100 MB+), այսինքն կրկնվում է ամբողջ interpreter-ը ևս. Բայց layer caching-ը read-only layer-ները մեկ անգամ է պահում հոսթի վրա, կիսելով նույն base-ով աշխատող կոնտեյների միջև.

### Ինքնաստուգում

1. Որ ֆայլերն են կրկնվում ամեն venv-ում, և որո՞նք ոչ.
2. Ի՞նչ է ցույց տալիս `pyvenv.cfg`-ի `home` բանալին, և ինչու է venv-ը կիսում base Python-ը.
3. Ինչու՞ չի կարելի օպտիմիզացնել ջնջելով կամ կիսելով venv-ների `pip`/`site-packages`-ը.
4. Ինչու՞ venv-ը չի ներառվում Git-ում, և ինչպե՞ս է այն վերստեղծվում նոր մեքենայում.
5. Ինչպե՞ս է venv-ի կրկնությունը համեմատվում Docker-ի, և ինչպե՞ս է layer caching-ը մեղմում այն.

### Հաջորդ քայլեր

Հիմա, երբ հասկանում ես, որ venv-ում կրկնությունը գիտակցված փոխզիջում է, խորացրու թեման [Docker և Կոնտեյներներ](containers.md), որտեղ նույն մեկուսացման սկզբունքը լուծվում է image layer-ների միջոցով, կամ [Linux Processes](linux.md#processes) որպեսզի տեսնես, թե ինչպես են process-ները դիտարկվում հոսթի վրա. Package management-ի հաջորդ քայլը `pip-tools`/`uv` տարբերակների ամրագրման և վերարտադրելի build-ի գործիքներն են։

Անցիր [Python մեթոդներ և type hints](#python-methods), որտեղ «պահիր սահմանը» սկզբունքը շարունակվում է մեթոդների և type hints-ի մակարդակում։

## Python մեթոդներ և type hints {#python-methods}

### Տեսություն

Class-ի ներսում գրված ֆունկցիաները **մեթոդներ** են։ Կարևորն այն է, թե ամեն մեթոդ ինչ է ստանում որպես առաջին արգումենտ և ինչ է դա նշանակում runtime-ում։

#### Ինչ է կատարվում մեթոդի կանչի ժամանակ

Python-ում ֆունկցիան **descriptor** է։ Երբ գրում ես `obj.method()`, Python-ը class-ի namespace-ից վերցնում է ֆունկցիան և ստեղծում է **bound method**, որում `self`-ն արդեն լրացված է։

```python
class User:
    def __init__(self, name):
        self.name = name

    def greet(self):
        return f"hello {self.name}"

u = User("Anna")
u.greet()        # Python-ը u-ն փոխանցում է որպես self
User.greet(u)    # համարժեք է. self-ը փոխանցում ես ձեռքով
```

Այսինքն `self`-ը կախարդանք չէ. դա ուղղակի առաջին արգումենտի ընդունված անունն է, որը instance-ի դեպքում Python-ը լրացնում է ինքնաշխատ։ Սա տարբերում է .NET/C#-ից, որտեղ `this`-ը keyword է, ոչ պարամետր։

#### Երեք տեսակի մեթոդ

| Տեսակ | Առաջին արգումենտ | Կանչը | Տիպիկ օգտագործում |
| --- | --- | --- | --- |
| Instance method | `self` (instance) | `obj.method()` | instance-ի state-ով աշխատող տրամաբանություն |
| `@classmethod` | `cls` (class) | `Class.method()` կամ `obj.method()` | այլընտրանքային constructor, factory |
| `@staticmethod` | չկա | `Class.method()` | class-ի namespace-ում ապրող անկախ ֆունկցիա |

`@classmethod`-ը օգտակար է, երբ մեթոդը ինքն է ստեղծում instance, բայց վերադարձրած դասը պետք է ճիշտ լինի նաև ժառանգորդի դեպքում։

```python
import json


class Config:
    def __init__(self, data):
        self.data = data

    @classmethod
    def from_file(cls, path):      # ժառանգորդի դեպքում cls-ը կլինի ժառանգորդը
        with open(path) as f:
            return cls(json.load(f))
```

`@staticmethod`-ը class-ի մեջ դրած սովորական ֆունկցիա է. չի ստանում ոչ `self`, ոչ `cls`։

```python
class PasswordPolicy:
    MIN_LEN = 12

    @staticmethod
    def is_strong(pw: str) -> bool:
        return len(pw) >= PasswordPolicy.MIN_LEN
```

Կա նաև չորրորդ, շատ գործածական դեկորատորը.

```python
class Repo:
    def __init__(self, path):
        self.path = path

    @property
    def label(self) -> str:        # կանչվում է առանց փակագծի. obj.label
        return f"Repo({self.path})"
```

#### .NET-ի համեմատություն

C#-ում `static`-ը մեթոդին հանում է instance-ից կապը, բայց այն դեռ դասի անդամ է, և `this` չկա։ Python-ում դրան ամենամոտը `@staticmethod`-ն է. երկուսում էլ կանչը `Class.Method()` ձևով է։

Երկու տարբերություն կարևոր է հիշել.

1. Python-ում ամեն ինչ runtime-ում է։ `@staticmethod`-ը ուղղակի wrapper օբյեկտ է, ոչ compile-ի ժամանակի որոշում։ Ամբողջությամբ static դասի փոխարեն Python-ում սովորաբար գրում են **մոդուլի մակարդակի ֆունկցիաներ**, որովհետև մոդուլն ինքն է namespace-ը։
2. `@staticmethod` պարտադիր չէ այն մեթոդի համար, որը `self` չի օգտագործում։ Python-ը չի բողոքում, եթե ուղղակի թողնես `self` չօգտագործված։ Այդ պատճառով code review-ում հարց է դառնում ոչ թե «ինչու՞ static», այլ «ինչու՞ այս ֆունկցիան դասի մեջ է»։

#### Գաղտնիությունը `_` կոնվենցիայով

Python-ը չունի `private` keyword։ Կա կոնվենցիա.

- `_name` — ներքին օգտագործման համար է. դրսից չի դիպչում։ Runtime-ը չի արգելում, սակայն code review-ում դա սահմանի խախտում է։
- `__name` (երկու underscore սկզբում, ամենաշատը մեկը վերջում) — **name mangling**։ Դասի ներսում `__total` անունը դառնում է `_ClassName__total`։ Սա պատահական բախումից պաշտպանելու մեխանիզմ է, ոչ մուտքի արգելք. դրսից այն դեռ հասանելի է ամբողջական անունով։
- `__name__` (երկու կողմերում) — Python-ի հատուկ անուններ (`__init__`, `__str__`, `__enter__`)։

C#-ի `private`-ի ուղիղ համարժեքը չկա։ Այդ պատճառով Python-ում API-ի սահմանը պահվում է կարգապահությամբ և static check-երով, ոչ լեզվի կողմից։

#### Type hints

Type hints-ը signature-ում գրած տիպի նշումներ են։ Runtime-ը դրանք չի պարտադրում, բայց արժեքը մեծ է։

```python
from collections.abc import Iterator, Sequence


def get_active_users(limit: int = 100) -> list[dict[str, str]]:
    ...
```

- `int`, `str`, `bool`, `float` — պարզ տիպեր։
- `list[dict[str, str]]` — ժամանակակից generics (PEP 585)։ Հին կոդում դեռ կհանդիպես `List[Dict[str, str]]` `typing`-ից։
- `str | None` (PEP 604) կամ հին `Optional[str]`։
- `Iterator[str]`, `Sequence[int]` — աբստրակտ կոլեկցիաներ։ Արգումենտի համար նախընտրելի են կոնկրետ `list`-ից, որ ֆունկցիան ավելի քիչ սահմանափակող լինի։
- `Any` — «ինձ համար նշանակություն չունի»։ Ամեն դեպքում ազդանշան է, որ տիպը չի մտածվել։

Կարևոր է հասկանալ. **Python-ը runtime-ում չի ստուգում type hints-ը**։ Նշումները պահվում են որպես `__annotations__` dictionary, և վերջ։ Ստուգումը կատարում են.

- static checker-ները (`mypy`, `pyright`, `ruff`-ի որոշ կանոններ) — CI-ում. սա է իրական արժեքը։
- IDE-ն — autocomplete, inline սխալ, refactor։
- runtime-վալիդացնող գրադարանները (`pydantic`, FastAPI) — դրանք hints-ը կարդում են որպես սխեմա և ստուգում իրական արժեքները։

Hints-ը նաև փաստաթուղթ է. `-> list[dict[str, str]]`-ը ուղղակի ասում է, որ մեթոդը վերադարձնում է dict-ների ցուցակ, այսինքն SQL-ի տողերի ցուցակ, ոչ ORM-ի օբյեկտներ։

#### Իրար կողքի string literals-ի միացումը

Python-ում կողք-կողքի գրած string literals-ը **միանում են compile-ի ժամանակ**։

```python
query = (
    "SELECT id, name "
    "FROM users "
    "WHERE active = 1"
)
# մեկ string է. "SELECT id, name FROM users WHERE active = 1"
```

Սա հարմար է երկար SQL-ը կարդալի մասերի բաժանելու համար, առանց `+`-ի և առանց runtime-ում միացման ծախսի։

Բայց նույն տեսքը ունի մի նենգ սխալ. `("Anna")`-ն **string** է, ոչ tuple։ Tuple-ի համար պարտադիր է ստորակետ. `("Anna",)`։ SQL-ի parameter-ների հետ այդ տարբերությունը առաջացնում է իրական bug, որը քննարկվում է [SQL placeholders](#sql-placeholders) բաժնում։

#### Key concepts

- **descriptor** մեխանիզմը, որով class-ի ատրիբուտը կառավարում է իր մուտքը (`__get__`)։
- **bound method** instance-ին արդեն կապված մեթոդ, որում `self`-ը լրացված է։
- **`cls`** class-ի հղումը `@classmethod`-ի մեջ, ժառանգության դեպքում՝ ճիշտ դասը։
- **name mangling** `__name`-ի վերածումը `_ClassName__name`-ի։
- **type hint** signature-ում գրած տիպի նշումը, որ runtime-ում չի ստուգվում։
- **dunder** կրկնակի underscore-ով հատուկ անունները (`__init__`, `__str__`)։

### Հիմնական հրամաններ

| Հրաման | Նպատակ | Սպասվող արդյունք | Անվտանգության ռիսկ |
| --- | --- | --- | --- |
| `python -m mypy app/` | Ստուգում է type hints-ը static | Սխալների ցուցակ կամ `Success: no issues found` | Անվտանգ, միայն ընթերցում |
| `python -m mypy app/ --strict` | Նույնը, խիստ ռեժիմով | Ավելի շատ սխալներ, ավելի պահանջկոտ | Անվտանգ |
| `python -m pyright app/` | Այլընտրանքային static checker | Նույն տիպի արդյունք | Անվտանգ |
| `python -c "import inspect, app; print(inspect.signature(app.Repo.count))"` | Ցույց է տալիս մեթոդի signature-ը runtime-ում | `(self, limit: int = 10) -> list[...]` | Անվտանգ |
| `obj.__dict__` | Ցույց է տալիս instance-ի ատրիբուտները | dict | Անվտանգ է, բայց log-ում կարող է արտահոսել գաղտնիքներ |
| `Name._Class__private` | Հասնում է mangled անդամին | Արժեք | Սահմանի խախտում. վտանգում է կայունությունը, ոչ անվտանգությունը |
| `python -X dev -m pytest` | Գործարկում է թեստերը dev-ռեժիմով | Զգուշացումներ + թեստերի արդյունք | Անվտանգ |

### Փորձարկում (Lab)

Միջավայր սովորական Ubuntu VM. լրացուցիչ ծրագրեր պետք չեն, բացի `mypy`-ից, որը տեղադրում ենք lab-ի ընթացքում։

```bash
cat > /tmp/methods_demo.py <<'PY'
class Repo:
    TABLE = "users"

    def __init__(self, db_path: str) -> None:
        self.db_path = db_path

    def count(self, limit: int = 10) -> list[dict[str, str]]:
        return [{"id": str(i)} for i in range(1, limit + 1)]

    def add_tag(self, tag: str) -> str:
        return f"[{tag}]"

    @classmethod
    def from_env(cls, env_value: str):
        return cls(env_value or "/tmp/app.db")

    @staticmethod
    def is_valid_id(value: str) -> bool:
        return value.isdigit()

    @property
    def label(self) -> str:
        return f"Repo({self.db_path})"


r = Repo.from_env("/tmp/a.db")

print(r.label)                 # property. կանչվում է առանց փակագծի
print(r.count(3))              # instance method
print(Repo.is_valid_id("42"))  # static method
print(r.add_tag(42))           # hint-ը չի ստուգվում. կաշխատի

try:
    print(r.count("3"))        # կընկնի միայն օգտագործման պահին
except TypeError as e:
    print("runtime:", e)
PY

python3 /tmp/methods_demo.py
python3 -m pip install --user mypy
python3 -m mypy /tmp/methods_demo.py
```

Սպասվող արդյունք.

1. `python3 /tmp/methods_demo.py`-ը կտպի `Repo(/tmp/a.db)`, երեք dict, `True`, `[42]` և `runtime: 'str' object cannot be interpreted as an integer`։ Ուշադրություն դարձրու `r.add_tag(42)`-ին. `42`-ը `str` չէ, բայց ոչ ոք չբողոքեց, որովհետև տիպը runtime-ում չի ստուգվում։
2. `mypy`-ը կտա երկու սխալ. `Argument 1 to "add_tag" of "Repo" has incompatible type "int"; expected "str"` և նույնը `count`-ի համար։ Սա հենց ցույց է տալիս, որ type hints-ը **CI-ի ժամանակի պայմանագիր** է, ոչ runtime-ի պաշտպանություն։

!!! tip "Best Practices"
    - `@staticmethod` օգտագործիր միայն այն դեպքում, երբ ֆունկցիան իսկապես տրամաբանորեն կապված է դասի հետ. հակառակ դեպքում մոդուլի ֆունկցիան ավելի հասարակ է։
    - `@classmethod` նախընտրիր այլընտրանքային constructor-ների համար. այդպես ժառանգորդը ճիշտ դասը կվերադարձնի։
    - Ամեն public ֆունկցիայի վրա գրիր type hints և CI-ում գործարկիր `mypy`/`pyright`։
    - Սահմանի վրա (ENV, JSON, HTTP) արժեքը անմիջապես փոխարիր ճիշտ տիպին, որ hint-ը իրականանա նաև runtime-ում։
    - `_`-ով սկսվող անդամը դրսից մի կանչիր. դա ներքին պայմանագիր է։

### Իրական DevOps իրավիճակ

#### Ախտանիշ

CI-ում `pytest`-ը կանաչ է, իսկ production-ում ամեն օր ընկնում է `TypeError: 'str' object cannot be interpreted as an integer` վիճակագրական մոդուլում։ Թիմը համոզված էր, որ բավական է `limit: int` գրել, և Python-ն ինքը կստուգի։ Սխալը ի հայտ է գալիս միայն այն ժամանակ, երբ արժեքը եկել է ENV-ից որպես string։

#### Ախտորոշում

1. Ստուգիր, թե արդյոք type hints-ը որևէ static checker-ով անցնում է.
   ```bash
   python3 -m mypy app/ || echo "mypy կամ կարգավորված չէ, կամ սխալներ կան"
   ```
2. Ստուգիր, թե որտեղից է գալիս արժեքը. `os.environ`, JSON-ը, HTTP query-ն միշտ string են տալիս։
3. Ավելացրու traceback-ի իրական աղբյուրը գտնելու համար մուտքի log.
   ```bash
   grep -rn "os.environ" app/ | grep -i limit
   ```
4. Եզրակացրու. hint-ը միայն փաստաթուղթ էր, ոչ ոք runtime-ում չէր ստուգում, և CI-ում checker չկար։

#### Լուծում

Նորմալացրու արժեքը սահմանի վրա և static ստուգումն ընդգրկիր CI-ում։

```python
def parse_limit(raw: str) -> int:
    value = int(raw)          # ակնհայտ փոխարկում սահմանի վրա
    if value <= 0:
        raise ValueError("limit պետք է լինի դրական")
    return value


limit = parse_limit(os.environ.get("REPORT_LIMIT", "100"))
```

```bash
python3 -m mypy app/ --strict
```

#### Կանխարգելում

- CI-ում ավելացրու `mypy`/`pyright` քայլ և դարձրու պարտադիր. կամային ստուգումը միշտ մոռացվում է։
- Արտաքին մուտքը (ENV, JSON, HTTP) փոխարկիր տիպերին անմիջապես սահմանի վրա, ոչ օգտագործման տեղում։
- Բարդ տվյալի համար օգտագործիր `pydantic` սխեմա, որ վալիդացիան լինի և՛ static, և՛ runtime։
- Code review-ում պահանջիր type hints public ֆունկցիաների և մեթոդների համար։
- Չօգտագործված `self`-ով մեթոդը կամ դարձրու `@staticmethod`, կամ տեղափոխիր մոդուլի մակարդակ։

### Հարցազրույցի հարցեր և պատասխաններ

#### Ի՞նչ տարբերություն կա `@staticmethod`-ի, `@classmethod`-ի և սովորական մեթոդի միջև (mid-level)

Սովորական մեթոդը ստանում է `self` (instance) և աշխատում է instance-ի state-ով։ `@classmethod`-ը ստանում է `cls` (դաս) և սովորաբար օգտագործվում է այլընտրանքային constructor-ների համար. ժառանգության դեպքում `cls`-ը ճիշտ դասն է, այստեղից էլ օգուտը։ `@staticmethod`-ը ոչ `self`, ոչ `cls` չի ստանում. դա class-ի namespace-ում դրված սովորական ֆունկցիա է։ Ավելացրու, որ `@property`-ն չորրորդ դեկորատորն է՝ մեթոդը attribute-ի նման կարդալու համար։

#### Ինչպե՞ս են Python-ի մեթոդները կապվում instance-ին, և ինչու՞ type hints-ը runtime-ում չի պաշտպանում (senior-level)

Python-ում ֆունկցիան descriptor է. `obj.method()` արտահայտության ժամանակ class-ի namespace-ից վերցվում է ֆունկցիան և ստեղծվում է bound method, որում `self`-ը արդեն լրացված է. նույն մեխանիզմն է աշխատում `@property`-ի դեպքում։ Type hints-ը ուղղակի `__annotations__` dictionary է և interpreter-ի կողմից չի ստուգվում, որովհետև Python-ը դինամիկ լեզու է. դրանք պայմանագիր են static checker-ների (`mypy`, `pyright`), IDE-ների և runtime-վալիդացնող գրադարանների (`pydantic`) հետ։ Սա է պատճառը, որ hints-ը պետք է լինեն CI-ում պարտադիր, այլապես դրանք ուղղակի մեկնաբանություն են։ Հիշիր նաև, որ տիպերը պարտադրելի են դառնում միայն այն ժամանակ, երբ արժեքը շոշափում է գործողությունը. այդ պատճառով ամենավտանգավոր մուտքը ENV/JSON/HTTP-ն է, որը պետք է նորմալացնել սահմանի վրա։

### Ինքնաստուգում

1. Ի՞նչ է ստանում `@staticmethod`-ը որպես առաջին արգումենտ, և երբ այն ավելի լավ է փոխարինել մոդուլի ֆունկցիայով։
2. Ինչու՞ է `@classmethod`-ը նախընտրելի այլընտրանքային constructor-ի համար, և ինչ է փոխվում ժառանգության դեպքում։
3. Ի՞նչ տեղի կունենա `r.add_tag(42)` կանչի ժամանակ, և ի՞նչ կասի `mypy`-ը։
4. Ինչու՞ է `("Anna")`-ն string, ոչ tuple, և ինչպիսի՞ bug-ի կհանգեցնի SQL parameter-ի հետ։
5. Ի՞նչ է սահմանում `_name`-ը, և ինչ է անում name mangling-ը։
6. Ինչու՞ type hints-ը բավարար չեն production-ում, և ի՞նչ անվտանգության շերտ է պակասում։

### Հաջորդ քայլեր

Հիմա, երբ մեթոդների սահմանները պարզ են, անցիր [SQL placeholders և injection-ից պաշտպանություն](#sql-placeholders), որտեղ նույն «ստուգիր սահմանի վրա» սկզբունքը ցույց է տալիս, թե ինչպես է մուտքը դադարում հրաման լինել։ Ապա վերադարձիր [Virtual Environments (venv)](#venv) բաժինը, եթե պետք է տարբեր DB driver-ներով նախագծեր մեկուսացնել, կամ [Least Privilege](security.md#least-privilege), որ սահմանափակես վնասի շառավիղը, երբ պաշտպանությունը շրջանցվել է։

## SQL placeholders և injection-ից պաշտպանություն {#sql-placeholders}

### Տեսություն

Օգտատիրոջ մուտքը SQL տեքստի մեջ ուղղակի միացնելը ամենահին և ամենավտանգավոր խոցելիություններից է։ Խնդիրը լուծվում է մեկ սկզբունքով. **մուտքը երբեք SQL տեքստ չի դառնում, այն միշտ parameter է**։

#### Placeholder-ների ոճերը (DB-API)

Python-ի տվյալների բազայի driver-ները հետևում են DB-API 2.0 (PEP 249) ստանդարտին, որը սահմանում է **paramstyle** հասկացությունը։

| Paramstyle | Placeholder | Որտեղ | Փոխանցվող արժեք | Օրինակ |
| --- | --- | --- | --- | --- |
| `qmark` | `?` | `sqlite3`, `pyodbc` | tuple կամ list | `cur.execute("... WHERE name = ?", (name,))` |
| `format` | `%s` | `psycopg2`, `mysql-connector-python` | tuple կամ list | `cur.execute("... WHERE name = %s", (name,))` |
| `named` | `%(name)s` | `psycopg2`, `mysql-connector-python` | dict | `cur.execute("... WHERE name = %(n)s", {"n": name})` |
| `numeric` | `:1` | `oracledb` | tuple կամ list | `cur.execute("... WHERE name = :1", (name,))` |

`sqlite3`-ը նաև ընդունում է իր **named** տեսքը՝ `:name` կամ `$name`, այդ դեպքում փոխանցվում է dict։

Առանձին դեպք է `@name`-ը։ Դա DB-API-ի paramstyle չէ. դա SQL Server / T-SQL լեզվի placeholder-ն է, և `pymssql`-ը այն ընդունում է որպես named parameter դիկտով։ Այստեղից էլ գալիս է հաճախ տրվող հարցը՝ «ինչու՞ օրինակը չի աշխատում մեկ այլ բազայում»։ Կարևորն ընդհանրականն է. **ձևը կախված է driver-ից, իսկ սկզբունքը միշտ նույնն է**։

#### Ինչ է անում prepared statement-ը

`cur.execute("... WHERE name = %s", (name,))` կանչի ընթացքում.

1. Driver-ը SQL-ի տեքստը **առանձին** է ուղարկում server-ին. server-ը այն parse անում և planավորում է մեկ անգամ։
2. Parameter-ները ուղարկվում են **առանձին ուղիով** և կցվում են արժեքի մակարդակում։
3. Հետևանքը. `name`-ի մեջ ինչ էլ գրես (`' OR 1=1 --`, `'; DROP TABLE users; --`), այդ տեքստը երբեք SQL չի parse-վի։ Դա ուղղակի string արժեք կլինի։

Այս մեխանիզմի երկրորդ առավելությունը execution plan-ի cache-ն է. նույն SQL-ը տարբեր parameter-ներով օգտագործում է նույն plan-ը։

#### Սխալ և ճիշտ գրառում

```python
# ❌ Սխալ. օգտատիրոջ մուտքը դառնում է SQL տեքստ
name = input("name: ")
cur.execute("SELECT id, name FROM users WHERE name = '%s'" % name)
cur.execute(f"SELECT id, name FROM users WHERE name = '{name}'")

# ✅ Ճիշտ. ?-ը placeholder է, արժեքը գնում է parameter-ով
cur.execute("SELECT id, name FROM users WHERE name = ?", (name,))
```

Եթե մուտքը լինի `' OR '1'='1`, առաջին տարբերակում պայմանը կդառնա միշտ ճիշտ, և query-ն կվերադարձնի ամբողջ աղյուսակը։ Ծայրահեղ դեպքում `'; DROP TABLE users; --` տիպի մուտքը կկործանի տվյալները, եթե DB օգտատերը դրա իրավունքն ունի։ Երկրորդ տարբերակում այդ string-ը ուղղակի անուն չի գտնի։

#### Parameter-ի արժեքը պետք է լինի sequence

Ամենահաճախ հանդիպող bug-ը.

```python
cur.execute("... WHERE name = ?", ("Anna"))    # ❌ ("Anna")-ն string է, ոչ tuple
cur.execute("... WHERE name = ?", ("Anna",))   # ✅ մեկ տարրով tuple
```

Փակագծերը միայն tuple չեն դարձնում. դրա համար պետք է ստորակետ։ Եթե string փոխանցես այն տեղում, որտեղ driver-ը սպասում է հաջորդականություն, driver-ը կընդունի այն որպես **հաջորդականություն** և կսկսի տառ առ տառ բաշխել placeholder-ներին։ Մեկ `?`-ի դեպքում `sqlite3`-ը կտա `Incorrect number of bindings supplied` սխալը, իսկ այլ driver-ների դեպքում հնարավոր է լուռ սխալ արժեք ստանաս։ Սա նենգ է հատկապես այն պատճառով, որ `("Anna")`-ն աչքով կարդում ես որպես tuple։

#### Positional թե՞ named

- **Positional** (`?`, `%s`) արագ է, բայց չորս-հինգ parameter-ից հետո հեշտ է խառնել հերթականությունը։
- **Named** (`%(name)s`, `:name`) թույլ է տալիս կրկնել նույն արժեքը մի քանի տեղ և կարդալի է դարձնում երկար `INSERT`/`UPDATE`-ները։ Բացակայող key-ը տալիս է հստակ սխալ, այլ ոչ թե լուռ սխալ արժեք։

Արտադրական կանոն. երկար SQL-ի դեպքում named parameter-ները ավելի ապահով են մարդու սխալից։

#### Նույն սկզբունքը այլ տեղերում

Injection-ը SQL-ի մենաշնորհը չէ։ Սկզբունքը «տվյալը թող մնա տվյալ, ոչ կոդ» գործում է ամենուր, որտեղ տվյալը մտնում է կատարվող լեզվի մեջ։

| Միջավայր | Ինչպես է injection-ը առաջանում | Ճիշտ մոտեցում |
| --- | --- | --- |
| Shell / process | `subprocess.run(f"ls {user_input}", shell=True)` | `subprocess.run(["ls", user_input])` ցուցակով, առանց `shell=True` |
| HTML | `f"<p>{comment}</p>"` կամ template-ում `safe` ֆիլտրը | Jinja2-ի autoescape (լռությամբ միացած է) կամ `markupsafe.escape()` |
| SQL | string-ի միացում կամ `%`/f-string | placeholder + parameter tuple/dict |
| Log | `logger.info("user: " + name)` | parameter-ով logging. `logger.info("user: %s", name)` |

Shell injection-ի օրինակ.

```python
import subprocess

user_input = "notes.txt; rm -rf /tmp/data"   # օգտատիրոջ մուտք

# ❌ shell=True-ով մուտքը դառնում է հրամանների շղթա
subprocess.run(f"cat {user_input}", shell=True)

# ✅ ցուցակով. ;-ը ուղղակի ֆայլի անվան մաս կլինի
subprocess.run(["cat", user_input])
```

XSS-ի օրինակ.

```python
# ❌ comment-ը կարող է պարունակել <script>...</script>
html = f"<p>{comment}</p>"

# ✅ escape()-ը չեզոքացնում է HTML-ի հատուկ նշանները
from markupsafe import escape
html = f"<p>{escape(comment)}</p>"
```

#### Key concepts

- **paramstyle** DB-API-ի սահմանած placeholder-ի ոճը։
- **prepared statement** նախապես parse արված SQL, որին արժեքները կցվում են առանձին։
- **binding** parameter-ի արժեքը placeholder-ին կապելու գործողությունը։
- **SQL injection** մուտքի միջոցով SQL-ի տրամաբանությունը փոխելու հարձակումը։
- **shell injection** մուտքը shell-ի հրամանի մեջ դնելու հարձակումը, տիպիկ `shell=True`-ի հետ։
- **XSS** չեզոքացված չլինող մուտքի կատարումը զոհի browser-ում։

### Հիմնական հրամաններ

| Հրաման | Նպատակ | Սպասվող արդյունք | Անվտանգության ռիսկ |
| --- | --- | --- | --- |
| `cur.execute(sql, (a, b))` | Կատարում է SQL parameter-ներով | Տողերը cursor-ում | Անվտանգ է, եթե SQL-ը static է |
| `cur.executemany(sql, rows)` | Կրկնվող INSERT ցուցակով | Բոլոր տողերը մտան | Անվտանգ, եթե SQL-ը static է |
| `cur.description` | Սյուների անուններն ու տիպերը | tuple-ների ցուցակ | Անվտանգ |
| `conn.commit()` | Հաստատում է գործարքը | Փոփոխությունը պահպանվում է | Առանց դրա փոփոխությունը կկորչի |
| `sqlite3.connect(":memory:")` | Ժամանակավոր DB RAM-ում | Աշխատող միացում | Անվտանգ է lab-ի համար |
| `subprocess.run(["cmd", arg])` | Արտաքին հրաման առանց shell-ի | Արդյունքը stdout-ում | Անվտանգ է, մի օգտագործիր `shell=True` մուտքի հետ |

### Փորձարկում (Lab)

Միջավայր սովորական Ubuntu VM. `sqlite3`-ը Python-ի հետ լռությամբ հասանելի է, ոչ մի լրացուցիչ տեղադրում պետք չէ։

```bash
cat > /tmp/sql_injection_demo.py <<'PY'
import sqlite3

conn = sqlite3.connect(":memory:")
cur = conn.cursor()
cur.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
cur.executemany("INSERT INTO users (name) VALUES (?)", [("Anna",), ("Bob",)])
conn.commit()

user_input = "' OR '1'='1"     # անմեղ տեսք ունեցող որոնման տող

# ❌ string-ի միացում. injection
bad = "SELECT id, name FROM users WHERE name = '%s'" % user_input
print("bad sql :", bad)
print("bad rows:", cur.execute(bad).fetchall())

# ✅ parameter. մուտքը մնում է արժեք
print("ok rows :", cur.execute(
    "SELECT id, name FROM users WHERE name = ?", (user_input,)).fetchall())

# Սխալ tuple-ի ցուցադրում. ("Anna")-ն string է
try:
    cur.execute("SELECT id, name FROM users WHERE name = ?", ("Anna"))
except sqlite3.ProgrammingError as e:
    print("tuple bug:", e)

conn.close()
PY

python3 /tmp/sql_injection_demo.py
```

Սպասվող արդյունք.

1. `bad sql`-ը կտպի `SELECT id, name FROM users WHERE name = '' OR '1'='1'`։ Այստեղ պայմանը դարձել է միշտ ճիշտ, և `bad rows`-ը կվերադարձնի **երկու** տող Anna-ն ու Bob-ը, այն դեպքում, երբ որոնումը ուղղակի անուն էր փնտրում։ Սա հենց injection-ի էությունը.
2. `ok rows`-ը կվերադարձնի դատարկ ցուցակ, որովհետև այդ անունով օգտատեր չկա։ Մուտքի տեքստը չմեկնաբանվեց որպես SQL.
3. `tuple bug`-ը կտա `Incorrect number of bindings supplied. The current statement uses 1, and there are 4 supplied`։ Սա ցույց է տալիս, որ `("Anna")`-ն string էր, և driver-ը չորս տառերը համարել էր չորս parameter.

!!! tip "Best Practices"
    - Երբեք մի կառուցիր SQL-ը f-string-ով, `%`-ով կամ `+`-ով, անկախ այն բանից, թե ինչ է մուտքը.
    - Մեկ parameter-ի համար միշտ գրիր tuple ստորակետով. `("Anna",)`.
    - Երկար SQL-ի համար օգտագործիր named parameter-ներ.
    - `subprocess`-ում մուտքը փոխանցիր ցուցակով, `shell=True`-ը հեռու պահիր.
    - HTML-ում վստահիր template-ի autoescape-ին, մի գործադրիր `| safe` օգտատիրոջ տվյալի վրա.
    - DB օգտատիրոջը տուր միայն անհրաժեշտ իրավունքները, տես [Least Privilege](security.md#least-privilege)։

### Իրական DevOps իրավիճակ

#### Ախտանիշ

Որոնման endpoint-ը սկսում է տարօրինակ վարվել. դատարկ որոնումը վերադարձնում է ամբողջ ցանկը, իսկ օգտատերերը տեսնում են ուրիշների տվյալները։ Մեկ շաբաթ անց աղյուսակներից մեկը անհետանում է։ DB log-երում երևում են անսովոր, կրկնվող query-ներ, որոնք ոչ ոք չի գրել։

#### Ախտորոշում

1. Գտիր բոլոր տեղերը, որտեղ SQL-ը կառուցվում է դինամիկ.
   ```bash
   grep -rnE "execute\(.*(f\"|%|\+ )" app/ || true
   grep -rn "shell=True" app/ || true
   ```
2. Ստուգիր DB log-երում անսովոր նախշեր. `' OR`, `--`, `UNION SELECT`, `; DROP`.
3. Ստուգիր, թե ինչ իրավունքներ ունի app-ի DB օգտատերը.
   ```sql
   SELECT current_user;
   ```
4. Եզրակացրու. եթե օգտատերը ունի `DROP`/`ALTER` իրավունք, վնասի շառավիղը շատ ավելի մեծ է, քան միայն տվյալների արտահոսքը։

#### Լուծում

Կարճաժամկետ՝ փակիր վնասված endpoint-ը, պտտիր DB credential-ը և վերականգնիր տվյալները backup-ից։ Երկարաժամկետ՝ անցիր parameter-ներին։

```python
# ❌ առաջ
cur.execute(f"SELECT id, name FROM users WHERE name = '{name}'")

# ✅ հետո
cur.execute("SELECT id, name FROM users WHERE name = ?", (name,))
```

Եթե աղյուսակի անունը պետք է դինամիկ լինի, parameter չի օգնի. այդ դեպքում օգտագործիր allowlist։

```python
ALLOWED_TABLES = {"users", "orders"}

table = "orders"
if table not in ALLOWED_TABLES:
    raise ValueError("անթույլատրելի աղյուսակ")
cur.execute(f"SELECT count(*) FROM {table}")   # արժեքը արդեն allowlist-ից է
```

#### Կանխարգելում

- CI-ում գործարկիր static analysis (`bandit`, `semgrep`) SQL-ի string-կառուցման նախշերի համար։
- DB օգտատիրոջը տուր միայն `SELECT`/`INSERT`/`UPDATE` իրավունքները app-ի սխեմայի վրա (least privilege)։
- Արտաքին մուտքը վալիդացրու երկարությամբ և նախշով, բայց վալիդացիան **չի** փոխարինում parameter-ներին։
- Ամեն նոր query-ի code review-ում պահանջիր placeholder կամ allowlist։
- `shell=True`-ը դարձրու արգելված նախշ linter-ի կանոնում։

### Հարցազրույցի հարցեր և պատասխաններ

#### Ինչու՞ է `%s`-ը SQL-ում ապահով, իսկ `%`-ով string formatting-ը ոչ (mid-level)

`%s`-ը placeholder է, որը driver-ը փոխարինում է parameter-ի արժեքով **SQL-ի parse-ից հետո**. Արժեքը երբեք չի մասնակցում SQL-ի կառուցմանը, այդ պատճառով էլ մեջը գրված `' OR 1=1`-ը ուղղակի string է։ `%`-ը Python-ի operator է, որը արժեքը կպցնում է տեքստին մինչև driver-ին հասնելը. այդ պահին `' OR 1=1`-ը դառնում է SQL-ի syntax-ի մաս, այստեղից էլ injection-ը։

#### Ինչպե՞ս է prepared statement-ը պաշտպանում, և ինչու՞ parameter-ները բավարար չեն դինամիկ աղյուսակի/սյունակի անվան համար (senior-level)

Prepared statement-ը SQL-ի տեքստը և արժեքները տարանջատում է protocol-ի մակարդակով. server-ը parse-ում ու planավորում է SQL-ը, հետո parameter-ները գալիս են առանձին ուղիով որպես արժեքներ և կապվում են placeholder-ներին։ Այդ պատճառով մուտքի մեջ գրված SQL-ի syntax-ը անիմաստ է. այն չի անցնում parser-ով։ Բայց placeholder-ը կարող է պահել **միայն արժեք**, ոչ identifier. աղյուսակի անունը, սյունակի անունը կամ `ORDER BY`-ի ուղղությունը չեն կարող parameter լինել, որովհետև դրանք plan-ի կառուցվածքի մաս են, ոչ տվյալ։ Այդ դեպքի համար օգտագործում ես սերվերի կողմից վերահսկվող allowlist, այսինքն երաշխավորում ես, որ արժեքը գալիս է կոդից, ոչ օգտատիրոջից։ Հավելում. նույն խնդիրը կա նաև SQL-ից դուրս. shell-ը, HTML-ը և log-ը ունեն իրենց parameter-ացման մեխանիզմը, իսկ ընդհանուր սկզբունքը՝ «տվյալը երբեք կոդ մի դարձրու»։

### Ինքնաստուգում

1. Ինչ է տարբերությունը `qmark`-ի, `format`-ի և `named` paramstyle-ների միջև, և ինչու՞ է այն կարևոր port-ի ժամանակ։
2. Ինչու՞ է `("Anna")`-ն վտանգավոր SQL-ում, և ինչպիսի՞ սխալ կտա `sqlite3`-ը։
3. Ինչպե՞ս ստուգել, որ SQL-ը injection-ի ենթակա է, այսինքն ինչի՞ վրա նայել։
4. Ինչու՞ parameter-ները չեն լուծում դինամիկ աղյուսակի անվան խնդիրը, և ի՞նչ է allowlist-ը։
5. Բերիր shell injection-ի օրինակ Python-ում և ցույց տուր ապահով տարբերակը։
6. Ինչպե՞ս է least privilege-ը նվազեցնում injection-ի վնասը, նույնիսկ երբ խոցելիությունը դեռ չի փակվել։

### Հաջորդ քայլեր

Անցիր [Least Privilege](security.md#least-privilege), որտեղ DB օգտատիրոջ իրավունքը, secret-ի պտույտը և մուտքի նվազագույն մակարդակը ցույց են տալիս, թե ինչպես սահմանափակել վնասը, երբ ինչ-որ շերտ բաց է թողնվել։ Ապա վերադարձիր [Python մեթոդներ և type hints](#python-methods), եթե պետք է մուտքի արժեքները փոխարկել ճիշտ տիպերի, կամ [Automation և IaC](automation-iac.md)՝ գաղտնիքների կառավարումը Ansible Vault-ով ավտոմատացնելու համար։

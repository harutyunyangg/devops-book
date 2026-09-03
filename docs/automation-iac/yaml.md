# YAML. Տվյալների ձևաչափ DevOps-ում (Ansible, Docker, Kubernetes, CI/CD)

## Տեսություն

**YAML** (նախկինում՝ *YAML Ain't Markup Language*, «YAML-ը նշագրման լեզու չէ») մարդու համար ընթեռնելի **տվյալների սերիալիզացման** ձևաչափ է՝ կոնֆիգուրացիան նկարագրելու համար։ Ի տարբերություն JSON-ի՝ YAML-ը թույլ է տալիս մեկնաբանություններ և «չկրկնվող» գրառում՝ ընդհանուր կարգավորումները reusable դարձնելով anchor-ներով ու alias-ներով։ DevOps-ում YAML-ը *դե ֆակտո* ստանդարտ է. Ansible playbook-ները, Docker Compose-ը, Kubernetes-ի մանիֆեստները և GitHub Actions-ի/GitLab CI-ի pipeline-ները գրվում են YAML-ով։ Դրա հզորությունն այն է, որ կոդը կարդացվում է արագ, հեշտ է պահվում Git-ում (ամեն změնավորությունը՝ տեսանելի), իսկ սխալը հնարավոր է հայտնաբերել մինչև գործարկումը։ Git-ի և պիփլայնների մասին՝ [Git և CI/CD](../git-ci-cd/index.md) բաժնում, իսկ YAML-ի գործնական կիրառման՝ [Ansible](ansible.md) էջում։

### Ի՞նչ խնդիր է լուծում

Միջավայրի կոնֆիգուրացիան (որ փոփոխականը՝ ինչ արժեքով) նկարագրվում է կոդով, որպեսզի այն դառնա կրկնվող, վերանայելի և վարկածավորված։ Առանց կառուցվածքային ձևաչափի կարգավորումները մնում են ձեռքի, մարդու հիշողության վրա, և առաջանում է **config drift** (կոնֆիգուրացիիայի շեղում)՝ յուրաքանչյուր սերվեր ժամանակի atravesti դառնում է տարբեր։ YAML-ը տալիս է միասնական «լեզու», որով և մարդը, և ծրագիրը նույնպես են հասկանում, թե ինչ պետք է լինի համակարգում։ Չthose播出шь readability-ն, YAML-ը մեքենայական ձևաչափ է, ուստի նրա «կտրուկ» կանոնները (նահանջի ճշտություն, արժեքի տիպեր) պետք է խստորեն պահպանել:

### Ինչպե՞ս է աշխատում

- **Key–value**՝ ամենատարրական միավորը, `key: value`։ Բանալին և արժեքը բաժանվում են երկու կետից ու բացատից հետո.
- **Nested** (տեղադրված) կառուցվածք՝ ավելի խոր մակարդակի բանալիները սկսվում են ավելի մեծ նահանջով (indentation), սովորաբար 2 կամ 4 բացատ։ YAML-ը **Tab նիշն արգելում է** որպես նահանջ:
- **Նահանջի միատեսակություն**՝ միևնույն մակարդակի (level) բանալիները պետք է ունենան ճիշտ նույն քանակի բացատներ, հակառակ դեպքում փաստաթուղթն անվավեր է (invalid):
- **Collections**՝ YAML-ում կա երկու հիմնական կառուցվածք. **map** (բանալի-արժեք զույգերի բազմություն) և **list** (հերթական ցուցակ, որի տարրերը սկսվում են «-» գծիկով):
- **Scalars** (պարզ արժեքներ)՝ տեքստ (string), թիվ (number), ճշմարիտ/կեղծ (boolean), դատարկ (`null`), ինչպես նաև ժամանակ/ամսաթebab տիպեր, որոնք YAML-ը ճանաչում է ըստ արժեքի տեսքի:
- **Parsing-ի «կանխատեսելիություն»**՝ YAML-ը փորձում է գուշակել արժեքի տիպը. օրինակ, `yes`/`no`-ն, `on`/`off`-ը և `true`/`false`-ը դառնում են boolean: Ուստի տեքստային արժեքները, որոնք նման են այդ բանալի-բառերին կամ հատուկ նշաններ են պարունակում (`:`), ավելի անվտանգ է չակերտել:
- **Document markers**՝ «---»-ը նշում է նոր փաստաթղթի սկիզբը (երբ մեկ ֆայլում մի քանի փաստաթուղթ կա), իսկ «...»-ը՝ վերջը (հազվադեպ է օգտագործվում):

### Հիմնական հայեցակարգեր

**[1] Դատարկ արժեք (`null`)** — ասում է՝ «այս բանալին չունի արժեք»*. Գրելիս ե՛րեք համարժեք եղանակ՝ `key: null` (կամ `~`, կամ բանալուց հետո դատարկ lämնելը):

```yaml
անուն: Արծան
հեռախոս: null     # առաջին տարբերակ
էլփոստ: ~        # երկրորդ տարբերակ (ավելի կարճ)
հասցե:            # երրորդ տարբերակ (դատարկ lämնել)
```

**Իրական օրինակ (Docker Compose).**
```yaml
services:
  web:
    image: nginx
    environment:
      DATABASE_URL: null   # նշանակում է՝ չսահմանել այս փոփոխականը
```

Այս հատկությունը հաճախ օգտագործվում է «չսահմանված» փոփոխական ցույց տալու համար։ Օրինակ, Docker Compose-ում `environment`-ի արժեքը `null` դնելը նշանակում է «այս միջավայրի փոփոխականը չսահմանել» (այն ժառանգվում է host-ի միջավայրից).

**[2] Տրամաբանական արժեքներ (boolean)** — YAML-ն ընդունում է `true`/`false`, ինչպես նաև `yes`/`no` և `on`/`off` համարժեքները (YAML 1.1).

```yaml
միացված: true
անջատված: false
այո: yes
ոչ: no
ակտիվ: on
պասիվ: off
```

**Կարևոր.** «yes»-ը և «no»-ն կարող են խնդիր առաջացնել, եթե դրանք տեքստ են (օրինակ՝ քաղաքի անուն «Norway»-ն «no» չի համարվի, քանի որ մեծատառով է)։ Բայց զգույշ եղեք, որ «on»-ը կամ «off»-ը պատահաբար չօգտագործեք տեքստ որպես։

**Իրական օրինակ (Ansible).**
```yaml
- name: Տեղադրել nginx
  apt:
    name: nginx
    state: present
  become: yes    # աշխատել admin իրավունքներով
```

**[3] Բազմակի տող` folded (`>`)** — բոլոր նոր տողերը վերածվում են բացատների (մեկ երկար տող)։

```yaml
նկարագրություն: >
  Սա շատ երկար տեքստ է,
  որը գրված է մի քանի տողերում,
  բայց YAML-ը այն կմիացնի մեկ տողի մեջ։
  Այս տողը նույնպես կմիանա։
```

**Արդյունք.**
```
Սա շատ երկար տեքստ է, որը գրված է մի քանի տողերում, բայց YAML-ը այն կմիացնի մեկ տողի մեջ։ Այս տողը նույնպես կմիանա։
```

**Իրական օրինակ (GitLab CI).**
```yaml
deploy:
  script:
    - echo "Տեղադրում եմ..."
    - >
      ssh user@server
      "cd /app && git pull && npm install && pm2 restart app"
```
Այստեղ ամբողջ ssh հրամանը միանում է մեկ տողի, բայց մարդու համար հեշտ է կարդալ։

**[4] Բազմակի տող` literal (`|`)** — Պահպանում է բոլոր նոր տողերը։

```yaml
բանաստեղծություն: |
  Ձյուն է գալիս,
  Սպիտակ փաթիլներ,
  Ծածկում են քաղաքը։
  Ամեն ինչ լռում է։
```

**Արդյունք (պահպանում է ճիշտ այդպես).**
```
Ձյուն է գալիս,
Սպիտակ փաթիլներ,
Ծածկում են քաղաքը։
Ամեն ինչ լռում է։
```

**Իրական օրինակ (Kubernetes ConfigMap).**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    database.host=postgres
    database.port=5432
```
Այստեղ ամբողջ կոնֆիգուրացիան պահվում է իր սկզբնական ձևով (տող առ տող)։

**[5] Anchor — `&անուն`** — Ստեղծում է «կպչուկ» (bookmark) որևէ արժեքի համար, որպեսզի հետո կարողանաք օգտագործել այն։

```yaml
defaults: &defaults
  timeout: 30
  retries: 3
  protocol: http
```

Այստեղ `&defaults`-ը «կպցնում» է ամբողջ `defaults` բլոկը։

**[6] Alias — `*անուն`** — Օգտագործում է նախկինում ստեղծված anchor-ը։

```yaml
service1:
  <<: *defaults
  host: example.com
  port: 80
```

**Բացատրություն.** `*defaults`-ը ասում է. «բեր այն ամենը, ինչ կա defaults-ի մեջ»։

**[7] Merge — `<<: *անուն`** — Միավորում է (merge) anchor-ի բոլոր արժեքները ընթացիկ բլոկի հետ։

**Ամբողջական օրինակ.**
```yaml
defaults: &defaults
  timeout: 30
  retries: 3
  protocol: http

service1:
  <<: *defaults
  host: example.com
  port: 80

service2:
  <<: *defaults
  host: test.com
  port: 443
  protocol: https   # վերաիմաստավորում է defaults-ի protocol-ը
```

**Merge երկու anchor-ից (multi-source merge).**
```yaml
base: &base
  replicas: 2
  image: nginx
  resources:
    limits:
      cpu: 100m

logging: &logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

app:
  <<: *base
  <<: *logging
  name: myapp
  image: myapp:latest
  replicas: 3
```

**Ցուցակի սինտաքսով ( sequência).**
```yaml
app:
  <<: [*base, *logging]
  name: myapp
  image: myapp:latest
  replicas: 3
```
Երկու տարբերակները էլ հավասար են և տալիս են նույն արդյունքը։ Ցուցակի սինտաքսը ([*]) ավելի փոքր է, սակայն մի քանի <<: տողեր ավելի ընթեռնելի են՝ տարբեր merging-ների համար տարբեր annotation-ներ նախագծված դեպքում։**
```yaml
app:
  replicas: 3
  image: myapp:latest
  resources:
    limits:
      cpu: 100m
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
  name: myapp
```
Երկու `<<:` Միաձուլում են և `base`-ի, և `logging`-ի բոլոր բանալիները՝ իսկ `replicas`-ն ու `image`-ը վերաիմաստացնում են `base`-ի արժեքը։

**Ստացվում է.**
```yaml
service1:
  timeout: 30
  retries: 3
  protocol: http
  host: example.com
  port: 80

service2:
  timeout: 30
  retries: 3
  protocol: https
  host: test.com
  port: 443
```

**Իրական օրինակ (Kubernetes Helm).**
```yaml
base: &base
  replicas: 2
  image: nginx
  resources:
    limits:
      cpu: 100m

app1:
  <<: *base
  name: app1
  image: myapp:latest   # վերաիմաստավորում է image-ը

app2:
  <<: *base
  name: app2
  replicas: 5           # վերաիմաստավորում է replicas-ը
```

**[8] Մեկնաբանություն — `#`** — Ամեն ինչ `#`-ից հետո մինչև տողի վերջը անտեսվում է։

```yaml
# Սա մեկնաբանություն է ամբողջ տողի համար
անուն: Արծան   # սա նույնպես մեկնաբանություն է
տարիք: 30      # կարող եք բացատրել, թե ինչու է այս արժեքը
```

**Իրական օրինակ (Docker Compose).**
```yaml
version: '3'
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    # Այս կոմենտը բացատրում է, որ օգտագործում ենք alpine տարբերակը չափը փոքրացնելու համար
```

**[9] Փաստաթղթի սկիզբ — `---`** — Նշում է, որ սկսվում է նոր YAML փաստաթուղթ։ Օգտագործվում է, երբ մեկ ֆայլում մի քանի առանձին YAML փաստաթղթեր կան (օրինակ՝ Kubernetes-ում)։

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
```

**Իրական օրինակ (Kubernetes).** Շատ հաճախ մեկ YAML ֆայլում նկարագրվում են մի քանի ռեսուրսներ (Pod, Service, ConfigMap), և դրանք բաժանվում են `---`-ով։

**[10] Փաստաթղթի վերջ — `...`** — Նշում է, որ փաստաթուղթն ավարտվել է։ Գրեթե չի օգտագործվում, բայց կա ստանդարտում։

```yaml
---
անուն: Արծան
...
---
անուն: Աննա
...
```

### Ընդհանուր ամփոփ աղյուսակ (հեշտ հիշելու համար)

| Ի՞նչ եք ուզում | Օգտագործեք | Օրինակ |
|---------------|------------|--------|
| Դատարկ արժեք | `null` կամ `~` | `key: null` |
| Ճշմարիտ / կեղծ | `true`/`false`, `yes`/`no`, `on`/`off` | `enabled: yes` |
| Տեքստ (միացված տողեր) | `>` | `text: >` <br> `  սա միանում է` |
| Տեքստ (պահպանված տողեր) | `\|` | `text: \|` <br> `  սա մնում է` |
| Ստեղծել հղում | `&անուն` | `base: &base` |
| Օգտագործել հղում | `*անուն` | `service: *base` |
| Միավորել հղումը | `<<: *անուն` | `<<: *base` |
| Մեկնաբանություն | `#` | `# բացատրություն` |
| Փաստաթղթի սկիզբ | `---` | `---` |
| Փաստաթղթի վերջ | `...` | `...` |

### Գործնական խորհուրդներ

1. **Միշտ օգտագործեք բացատներ, ոչ թե Tab:** YAML-ը Tab-ը չի սիրում։
2. **Նահանջները պետք է հավասար լինեն:** եթե մեկ տեղում 2 բացատ եք դնում, ամենուր 2 բացատ դրեք (կամ 4, բայց մի խառնեք)։
3. **Օգտագործեք online validator:** կարող եք YAML-ը ստուգել [yamlvalidator.com](https://www.yamllint.com/) կամ IDE-ի plugin-ով (օրինակ՝ VSCode-ում `YAML` ընդլայնումը)։
4. **Տեքստերում չակերտները անհրաժեշտ չեն,** բայց եթե ունեք հատուկ նշաններ (`:`, `#`, `@`), ավելի լավ է չակերտների մեջ վերցնել։

### Իրական կյանքի օրինակ (բոլորը միասին)

```yaml
# Docker Compose ֆայլ
---
version: '3.8'

# Ընդհանուր կարգավորումներ
defaults: &defaults
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

services:
  # Web ծառայություն
  web:
    <<: *defaults
    image: nginx:alpine
    ports:
      - "80:80"
    environment:
      NODE_ENV: production   # սա տեքստ է
      DEBUG: "false"         # boolean, բայց չակերտներով անվտանգ է
    volumes:
      - ./html:/usr/share/nginx/html:ro   # ro = read-only

  # Database ծառայություն
  db:
    <<: *defaults
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ~   # դատարկ արժեք (նշանակում է՝ չսահմանել)
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:   # դատարկ volume
```

### Հիմնական հրամաններ / գործիքներ

«Հրամաններ» այս թեմայում YAML-ի **ստուգման (validation)** և **parsing-ի** գործիքներն են, որոնք DevOps-ում անհրաժեշտ են` նախքան կոնֆիգուրացիան deploy անելը.

| Հրաման | Նպատակ |
| --- | --- |
| `python -c "import yaml,sys; yaml.safe_load(sys.stdin)"` (ֆայլն ուղղելով) | Ստուգում է YAML-ի վավերականությունը (syntax)` ոչինչ չկատարելով |
| `python -c "import yaml; print(yaml.safe_load(open('x.yml')))"` | Բեռնում է և ցույց է տալիս parsed-տվյալների կառուցվածքը (Python dict/list) |
| `yq eval . file.yml` | YAML-ի «jq»-ն. արագ արժեք հանելու և ստուգելու համար |
| IDE plugin (օր. VSCode «YAML» ընդլայնում) | Իրական ժամանակում ցույց է տալիս սխալներն ու schema-ով ավտոլրացումը |

## Փորձարկում (Lab)

Միջավայրը` սովորական Linux VM կամ Python-ով Docker կոնտեյներ (`python:alpine`)։ Ստորև աշխատում ենք Python-ի PyYAML գրադարանով` YAML-ի parsing-ի ամենատարածված իրականացումը.

### Քայլ 1. Միջավայրի պատրաստում

```bash
python3 -m pip install pyyaml
```

Ստուգենք, որ PyYAML-ը հասանելի է.

```bash
python3 -c "import yaml; print(yaml.__version__)"
```

Սպասվող արդյունք` տարբերակի համար (օր.` 6.0.1) որը հաստատում է, որ գրադարանը պատրաստ է.

### Քայլ 2. Բազային կառուցվածքի ստուգում

Ստեղծեք `demo.yml`.

```yaml
# Դատարկ արժեք
անուն: ~
# boolean
միացված: on
# folded / literal
նկարագրություն_a: >
  Սա միանում է
  մեկ տողի մեջ
նկարագրություն_b: |
  Սա մնում է
  տող առ տող
```

Ստուգեք parsing-ը.

```bash
python3 -c "import yaml; print(yaml.safe_load(open('demo.yml')))"
```

Սպասվող արդյունք` Python dict` `{'անուն': None, 'միացված': True, ...}`։ Նկատեք` «`on`»-ը boolean դարձավ, «`~`»-ը` None-ի, `նկարագրություն_a`-ի արժեքը` մեկ տող, իսկ `_b`-ինը` տող առ տող:

### Քայլ 3. Anchor, alias և merge

Ստեղծեք `anchors.yml`.

```yaml
defaults: &defaults
  timeout: 30
  retries: 3

service1:
  <<: *defaults
  host: example.com
  port: 80

service2:
  <<: *defaults
  host: test.com
  port: 443
  retries: 5          # վերաիմաստավորում
```

Ստուգեք.

```bash
python3 -c "import yaml; print(yaml.safe_load(open('anchors.yml')))"
```

Սպասվող արդյունք` `service1`-ին և `service2`-ին ավելացված են `timeout`-ն ու base-ի `retries`-ը, իսկ `service2.retries`-ը` 5 է. merge-ը թույլ է տալիս մասնակի վերաիմաստավորում` առանց base-ի փոփոխության.

### Քայլ 4. Սխալի ցուցադրում. Tab նիշ և անհամաչափ նահանջ

Ստեղծեք `bad.yml`, որի երկրորդ տողի նահանջը դրված է **Tab** նիշով (կամ խառնեք 2 և 4 բացատ միևնույն մակարդակում).

```yaml
անուն: Փորձ
	հասցե: Սխալ    # Tab-ով նահանջ
```

(Տեղադրեք նշված տեքստը, փոխարինելով նահանջը Tab-ով) ապա ստուգեք.

```bash
python3 -c "import yaml; yaml.safe_load(open('bad.yml'))"
```

Սպասվող արդյունք` `yaml.scanner.ScannerError`-ը դադարեցնում է parsing-ը` «found character '\t' that cannot start any token» (կամ `mapping values are not allowed here`) հաղորդագրությամբ։ Սա ապացուցում է, որ YAML-ը պահանջում է միատեսակ` բացատանման նահանջ` Tab-ը` արգելված.

### Քայլ 5. Բոլորը միասին. «իրական կյանքի» փաստաթուղթ

Ստեղծեք `compose-like.yml`.

```yaml
---
version: '3.8'

# Ընդհանուր կարգավորումներ
defaults: &defaults
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

services:
  # Web ծառայություն
  web:
    <<: *defaults
    image: nginx:alpine
    ports:
      - "80:80"
    environment:
      NODE_ENV: production   # սա տեքստ է
      DEBUG: "false"         # boolean, բայց չակերտներով անվտանգ է
    volumes:
      - ./html:/usr/share/nginx/html:ro   # ro = read-only

  # Database ծառայություն
  db:
    <<: *defaults
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ~   # դատարկ արժեք (նշանակում է՝ չսահմանել)
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:   # դատարկ volume
```

Ստուգեք ամբողջը.

```bash
python3 -c "import yaml; d=yaml.safe_load(open('compose-like.yml')); print(d['services']['web']['image'], d['services']['db']['environment'])"
```

Սպասվող արդյունք՝ `nginx:alpine` և `{'POSTGRES_USER': 'admin', 'POSTGRES_PASSWORD': None}`. Այստեղ մեկ հայացքից տեսանելի է՝ `<<:` merge-ը՝ `~` null-ը՝ և «`"false"`» չակերտումը.

! Best Practices
    - Միշտ բացատներով նահանջ՝ երբեք Tab. և միևնույն մակարդակում՝ նույն քանակի բացատ (2 կամ 4՝ բայց չխառնել)։
    - Տեքստային արժեքները՝ որոնք նման են `yes`/`no`/`on`/`off`/`null`/`true`/`false`-ի կամ պարունակում են `:` կամ `#`-ի՝ չակերտեք՝ որպեսզի parsing-ը դրանց տիպը չփոխի:
    - Ճշգրտով տարբերակեք «`key: null`»-ը՝ «բանալին բացակայում է»-ից. առաջինում բանալին կա՝ արժեքը` դատարկ. երկրորդում` բանալին չկա:
    - Օգտագործեք «`---`»՝ երբ մեկ ֆայլում մի քանի փաստաթուղթ կա (օր. Kubernetes-ի մանիֆեստները)։
    - Բարդ արժեքների դեպքում գրեք մեկնաբանություն (`#`)՝ որ թիմը հասկանա՝ ինչու է այդ արժեքը:
    - Deploy-ից առաջ ստուգեք syntax-ը (օր. Python+PyYAML կամ `yq`)՝ նախքան Kubernetes/Ansible/CI-ում գործարկելը:

## Իրական DevOps իրավիճակ

### Ախտանիշ

Kubernetes-ում deploy-ի ժամանակ գործարկվում է `kubectl apply -f config.yaml`, բայց ստացվում է մոտավոր սխալ.

```
error: error validating "config.yaml": error validating data:
[ValidationError(...): unknown field ...]
```

Մանրամասն վերլուծությամբ պարզվում է, որ ռեսուրսի մի մաս्स (օր. ConfigMap-ի `data`-ն) դատարկ է կամ սխալ տիպով է, կամ ռեսուրսն ընդհանրապես չի ստեղծվում.

### Ախտորոշում

1. Ստուգեք, թե որ տողում է սխալը՝ `kubectl apply --dry-run=client -f config.yaml -o json`.
2. Բացեք ռեսուրսի YAML-ը և ուսումնասիրեք նահանջը. հաճախ մեղավորը **Tab նիշն** է կամ 2/4 բացատի «խառն» օգտագործումը, որի պատճառով բանալին «կորցնում» է իր արժեքը:
3. Ստուգեք արժեքի տիպը. օրինակ՝ `enabled: yes`-ը boolean է դառնում, և եթե կոդում ակնկալվում է տեքստ, արժեքը սխալ է մեկնաբանվում:
4. Փորձեք parsing-ը Python-ով. `python3 -c "import yaml; print(yaml.safe_load(open('config.yaml')))"` և նայեք, արդյոք `data`-ն `None` կամ list է դառն católica map-ի փոխարեն.

### Լուծում

- Ուղղեք նահանջը` բոլոր դաշտերը մեկ մակարդակում դնելով միևնույն քանակի բացատով (առանց Tab).
  ```bash
  # Tab-ը տեսնելու համար
  cat -A config.yaml   # ^I-ը Tab նիշն է
  ```
- Եթե արժեքը պետք է տեքստ լինի, չակերտեք այն՝ «`"no"`», «`"false"`» և այլն:
- Դատարկ դաշտի դեպքում՝ կամ հանեք այն, կամ ստեղծեք ակնհայտ `null` (եթե դա թույլատրելի է), կամ տվեք ակնկալվող արժեք:
- Օգտագործեք schema-ով ավտոլրացում տվող IDE plugin-ը (օր. VSCode «YAML» + Kubernetes schema)՝ որը սխալը ցույց է տալիս մինչև `kubectl` գործարկելը:

### Կանխարգելում

- **Deploy-ից առաջ syntax/validation** `kubectl apply --dry-run=client` + Python+PyYAML կամ `yq`:
- **CI-ում YAML lint**՝ (օր. `yamllint`)` որը indentation/type-սխալը բռնի pipeline-ում:
- **Նահանջի ստանդարտ**՝ նախագծում բոլոր YAML-ում 2 բացատ` ոչ Tab:
- **Schema validation**՝ Kubernetes-ի/Ansible-ի համար` որպեսզի type/schema-սխալը բռնվի մինչև deploy:

## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչու «`:`, `#`» պարունակող տեքստն անհրաժեշտ է չակերտել՝ և ի՞նչ ռիսկ կա այն չակերտելու դեպքում (mid-level)

«`:`»-ը YAML-ի map-բաժանարարն է. եթե արժեքը պարունակում է «`key: value`», YAML-ը դա կհամարի կեղծ map-ենթակառուցվածք կամ կսխալվի. իսկ «`#`»-ը մեկնաբանության սկիզբ է, հետևաբար դրանից հետո ամեն ինչ կկորչի։ Բացի այդ՝ տիպը կարող է անսպասելի փոխվել, օրինակ `yes`/`no`/`on`/`off`/`null`-ը boolean/null են դառն bắc՝ ոչ թե տեքստ։ Անվտանգ և «նույն ինքը» արժեք պահելու համար պարզապես չակերտեք («`"yes"`», «`"#"`», «`"host:80"`»).

### Ի՞նչո՞ւ է «`<<: *base`» merge-ը (և ոչ `*alias`-ը) նախընտրելի «պատճենել և পরিবর্তել» սցենարում (senior-level)

«`*alias`»-ը բերում է **այնmateix օբյեկտը**, որին anchor-ն է հղում. այն *reference* է, ոչ թե *copy*։ Ուստի alias-ի միջոցով արժեքը փոփոխելիս կարող եք «վնասել» բնօրինակ anchor-ը (և բոլոր մյուս alias-ները)։ Փոխարենը՝ «`<<:`»-ը **map-ի merge** է, որը anchor-ի բանալի-արժեք զույգերը միավորում է ընթացիկ block-ում, որտեղ ընթացիկ block-ի առկա բանալիները **գերակշռում** (override) են, և արդյունքում ստեղծվում է **նոր** data-կառուցվածք՝ բնօրինակ anchor-ն անփոփոխ մնալով։ Սա «inherit + override»-ի թեթև համարժեքն է՝ որը օգտագործվում է՝ նույն base-ը (օր. logging, security context) տարբեր սերվիսներին` տարբեր մասնակի փոփոխություններով կիրառելու համար:

## Ինքնաստուգում

1. Ի՞նչ տարբերություն կա «`>`» (folded)-ի և «`|`» (literal)-ի միջև, և ե՞րբ կօգտագործեք յուրաքանչյուրը:
2. Ի՞նչ ռիսկ կա `yes`/`no`/`on`/`off` արժեքներ օգտագործելիս, և ինչպե՞ս վերածել դրանք պարզ տեքստի:
3. Ինչու՞ «Tab» նիշը արգելված է YAML-ում, և ինչպե՞ս հայտնաբերել այն ֆայլում:
4. Ի՞նչ է anchor-ը, alias-ը և merge-ը («`&`», «`*`», «`<<:`»), և ինչո՞վ է alias-ը տարբերվում merge-ից:
5. Ե՞րբ է պարտադիր արժեքը չակերտել («`"..."`»)` օրինակ բերեք:
6. Ի՞նչ է նշանակում `key: null` ընդդեմ «բանալին պարզապես բացակայում է»-ի, և ինչու կարող է դա կարևոր լինել:
7. Ի՞նչու՞ և ե՞րբ է անհրաժեշտ «`---`»-ը, և որտե՞ղ է դրա ամենատարածված կիրառությունը (Kubernetes):

## Հաջորդ քայլեր

Այժմ, երբ YAML-ի սինտաքսը ձերն է, ամրապնդեք այն գործնականում՝ [Ansible](ansible.md), որտեղ playbook-ները հենց YAML-ով են գրվում (inventory, tasks, handlers, `.j2` templates), կամ [Git և CI/CD](../git-ci-cd/index.md), որտեղ pipeline-ի նկարագրությունը նույն YAML-ով է։ Խորացման հաջորդ քայլը՝ Jinja2-ի ձևանմուշները Ansible-ում, ապա` Terraform-ը cloud-ի IaC-ի համար:
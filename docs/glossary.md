# Բառարան

| Տերմին | Բացատրություն |
| --- | --- |
| PID | Process ID — գործող process-ի եզակի նույնացուցիչ |
| PPID | Parent Process ID — process-ը գործարկած ծնողի PID |
| Signal | Kernel-ի հաղորդագրություն process-ին, օրինակ՝ `SIGTERM` |
| systemd | Շատ Linux համակարգերի service manager և init system |
| Runbook | Կրկնվող operational խնդրի քայլ առ քայլ գործնական հրահանգ |
| IaC | Infrastructure as Code. ենթակառուցվածքի կարգավորումները կոդով նկարագրելու մոտեցում (օր.՝ Ansible, Terraform) |
| Ansible | Agentless ավտոմատացման գործիք. SSH-ով կիրառում է YAML playbook-ներ կառավարվող սերվերների վրա |
| Playbook | Ansible-ի YAML ֆայլ, որը նկարագրում է թիրախ host-երի ցանկալի վիճակը |
| Handler | Ansible-ի հատուկ task, որը գործարկվում է play-ի վերջում` notify ստանալիս |
| Least Privilege | Նվազագույն արտոնությունների սկզբունք. յուրաքանչյուր սուբյեկտ ստանում է միայն աշխատանքի համար անհրաժեշտ նվազագույն իրավունքներն ու մուտքերը |
| IAM | Identity and Access Management. նույնականացման և մուտքի կառավարման համակարգ (օր.՝ AWS IAM) |
| RBAC | Role-Based Access Control. իրավունքների կառավարում ըստ դերերի/խմբերի |
| ABAC | Attribute-Based Access Control. մուտքի կառավարում ատրիբուտներով (ժամանակ, վայր, սարք) |
| JIT | Just-In-Time access. իրավունքների տրամադրում միայն անհրաժեշտ ժամանակահատվածի համար, ավտոմատ ավարտվող |
| Privilege Creep | Ժամանակի ընթացքում օգտատերերի իրավունքների «սողացող» կուտակում՝ առանց վերանայման |
| setcap | Linux-ի ֆայլի capability-ի կարգավորում. տալիս է ծրագրին մեկ նեղ հնարավորություն (օր.՝ cap_net_bind_service)՝ ամբողջ root-ի փոխարեն |
| SUID | Set User ID. permission բիթ, որի դեպքում executable ֆայլը գործարկվում է տիրոջ (սովորաբար root-ի) իրավունքներով. ցուցադրվում է `s`/`S`-ով owner-ի դիրքում |
| SGID | Set Group ID. ֆայլի վրա՝ գործարկում է խմբի իրավունքներով. գրացանակի վրա՝ ներսում ստեղծված ֆայլերը ժառանգում են գրացանակի խումբը |
| Sticky bit | Գրացանակի permission բիթ, որի դեպքում ֆայլը ջնջել/վերանվանել կարող է միայն ֆայլի տերը, գրացանակի տերը կամ root-ը (օր.՝ `/tmp`, `t` others-ի դիրքում) |
| umask | Ծրագրերի կողմից նոր ֆայլերի/գրացանակների ստեղծման ժամանակ «հանվող» permission բիթերի դիմակ (օր.՝ `022` → `644` ֆայլեր) |
| cgroup | Linux kernel-ի մեխանիզմ, որը պրոցեսները խմբավորում և սահմանափակում է ըստ ռեսուրսների (CPU, memory, I/O, PIDs) |
| namespace | Linux kernel-ի մեկուսացման մեխանիզմ, որը որոշում է, թե ինչ է տեսնում կոնտեյները (PID, network, mount) |
| UFW | Uncomplicated Firewall. Ubuntu-ի լռելյայն firewall-ի կառավարման frontend |
| nftables | Linux kernel-ի ժամանակակից firewall framework-ը, որը փոխարինել է iptables-ին |
| Netplan | Ubuntu-ի ցանցային կարգավորումների YAML-միջերես, որը կիրառում է systemd-networkd կամ NetworkManager |
| anchor / alias | YAML-ի կրկնությունը կրճատող մեխանիզմներ (`&name` սահմանում, `*name` հղում) |
| venv | Python-ի մեկուսացված virtual environment. ունի իր `site-packages`-ը և executable-ները |
| dangling image | Docker image առանց tag-ի (`<none>:<none>`), որը մնում է նույն tag-ով նոր build-ից հետո |
| bound method | Մեթոդ, որին instance-ը արդեն կապված է. կանչի ժամանակ `self`-ը լրացվում է ավտոմատ |
| descriptor | Python-ի մեխանիզմը, որով class-ի ատրիբուտը (օրինակ ֆունկցիան) կառավարում է իր մուտքը `__get__`-ով |
| `@classmethod` | Decorator, որի մեթոդը ստանում է `cls` (դասը, ոչ instance-ը). տիպիկ օգտագործումը այլընտրանքային constructor-ն է |
| `@staticmethod` | Decorator, որի մեթոդը չի ստանում ոչ `self`, ոչ `cls`. class-ի namespace-ում ապրող սովորական ֆունկցիա է |
| `@property` | Decorator, որը մեթոդը դարձնում է attribute-ի նման ընթերցվող հատկություն |
| name mangling | `__name` ձևի անդամի վերածումը `_ClassName__name`-ի. պաշտպանում է պատահական բախումից, ոչ մուտքից |
| type hint | Տիպի նշումը signature-ում (`def f(x: int) -> str`). runtime-ում չի ստուգվում, ստուգում են `mypy`/`pyright`-ը |
| DB-API | Python-ի տվյալների բազայի driver-ների ստանդարտ միջերեսը (PEP 249). `connect`, `cursor`, `execute` |
| paramstyle / placeholder | SQL-ում parameter-ի տեղը պահող նշանը (`?`, `%s`, `:name`, `@name`). driver-ից driver տարբերվում է |
| prepared statement | Նախապես parse/plan արված SQL. parameter-ները կցվում են առանձին ուղիով որպես արժեք, ոչ որպես SQL տեքստ |
| SQL injection | Հարձակում, երբ օգտատիրոջ մուտքը միանում է SQL տեքստին և մեկնաբանվում որպես հրաման |
| shell injection | Հարձակում, երբ մուտքը միանում է shell հրամանին (տիպիկ `shell=True`-ի հետ) և կատարվում որպես հրաման |
| XSS | Cross-Site Scripting. չչեզոքացված օգտատիրոջ տվյալը HTML-ում կատարվում է զոհի browser-ում որպես script |


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

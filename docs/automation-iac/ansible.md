# Ansible. Սերվերային կարգավորումների ավտոմատացում (IaC)

## Տեսություն

**Ansible**-ը **IaC** (**I**nfrastructure **a**s **C**ode, «ենթակառուցվածքը՝ որպես կոդ») գործիք է, որը կոդով նկարագրում է սերվերների ցանկալի վիճակը և ավտոմատ կիրառում այն բոլոր նշված մեքենաների վրա։ Դրա շնորհիվ կոնֆիգուրացիան, փաթեթների տեղադրումն ու ծառայությունների կառավարումը դառնում են կրկնվող (reproducible) և idempotent՝ միևնույն playbook-ը կարելի է անվտանգ գործարկել բազմիցս։ Idempotency հասկացությունը ավելի մանրամասն քննարկել ենք [Bash Scripting](../linux/bash-scripting.md) էջում։

### Ի՞նչ խնդիր է լուծում

Առանց ավտոմատացման յուրաքանչյուր սերվեր կարգավորվում է ձեռքով, և հաճախ սխալներ են առաջանում։ ինչ-որ քայլ մոռացվում է, ինչ-որ փոփոխություն արվում է միայն մեկ մեքենայի վրա։ Այսպես ստեղծվում է **config drift** (կոնֆիգուրացիաների շեղում)՝ նույնանման սերվերները ժամանակի ընթացքում դառնում են տարբեր։ Ansible-ն դա լուծում է «ցանկալի վիճակ» (desired state) մոտեցմամբ՝ նկարագրում ենք, թե ինչպիսին պետք է լինի համակարգը, իսկ Ansible-ը host-ը հասցնում է այդ վիճակին։

### Ինչպե՞ս է աշխատում

- **Control node (կառավարման հանգույց)**՝ այն մեքենան, որտեղ տեղադրված է Ansible-ը, և որտեղից գործարկվում են playbook-ները։ Սա ձեր աշխատանքային մեքենան է կամ CI runner-ը։
- **Managed nodes (կառավարվող հանգույցներ)**՝ այն սերվերները, որոնց վրա կիրառվում է ավտոմատացումը։ Դրանց մասին տե՛ս [Linux](../linux/index.md) բաժինը։
- **Agentless («առանց գործակալի»)**՝ կառավարվող սերվերների վրա ոչինչ նախապես տեղադրել պետք չէ՝ Ansible-ը միանում է SSH-ով, սերվերում գործարկում է կարճ Python ծրագիր (մոդուլը) և հավաքում արդյունքը։ Հետևաբար managed node-ում պետք է լինի Python։
- **YAML**՝ playbook-ները գրվում են [YAML](yaml.md) ձևաչափով, որը մարդու համար ընթեռնելի է, հեշտ է ստուգել (syntax check) և պահել Git-ում՝ ամեն փոփոխություն դառնում է տեսանելի։ Git-ի մասին՝ [Git և CI/CD](../git-ci-cd/index.md) բաժնում։
### Հիմնական բաղադրիչներ

**1. Inventory (ինվենտորիա)**՝ ֆայլ, որտեղ թվարկվում են host-ները և նրանց խմբերը՝ խմբերի շնորհիվ մեկ playbook-ը կարելի է կիրառել մի ամբողջ խմբի վրա։ Ընդունում է և՛ INI, և՛ YAML ձևաչափ՝ միացման մանրամասները նշվում են `ansible_` նախածանցով փոփոխականներով, օրինակ՝ `ansible_user`, `ansible_port`, `ansible_host`, `ansible_ssh_private_key_file`։ Օրինակ՝

```ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com ansible_user=deploy ansible_ssh_private_key_file=~/.ssh/deploy.key
```

**2. Modules (մոդուլներ)**՝ Ansible-ի «գործիքները», որոնք կատարում են կոնկրետ գործողություն՝ փաթեթի տեղադրում (`ansible.builtin.apt` կամ `yum`), ֆայլի պատճենում (`copy`), ձևանմուշից ֆայլի ստեղծում (`template`), ծառայության կառավարում (`service`, `systemd`), օգտատիրոջ ստեղծում (`user`) և այլն։ Մոդուլները idempotent են՝ եթե ցանկալի վիճակն արդեն կա, ելքում տեսնում ենք `ok`՝ ոչ թե `changed`։

**3. Playbook (փլեյբուք)**՝ YAML ֆայլ, որը նկարագրում է, թե *ինչ* պետք է արվի և *որտեղ* (որ host-ի կամ խմբի վրա)։ Օրինակ՝ Apache սերվերի տեղադրումն ու գործարկումը `webservers` խմբի host-ների վրա՝

```yaml
- name: Կարգավորել վեբ սերվերները
  hosts: webservers
  become: yes                  # բոլոր task-երը կատարվում են sudo-ով
  tasks:
    - name: Տեղադրել Apache-ն
      ansible.builtin.apt:
        name: apache2
        state: present

    - name: Գործարկել և միացնել Apache-ն
      ansible.builtin.service:
        name: apache2
        state: started
        enabled: yes
```
**4. Tasks (քայլեր)**՝ playbook-ի ներսի քայլերը, որոնք կատարվում են վերևից ներքև՝ հերթականությամբ։ Եթե task-ը ձախողվի մի host-ի վրա, Ansible-ը լռելյայն դադարեցնում է այդ host-ի հետագա task-երը (մյուս host-երը շարունակում են) և չի «հետ գլորում» արդեն կատարված փոփոխությունները՝ արդեն կատարված քայլերի արդյունքները մնում են տեղում։ Այս պահվածքը կարելի է փոխել՝ ավելացնելով `force_handlers` play-ում կամ `--force-handlers` դրոշակը գործարկելիս՝ handler-ները այդ դեպքում աշխատում են նաև ձախողված task-երի դեպքում։

**5. Handlers (հենդլերներ)**՝ հատուկ task-եր, որոնք աշխատում են միայն այն ժամանակ, երբ մեկ այլ task նրանց «ծանուցում» է (`notify`)։ Դրանց նպատակը՝ գործողություն կատարել միայն այն ժամանակ, երբ իսկապես փոփոխություն է եղել։ օրինակ՝ կոնֆիգը փոխվել է, ուրեմն՝ ծառայությունը վերագործարկիր։ Handlers-ի կանոնները՝

- Գործարկվում են play-ի վերջում՝ բոլոր task-երից հետո (կամ `meta: flush_handlers`-ով ստիպելու դեպքում)։
- Աշխատում են միայն այն ժամանակ, երբ notify-արած task-ը հաղորդել է «changed» (իրական փոփոխություն)։ Եթե task-ը ոչինչ չի փոխել, handler-ը չի աշխատում։
- Եթե նույն handler-ին notify անեն մի քանի task-եր, այն աշխատում է մեկ անգամ՝ ավելորդ վերագործարկումներից խուսափելու համար։
- Կատարման հերթականությունն ըստ `handlers` բաժնում սահմանված կարգի է, ոչ թե notify-ի կարգի։
- Խորհուրդ՝ handler-ի անունում փոփոխական մի դրեք, իսկ `listen`-ով կարելի է մի քանի handler «կապել» նույն notify-ի հետ։ Handler-ը չի կարող կանչել `import_role` կամ `include_role`։

Օրինակ՝ firewall-ի կանոնների անվտանգ կիրառման handler՝

```yaml
handlers:
  - name: apply_nftables
    ansible.builtin.shell: |
      nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf
```

Այս օրինակում կարևոր է երկու բան՝

- `|` — YAML-ի «literal block» նշանը, որը հաջորդող բոլոր տողերը դարձնում է մեկ ամբողջական տեքստ՝ նոր տողերը պահպանվում են։
- `&&` — bash-ի «և» օպերատորը, որը երկրորդ հրամանը կատարում է միայն այն դեպքում, երբ առաջինը հաջող է ավարտվել (վերադարձրել է 0)։ Սա «fail-safe» (անվտանգ ձախողում) սկզբունքն է՝ նախ ստուգում, հետո կիրառում։

Առանձին հրամանների իմաստը՝

- `nft -c -f ֆայլ` — «check» ռեժիմ, որը ստուգում է կանոնների ճշտությունը, բայց չի կիրառում։
- `nft -f ֆայլ` — կանոնների կիրառում ֆայլից (այստեղ՝ `&&`-ից հետո)։

Արդյունքը՝ եթե կոնֆիգում սինտաքսի սխալ կա, ապա `nft -c`-ն ձախողվում է, և `&&`-ը ընդհատում է կատարումը։ Սխալ կոնֆիգը երբեք չի կիրառվում, kernel-ում արդեն եղած հին կանոնները մնում են, SSH-ը՝ հասանելի։ Այսպիսի «նախ ստուգիր, հետո կիրառիր» ձևանմուշը պաշտպանում է ամենավատ իրավիճակից՝ սերվերի կապը կորցնելուց։ nftables-ի մասին՝ [Firewall](../linux/firewall.md) էջում։
**6. Փոփոխականներ (Variables)**՝ playbook-ը ճկուն դարձնելու գործիք նույն playbook-ը կարող է տարբեր արժեքներ ստանալ՝ կախված խմբից կամ host-ից։ Փոփոխականների տեղադրման գլխավոր վայրերը՝

- `group_vars/<group>.yml` — կիրառվում է խմբի բոլոր host-ների համար։
- `host_vars/<host>.yml` — կիրառվում է կոնկրետ host-ի համար և գերակշռում է group-ի արժեքին (host_vars > group_vars)։
- Play-ի ներսում՝ `vars:` բաժինը, որը հասանելի է play-ի բոլոր task-երին։
- Command line-ից՝ `-e` կամ `--extra-vars`, որը «override all» է՝ գերակշռում է մնացած բոլորին։ հարմար է CI/CD-ից հատուկ արժեք փոխանցելու համար։
#### group_vars-ը ավելի խորը ուսումնասիրություն

**group_vars**-ը Ansible-ի հատուկ դիրեկտորիա է, որը պարունակում է փոփոխականներ host-երի խմբերի համար։ Դրա ֆայլերի անունները համընկնում են inventory-ում սահմանված խմբերի անունների հետ, իսկ Ansible-ը դրանք ավտոմատ բեռնում է մինչև playbook-ը գործարկելը։ Այսպես ամբողջ կոնֆիգուրացիան առանձնանում է playbook-ից և դառնում է ավելի հեշտ կառավարվող：



##### Կառուցվածք
​
```
project/
├── group_vars/
│   ├── all.yml          # Բոլոր host-ների համար
│   ├── webservers.yml   # webservers խմբի համար
│   ├── dbservers.yml    # dbservers խմբի համար
│   └── production/      # production խումբը` որպես դիրեկտորիա
│       └── vars.yml
├── inventory/
│   └── hosts.ini
└── playbooks/
```
​
`all`-ը ներկառուցված խումբ է, որը միշտ պարունակում է բոլոր host-ներին, ուստի `group_vars/all.yml`-ը կիրառվում է ամեն ինչում, իսկ `production/` դիրեկտորիան թույլ է տալիս մեկ խմբի համար մի քանի ֆայլ պահել（`vars.yml`, `secrets.yml` և այլն）։ Խումբն ինքնին, իհարկե, պետք է սահմանված լինի inventory-ում՝
​
```ini
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11
​
[dbservers]
db1 ansible_host=192.168.1.20
db2 ansible_host=192.168.1.21
```
​
##### Փոփոխականների առաջնահերթություն
​
Ansible-ում փոփոխականները` բարձրից ցածր, ունեն գերակշռման այս հաջորդականությունը՝
​
1. `--extra-vars` (`-e`)՝  CLI-ից «override all» է և գերակշռում է ամեն ինչի, նաև `host_vars`-ին։
2. Play-ի/role-ի ներսում `vars:`-ով սահմանված փոփոխականները։
3. `host_vars`-ը` կոնկրետ host-ի համար։
4. `group_vars`-ը` խմբի համար։
5. `role defaults`-ը՝ ամենացածրը. ցանկացած արժեք կարելի է «վերագրանցել» վերոնշյալ քայլերից որևէ մեկով。
​
Փաստորեն `host_vars > group_vars`, play-ի `vars:`-ը երկուսից էլ բարձր է, իսկ `--extra-vars`-ը` բոլորից բարձր։ Inventory ֆայլում անմիջականորեն սահմանված փոփոխականները (օր.` `ansible_host`) մակարդակով համարժեք են `group_vars`-ին և կարող են override-վել play-ի `vars:`-ով।​
​
##### Ձևաչափեր
​
`group_vars`-ի բովանդակությունը կարելի է պահել երեք եղանակով. YAML ֆայլով, JSON ֆայլով կամ դիրեկտորիայով（ենթաֆայլերով՝ `vars.yml` և այլն）։ Բոլորը Ansible-ի համար իմաստով համարժեք են, տարբերվում է միայն կազմակերպման ձևը۔
​
##### Գործնական օրինակ
​
```yaml
# group_vars/all.yml
docker_arch_map:
  x86_64: amd64
  aarch64: arm64
  armv7l: armhf
​
# group_vars/development.yml
docker_image_tag: latest
docker_registry: dev.docker.io
​
# group_vars/production.yml
docker_image_tag: stable
docker_registry: prod.docker.io
```
​
##### Override playbook-ում
​
Play-ի task-ի ներսում `vars:`-ով կարող եք override անել `group_vars`-ի  արժեքները՝
​
```yaml
- name: Կարգավորել վեբ սերվերները
  hosts: webservers
  tasks:
    - name: Կարգավորել nginx
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      vars:
        http_port: 8080   # override group_vars/webservers.yml-ի  ​80-ը
```
​
##### Առավելություններ և խորհուրդներ
​
1. **Կենտրոնացված կառավարում**`  ամբողջ միջավայրի կոնֆիգուրացիան մեկ վայրում է և հեշտ է դիտել ու փոփոխել։
2. **Կրկնության խուսափում**`  DRY սկզբունքը պահվում է. նույն playbook-ը ծառայում է տարբեր միջավայրերի（development, production...）。
3. **Մասշտաբայնություն**`  ավելի մեծ ենթակառուցվածքներում ընդհանուր արժեքները բաժանվում են խմբերով, և թիմը աշխատում է փոփոխականների հստակ անուններով。
4. **Անվտանգություն**՝  գաղտնիքները（գաղտնաբառեր, tokens） միշտ պահեք [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)-ով, ոչ թե պարզ տեքստով.
 `secrets.yml` ֆայլերը կարող եք Vault-ով գաղտնագրել, իսկ մնացածը` հասանելի պահել。
5. **Փաստաթղթավորում**`  յուրաքանչյուր փոփոխականի համար մեկ մեկնաբանությամբ («comment») նշեք, թե ինչ է այն անում և որտեղ է օգտագործվում։

Օրինակ՝ `group_vars/webservers.yml` ֆայլում՝

```yaml
apache_max_clients: 900
```

Փոփոխականն օգտագործվում է ձևանմուշներում՝ `{{ apache_max_clients }}`։ Jinja2-ն ապահովում է նաև պայմաններ (`{% if %}`), ցիկլեր (`{% for %}`) և ֆիլտրեր, օրինակ՝ `{{ apache_max_clients | default(800) }}`։

**7. Templates (`*.j2`)**՝ `.j2`-ը Jinja2 ձևանմուշի (template) ֆայլի ընդլայնումն է՝ ստատիկ տեքստ + դինամիկ մասեր։ `template` մոդուլը կարդում է `.j2` ֆայլը, փոխարինում `{{ }}`-ի փոփոխականները, կատարում `{% %}` կառուցվածքները և ստեղծում վերջնական ֆայլը՝ նշված տեղում՝ առանց `.j2` ընդլայնման։ Ի տարբերություն `copy`-ի, որը ֆայլը պատճենում է «ինչպես կա», template-ը դինամիկ է և կարող է տարբերվել՝ կախված host-ի տվյալներից (օր.՝ ՕՀ-ի տարբերակից)։ Օրինակ՝
```jinja
{% for ip in firewall_allow_ips %}
    ip saddr {{ ip }} accept
{% endfor %}
```

**8. Roles (դերեր)**՝ playbook-ի վերօգտագործելի բլոկներ, որոնք խմբավորում են tasks-ը, handlers-ը, templates-ը և փոփոխականները։ Ստանդարտ կառուցվածքը՝

```
roles/webserver/
├── tasks/       # հիմնական քայլերը (main.yml)
├── handlers/    # handler-ները
├── templates/   # .j2 ձևանմուշները
├── files/       # ստատիկ ֆայլերը (copy-ի համար)
├── vars/        # role-ի փոփոխականները
├── defaults/    # լռելյայն արժեքները (հեշտ override)
└── meta/        # role-ի կախվածությունները
```

Playbook-ում role-ը կանչվում է `roles:` բաժնով, և role-ի `tasks/main.yml`-ը ավտոմատ կատարվում է play-ի մեջ՝ tasks-երի ու handlers-երի հետ միասին։ Այդպես նույն role-ը կարելի է օգտագործել տարբեր playbook-ներում և նույնիսկ տարբեր նախագծերում (օր.՝ Ansible Galaxy-ի միջոցով)։

**9. Check mode (`--check`) և `check_mode: false`**

**Check mode** (նաև հայտնի որպես `dry run` «չոր գործարկում») Ansible-ի ռեժիմն է, որը ցույց է տալիս, թե ինչ փոփոխություններ կկատարվեն, բայց դրանք իրականում չի կիրառում։ Գործարկվում է `--check` դրոշակով՝

```bash
ansible-playbook playbook.yml --check
```

Check mode-ին աջակցող մոդուլները հաղորդում են «ինչ կփոխվեր» (what would change)՝ առանց համակարգին դիպչելու։ Սա օգտակար է playbook-ը գրելիս, փոխելիս և CI-ում թեստելիս։ Բայց կա կարևոր սահմանափակում. check mode-ը չի «ստեղծում» բացակայող ռեսուրսները, և apt-ը չի կարող «տեսնել» նոր repository-ից փաթեթների:

#### Խնդիրը Docker repository-ի դեպքում

Երբ playbook-ը նոր `apt` repository է ավելացնում (օրինակ Docker-ը) և ապա փաթեթ է տեղադրում (օր.՝ `docker-ce`), `--check`-ը ինքնուրույն չի աշխատում։ Պատճառը՝ check mode-ում GPG keyring-ի դիրեկտորիան «չի ստեղծվում», GPG key-ը «չի ներբեռնվում», repo-ի ֆայլը «չի գրվում», apt cache-ը «չի թարմացվում»։ Այդ ռեսուրսներից որևէ մեկը բացակայում է՝ `apt`-ը «չի տեսնում» `docker-ce`-ն, ուստի install task-ը ձախողվում է, նույնիսկ `--check`-ի ժամանակ.

```
fatal: => "No package matching 'docker-ce' is available"
```

Լուծումը. նախապատրաստական (prerequisite) task-երը նշվում են `check_mode: false`-ով, որը հրահանգում է Ansible-ին, որ այս task-երը միշտ աշխատեն իրական (normal) ռեժիմում, նույնիսկ `--check`-ի դեպքում։ [Ansible-ի պաշտոնական փաստաթղթերը](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html) հաստատում են, որ `check_mode: false`-ը task-ը «force» է անում՝ ամեն դեպքում փոփոխություններ մտցնելու համակարգում։ Օրինակ՝ Docker-ի repo-ի նախապատրաստական task-երը՝

```yaml
- name: Create directory for the Docker GPG keyring
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'
  check_mode: false

- name: Download Docker GPG key (ASCII-armored)
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
    force: no
  check_mode: false

- name: Add Docker apt repository
  ansible.builtin.copy:
    dest: /etc/apt/sources.list.d/docker.list
    content: 'deb [arch={{ docker_arch }} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} {{ docker_channel }}'
    mode: '0644'
  register: docker_repo_file
  check_mode: false

- name: Update apt cache with Docker repo included
  ansible.builtin.apt:
    update_cache: yes
  when: docker_repo_file.changed
  check_mode: false
```

Յուրաքանչյուր task-ի պատճառը՝

- **Create directory**`  keyring-ի դիրեկտորիան պետք է իսկապես գոյություն ունենա, որ key-ն ու repo-ն ավելացվեն`  idempotent`  ցածր ռիսկ (անվտանգ է կրկնել)։
- **Download Docker GPG key**`  apt-ին անհրաժեշտ public key`  `force: no`-ն ապահովում է idempotency`  ցածր ռիսկ, որովհետև public key է`  ոչ գաղտնիք։
- **Add Docker apt repository**`  ամենակարևոր prerequisite-ը`  առանց repo-ի apt-ը «չի տեսնի» `docker-ce`-ն`  idempotent`  copy-ը փոխում է միայն տարբեր բովանդակության դեպքում, ցածր ռիսկ։
- **Update apt cache**`  ակտիվացնում է repo-ն`  կատարվում է միայն repo-ն փոխվելիս (`when`), idempotent`  ցածր ռիսկ (apt update-ը անվտանգ է կրկնել)։

#### Ինչ կպատահի առանց `check_mode: false`

Նույն task-երն առանց `check_mode: false`-ի `--check`-ի ժամանակ «չեն ստեղծի» ոչ դիրեկտորիան, ոչ key-ը, ոչ repo-ն, ոչ cache-ը։ Հետևանքը՝ `docker_repo_file.changed`-ը կմնա `false` (task-ը check mode-ում ոչինչ չի փոխել), apt update task-ը կնշվի `skipped` (`when`-ը չի բավարարվի), իսկ install task-ը կձախողվի. apt-ը չի գտնի `docker-ce`-ն, և `--check`-ը կվերադարձնի «No package matching 'docker-ce' is available» սխալը՝ թեև իրականում ոչինչ չի փոխվել.

#### Համեմատական աղյուսակ

| Task | `check_mode: false` | Պատճառ | Ռիսկ |
| --- | --- | --- | --- |
| Create directory | Այո | Prerequisite, idempotent | Ցածր |
| Download GPG key | Այո | Prerequisite, public key | Ցածր |
| Add Docker repo | Այո | Կարևորագույն՝ apt-ի համար | Ցածր (idempotent) |
| Update apt cache | Այո | Ակտիվացնում է repo-ն | Ցածր (idempotent) |
| Install Docker | Ոչ | Իրական փոփոխություն | Բարձր |
| Start Docker | Ոչ | Իրական փոփոխություն (service) | Բարձր |
| Add user to group | Ոչ | Միայն ցուցադրում (`when: not ansible_check_mode`) | Միջին |

Իրական փոփոխություն կատարող task-երը (փաթեթի տեղադրում, ծառայության գործարկում, օգտատիրոջ փոփոխում) պետք է մնան check mode-ում, որ `--check`-ը ցույց տա «ինչ կփոխվի»՝ իրականում չփոխելով։ Դրանց հետ թիմը հաճախ օգտագործում է `ansible_check_mode` մոգական փոփոխականը, օրինակ `when: not ansible_check_mode`՝ task-ը check mode-ում ամբողջությամբ բաց թողնելու համար, քանի որ այն իմաստ չունի առանց իրական փոփոխության.

#### Այլընտրանքային մոտեցումներ

1. **Pre-flight ստուգում `stat`-ով՝  նախ ստուգել ռեսուրսի գոյությունը, ապա պայմանականորեն ավելացնել  բայց check mode-ում պայմանները հիմնվում են ոչ-իրական արժեքների վրա և կարող են սխալ արդյունք տալ` ավելի բարդ։
2. **Առանձին playbook/role-ներ**  նախապատրաստական և install-ի մասերը բաժանել և `--check`-ը գործարկել միայն install-ի մասի վրա՝  ավելի դժվար կառավարվող, բայց ավելի «մաքուր» dry run`  հարմար է մեծ playbook-ների համար։

Հիմնական կանոնը պարզ է՝  `check_mode: false`-ը գործածեք միայն prerequisite task-երի համար, որոնք idempotent են և ռեսուրսներ են «ստեղծում»՝ առանց որի check mode-ն ի վիճակի չի լինի ստուգելու մնացածը։ Սա տեղեկացված որոշում է, և պետք է մեկնաբանությամբ (comment) փաստաթղթավորվի կոդում, որ թիմի անդամը հասկանա, թե ինչու է այդ task-ը «dry run»-ից դուրս աշխատում.
## Հիմնական հրամաններ

| Հրաման | Նպատակ |
| --- | --- |
| `ansible --version` | Ցույց է տալիս Ansible-ի տարբերակը |
| `ansible-inventory --list` | Ցույց է տալիս inventory-ի ամբողջ կառուցվածքը (JSON) |
| `ansible <host> -m ping` | Ստուգում է կապը host-ի հետ (SSH և Python) |
| `ansible <host> -m setup` | Հավաքում է host-ի տվյալները (`ansible_facts`) |
| `ansible-playbook playbook.yml` | Գործարկում է playbook-ը |
| `ansible-playbook playbook.yml --syntax-check` | Ստուգում է YAML սինտաքսը՝ առանց կիրառելու |
| `ansible-playbook playbook.yml --check` | Չոր գործարկում (dry-run)՝ ցույց է տալիս, թե ինչ կփոխվի |
| `ansible-playbook playbook.yml --force-handlers` | Handler-ները գործարկում է նույնիսկ ձախողված task-երի դեպքում |
| `ansible-playbook playbook.yml -e var=val` | Փոխանցում է փոփոխական՝ գերակշռելով մնացածին |
## Փորձարկում (Lab)

Միջավայրը՝ Ubuntu 22.04 կամ 24.04 VM՝ root արտոնություններով (կամ Docker container)։ Քայլերը վերարտադրելի են մեկ host-ով՝ `localhost`։

### Քայլ 1. Տեղադրել Ansible-ը

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

Սպասվող արդյունք՝ Ansible-ի տարբերակի մասին հաղորդագրություն։ Զգուշացում՝ Ubuntu-ի պահոցի տարբերակը կարող է հին լինել, բայց այս lab-ի համար այն բավարար է։

### Քայլ 2. Նախագծի կառուցվածք և inventory

```bash
mkdir -p ~/ansible-lab/templates ~/ansible-lab/group_vars
cd ~/ansible-lab
```

Ստեղծեք `ansible.cfg`

```ini
[defaults]
inventory = hosts.ini
host_key_checking = False
interpreter_python = auto_silent
```

`interpreter_python = auto_silent`-ը թույլ է տալիս Ansible-ին ինքնուրույն գտնել Python-ը՝ առանց նախազգուշացումների, իսկ `host_key_checking = False`-ը՝ բաց թողնել SSH-ի առաջին միացման հարցումը (ընդունելի է միայն վստահելի ներքին միջավայրում)։ Ստեղծեք `hosts.ini`

```ini
[lab]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
```

Ստուգեք inventory-ն և կապը՝

```bash
ansible-inventory --list
ansible lab -m ping
```

Սպասվող արդյունք՝ `pong` պատասխան, որը ապացուցում է, որ Ansible-ը կարող է աշխատել host-ի հետ։
### Քայլ 3. Փոփոխական և ձևանմուշ

Ստեղծեք `group_vars/lab.yml`

```yaml
site_title: Իմ Ansible լաբը
```

Ստեղծեք `templates/index.html.j2`

```jinja
<html>
  <head><title>{{ site_title }}</title></head>
  <body>
    <h1>{{ site_title }}</h1>
    <p>{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}</p>
  </body>
</html>
```

`{{ site_title }}`-ը՝ group_vars-ից վերցված փոփոխական, իսկ `ansible_facts['distribution']`-ը՝ host-ի մասին `gather_facts`-ով հավաքված տվյալ (ՕՀ-ի անունը)։

### Քայլ 4. Playbook՝ handler-ով

Ստեղծեք `playbook.yml`

```yaml
---
- name: Կարգավորել նմուշային վեբ էջ
  hosts: lab
  become: yes
  gather_facts: yes
  tasks:
    - name: Տեղադրել nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Գրել index.html-ը ձևանմուշից
      ansible.builtin.template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
        mode: '0644'
      notify: reload_nginx

    - name: Համոզվել, որ nginx-ը աշխատում է
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: reload_nginx
      ansible.builtin.shell: |
        nginx -t && systemctl reload nginx
```

### Քայլ 5. Առաջին գործարկում

```bash
ansible-playbook playbook.yml
```

Սպասվող արդյունք՝ ելքում կա «RUNNING HANDLER [reload_nginx]» տողը, քանի որ ձևանմուշի task-ը «changed» է հաղորդել, handler-ի ներսում սկզբում `nginx -t`-ն է ստուգում կոնֆիգը, ապա՝ reload-ը։

### Քայլ 6. Երկրորդ գործարկում՝ idempotency

```bash
ansible-playbook playbook.yml
```

Սպասվող արդյունք՝ բոլոր task-երը «ok» են (ոչ «changed»), և «RUNNING HANDLER» տողը չկա, որովհետև ոչինչ չի փոխվել։ Սա idempotency-ի և «handler-ն աշխատում է միայն փոփոխության դեպքում» կանոնի ցուցադրումն է։
### Քայլ 7. Փոփոխականների առաջնահերթություն

```bash
ansible-playbook playbook.yml -e site_title="Վերագրված տիտղոս"
```

Սպասվող արդյունք՝ ձևանմուշը նորից «changed» է (տիտղոսը փոխվել է), և handler-ը նորից աշխատում է՝ `-e`-ն գերակշռեց `group_vars`-ի արժեքին։ Սա հաստատում է, որ CLI-ի փոփոխականն ամենաբարձր առաջնահերթությունն ունի։

### Քայլ 8. Failure-ի վարքագիծ handler-ի հետ

Ստեղծեք `fail-demo.yml`

```yaml
---
- name: Handler-ի և failure-ի վարքագիծ
  hosts: lab
  gather_facts: no
  tasks:
    - name: notify անող task
      ansible.builtin.command: /bin/true
      changed_when: true
      notify: demo_hook

    - name: Կանխամտածված ձախողվող task
      ansible.builtin.command: /bin/false

  handlers:
    - name: demo_hook
      ansible.builtin.debug:
        msg: "Ես handler եմ"
```

Գործարկեք առանց դրոշակի՝

```bash
ansible-playbook fail-demo.yml
```

Սպասվող արդյունք՝ երկրորդ task-ը FAILED է, playbook-ը դադարում է, և «RUNNING HANDLER» տողը չկա։ handler-ը չի աշխատում, քանի որ host-ը ձախողված է համարվում։ Հիմա գործարկեք դրոշակով՝

```bash
ansible-playbook fail-demo.yml --force-handlers
```

Սպասվող արդյունք՝ չնայած ձախողմանը՝ «RUNNING HANDLER [demo_hook]» տողը հայտնվում է՝ այդպես `force_handlers`-ը փոխում է լռելյայն պահվածքը։ Երկու դեպքում էլ արդեն կատարված փոփոխությունները մնում են՝ Ansible-ը դրանք չի «հետ գլորում»։

!!! tip "Best Practices"
    - Production-ից առաջ playbook-ը միշտ ստուգեք `--syntax-check` և `--check`։
    - Firewall-ի/SSH-ի կարգավորումներ պարունակող handler-ներում օգտագործեք «նախ ստուգիր, հետո կիրառիր» ձևանմուշը՝ «`nft -c ... && nft -f ...`»։
    - Գաղտնիքները (SSH keys, tokens) երբեք պարզ տեքստով մի պահեք, դրանք պահեք `ansible-vault`-ով։
    - Սկզբում՝ staging, ապա՝ production, իսկ playbook-ի յուրաքանչյուր փոփոխություն՝ պահեք Git-ում։
## Իրական DevOps իրավիճակ

### Ախտանիշ

Թիմը production սերվերում firewall-ի (nftables) կանոնները տեղակայում է Ansible playbook-ով, որը կանոնները ստանում է `.j2` ձևանմուշից։ Deploy-ի ժամանակ ելքում երևում է՝

```
RUNNING HANDLER [apply_nftables]
fatal: [web1]: FAILED! => nft: syntax error ...
```

Playbook-ն ավարտվում է `failed=1`-ով, բայց սերվերը շարունակում է աշխատել, SSH-ը չի կտրվում։ Հարցը՝ ինչո՞ւ է playbook-ը ձախողվում, բայց կապը չի կորչում։

### Ախտորոշում

1. Գործարկեք playbook-ը `-v` դրոշակով՝ տեսնելու, թե կոնկրետ որ handler-ն է ձախողվել՝
   ```bash
   ansible-playbook deploy-firewall.yml -v
   ```
2. Սերվերի վրա ստուգեք, թե արդյոք սկավառակի վրա եղած կոնֆիգը ճիշտ է՝
   ```bash
   sudo nft -c -f /etc/nftables.conf
   ```
   Սխալի հաղորդագրությունը ցույց է տալիս տողն ու պատճառը։
3. Ստուգեք, թե որ կանոններն են այդ պահին ակտիվ՝
   ```bash
   sudo nft list ruleset
   ```
   Երևում է, որ ակտիվ են հին՝ վերջին հաջող կիրառման կանոնները, քանի որ վատ կոնֆիգը երբեք չի կիրառվել «`&&`»-ի շնորհիվ։

### Լուծում

1. Ուղղեք `.j2` ձևանմուշի սխալը (օր՝ բաց թողած փակագիծ)։
2. Գործարկեք ստուգումն ու չոր գործարկումը՝
   ```bash
   ansible-playbook deploy-firewall.yml --syntax-check
   ansible-playbook deploy-firewall.yml --check
   ```
3. Գործարկեք playbook-ը՝ այժմ `nft -c`-ն անցնում է, և `nft -f`-ը կիրառում է նոր կանոնները՝
   ```bash
   ansible-playbook deploy-firewall.yml
   ```
4. Ստուգեք արդյունքը՝
   ```bash
   sudo nft list ruleset
   ```
5. Կարևոր նրբություն՝ `/etc/nftables.conf`-ը սկավառակի վրա «սխալ» է հասցվել, իսկ boot-ի ժամանակ `nftables.service`-ը հենց այդ ֆայլն է բեռնում՝ ուստի սխալը պետք է շատ արագ ուղղել կամ վերականգնել աշխատող կոնֆիգը, հակառակ դեպքում reboot-ից հետո firewall-ը կարող է ձախողվել։ Ահա թե ինչու «ստուգիր՝ հետո կիրառիր» ձևը փրկում է փակվելուց, բայց ձևանմուշը միայն փոփոխելիս պետք է զգույշ լինել՝ մինչև reboot-ը։

### Կանխարգելում

- Handler-ներում միշտ պահպանեք fail-safe «`nft -c ... && nft -f ...`» ձևը՝ վատ կոնֆիգը երբեք «չի կիրառվի»։
- Deploy-ից առաջ՝ `--syntax-check`, `--check` և staging-ում փորձարկում։
- `force_handlers`-ը օգտագործեք զգուշությամբ՝ միայն այն դեպքում, երբ handler-ը անվտանգ է՝ աշխատելու ձախողված task-երից հետո։
- Ունեցեք հայտնի-աշխատող կոնֆիգի backup (կամ պահեք այն Git-ում)՝ reboot-ից հետո արագ վերականգնման համար։
- Ախտորոշման հրամաններով runbook-ը պահեք թիմի համար մատչելի տեղում։
## Հարցազրույցի հարցեր և պատասխաններ

### Ի՞նչ է handler-ը Ansible-ում, և ե՞րբ է այն գործարկվում (mid-level)

Handler-ը հատուկ task է, որը գործարկվում է միայն այն ժամանակ, երբ մեկ այլ task նրան «ծանուցում» է (`notify`), և միայն այն ժամանակ, երբ այդ task-ը «changed» է հաղորդել։ Handler-ները գործարկվում են play-ի վերջում՝ բոլոր task-երից հետո, իրենց հայտարարման հերթականությամբ, և յուրաքանչյուրը՝ մեկ անգամ, նույնիսկ եթե մի քանի task է նրան «ծանուցել»։ Եթե play-ի ընթացքում task-ը ձախողվի, այդ host-ի handler-ները լռելյայն չեն աշխատում՝ վարքագիծը փոխվում է `force_handlers`-ով կամ `--force-handlers`-ով։

### Ինչու՞ է «nft -c -f ... && nft -f ...» ձևանմուշը կենսական firewall-ի deploy-ի ժամանակ (senior-level)

`nft -c`-ն «check» ռեժիմն է՝ ստուգում է կոնֆիգի վավերականությունը, բայց չի կիրառում, իսկ `&&`-ը bash-ի օպերատոր է, որը հաջորդ հրամանը կատարում է միայն նախորդի հաջող ավարտից հետո։ Եթե կոնֆիգում սինտաքսի սխալ կա, `nft -c`-ն ձախողվում է, և `nft -f`-ը (կիրառումը) երբեք չի կատարվում՝ արդյունքում kernel-ում մնում են հին, աշխատող կանոնները, և SSH-ը հասանելի է մնում։ Սա «fail-safe» մոտեցում է՝ playbook-ը ձախողվում է, արդեն կատարված փոփոխությունները մնում են, բայց firewall-ը չի «կոտրվում»։ Ավագ ինժեները նաև կնշի, որ `/etc/nftables.conf`-ը սկավառակի վրա սխալ է հասցվել, ուստի այն պետք է արագ ուղղել՝ մինչև reboot-ը, և որ `--check`-ը չի կիրառում և չի կատարում handler-ները, ուստի idempotent state-ը հաստատում է միայն լրիվ գործարկումը։

### Ե՞րբ օգտագործել `group_vars`, և ինչո՞վ է այն տարբերվում `host_vars`-ից（mid-level）

`group_vars`-ը օգտագործում եք, երբ միևնույն արժեքը պետք է կիրառվի խմբի բոլոր host-ների համար. օրինակ՝ նույն `http_port`, `timezone` կամ `docker_registry`-ը development-ի բոլոր սերվերներին。 `host_vars`-ը՝ կոնկրետ մեկ host-ի բացառիկ արժեքի համար և միշտ գերակշռում է `group_vars`-ին（`host_vars > group_vars`）： Օրինակ. եթե `group_vars/webservers.yml`-ում `http_port: 80` եք դրել, իսկ `host_vars/web1.yml`-ում` `8080՝, ապա web1 host-ը կստանա `8080`, մնացած webservers-ները՝ `80`։ Ընդհանրապես՝ `group_vars`-ը՝ խմբի ընդհանուրը, `host_vars`-ը` մասնավորը.,

### Ինչու՞ է պետք `check_mode: false` repository ավելացնող playbook-ում, և ի՞նչ ռիսկ է այն թաքցնում (senior-level)

`apt`-ը check mode-ում չի կարող «resolve» անել նոր repo-ից փաթեթ, քանի որ repo-ի ֆայլը, GPG key-ը և թարմացված cache-ը գոյություն չունեն. դրանցից որևէ մեկի բացակայության դեպքում `docker-ce`-ն չի երևում apt-ին, և install task-ը ձախողվում է «No package matching 'docker-ce' is available» սխալով՝ չնայած `--check`-ը «չոր» է։ Այդ պատճառով նախապատրաստական task-երը (directory, key, repo, cache) դրվում են `check_mode: false`-ով, որը ստիպում է դրանք գործարկվել իրական (normal) ռեժիմում. դրանք idempotent են (անվտանգ է կրկնել), ուստի ռիսկը ցածր է։ Ավագ ինժեները կընդգծի, որ սա «dry run»-ից հրաժարում է 4 task-ի համար. `--check`-ով իրական փոփոխություններ են կատարվում, հետևաբար պետք է մեկնաբանությամբ փաստաթղթավորվի կոդում, և որ `--check`-ը ճշգրիտ ցույց է տալիս «ինչ կփոխվի» միայն կիրառական task-երի վրա (install, service)` որոնք պետք է մնան check mode-ում` օրինակ `when: not ansible_check_mode`-ով։
## Ինքնաստուգում

1. Ի՞նչ է նշանակում «agentless»-ը, և ինչպե՞ս է Ansible-ը միանում կառավարվող host-երին։
2. Ի՞նչ է inventory-ը, և ինչ դեր ունեն խմբերն ու `ansible_user`, `ansible_ssh_private_key_file` փոփոխականները։
3. Ե՞րբ է գործարկվում handler-ը, և ինչո՞ւ կարող է այն աշխատել միայն մեկ անգամ՝ չնայած մի քանի task-ի notify-ին։
4. Ի՞նչ է idempotency-ն, և ինչպե՞ս է այն երևում երկրորդ գործարկման ելքում (`ok` ընդդեմ `changed`)։
5. Ի՞նչ է `.j2` ֆայլը, և ինչո՞վ է `template` մոդուլը տարբերվում `copy`-ից։
6. Ինչպե՞ս ապահովել handler-ի աշխատանքը, երբ play-ի ինչ-որ task ձախողվել է (`--force-handlers` / `force_handlers`)։
7. Ինչո՞ւ է «`nft -c ... && nft -f ...`»-ը fail-safe պաշտպանություն, և ի՞նչ է մնում ուժի մեջ՝ սխալ կոնֆիգի դեպքում։

8. Ի՞նչ է `group_vars`-ը, ինչ ձևաչափերով կարելի է պահել, և ինչո՞ւ է `host_vars`-ը գերակշռում դրան.
9. Ի՞նչ է `--check`-ը, և ինչո՞ւ repository ավելացնող playbook-ում նախապատրաստական task-երը «ստեղծվում» են `check_mode: false`-ով՝ չնայած `--check`-ի «չոր» լինելուն։



## Հաջորդ քայլեր

Շարունակեք [Git և CI/CD](../git-ci-cd/index.md) բաժնով՝ playbook-ները CI pipeline-ում գործարկելու համար, իսկ firewall-ի ավտոմատացման պրակտիկան ամրապնդելու համար՝ [Firewall](../linux/firewall.md) և [Linux](../linux/index.md) էջերով։ Թեմաներն ավելի խոր ուսումնասիրելուց առաջ լավ գաղափար է ամրապնդել [YAML](yaml.md) էջը, քանի որ playbook-ների և ձևանմուշների ամբողջ սինտաքսը YAML-ի վրա է հիմնված։ Յուրաքանչյուր playbook-ի փոփոխություն `deploy`-ից առաջ ստուգեք `--syntax-check` և `--check` դրոշակներով, իսկ repository ավելացնող role-երում `check_mode: false`-ի օգտագործումը արեք միայն idempotent նախապատրաստական task-երի համար՝ ըստ այս էջի օրինակի։ Հաջորդ խորացումը՝ roles-ի, `ansible-vault`-ի և Jinja2-ի՝ ավելի բարդ ձևանմուշների ուսումնասիրությունը, ապա՝ Terraform-ը cloud-ի IaC-ի համար։
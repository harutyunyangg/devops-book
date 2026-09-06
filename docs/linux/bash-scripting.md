# Linux Bash Scripting

## Տեսություն

**Bash** (Bourne Again SHell) -ը Linux-ի (և macOS-ի) ամենատարածված command-line ինտերպրետատորն է (shell)։ **Bash script**-ը պարզապես տեքստային ֆայլ է, որը պարունակում է հրամանների հաջորդականություն, որոնք կարող են գործարկվել միասին ավտոմատացնելու կրկնվող գործողությունները.

Bash-ը DevOps-ի հիմնական գործիքներից է. նախքան Ansible-ի, Terraform-ի կամ CI/CD pipeline-ների անցումը մենք հաճախ ամեն ինչ սկսում ենք սկրիպտից։ Այն թույլ է տալիս.

- **ավտոմատացնել** կրկնվող գործողությունները (backup, մոնիտորինգ),
- **կառավարել համակարգը** (ծրագրերի տեղադրում, կարգավորումներ, օգտատերեր),
- **ինտեգրել տարբեր Linux հրամաններ** մեկ աշխատանքային հոսքում,
- աշխատել **ավելի արագ**, քան GUI-ով, հատկապես սերվերներում.

### Սկրիպտի կառուցվածքն ու գործարկումը

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

### Հիմնական սինտաքս

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

#### Ավտոմատ արտահանում `set -a` / `set +a`

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

### Process execution model. subshell vs source

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

### Exit codes և `set -e` / `set -u`

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

### Իդեմպոտենտություն (Idempotency)

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

## Հիմնական հրամաններ

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

## Փորձարկում (Lab)

Այս փորձերը գործարկիր սովորական Linux VM-ում կամ development container-ում ոչ production-ում, և առանց production credentials-ի.
### Lab 1. subshell vs source

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

### Lab 2: Exit codes և `set -euo pipefail`

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

### Lab 3: Idempotent setup script

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

### Lab 4: Ավտոմատ արտահանում `set -a` / `set +a`

Ստուգիր, թե որ փոփոխականներն է տեսնում child shell-ը `env`-ի միջոցով.

```bash
set -a                # միացնում ենք ավտո-արտահանումը
MY_AUTO="hello"
set +a                # անջատում ենք
MY_PLAIN="bye"        # սա չի արտահանվելու

sh -c 'echo "MY_AUTO=$MY_AUTO MY_PLAIN=${MY_PLAIN:-EMPTY}"'
```

Սպասվող արդյունք. `MY_AUTO=hello MY_PLAIN=EMPTY` (`set -a`-ի տակ սահմանված փոփոխականը հասանելի է child-ին, իսկ `set +a`-ից հետոյինը՝ ոչ)։ Ստուգիր նաև, որ `set -a`-ի միացված վիճակում ընթացիկ օպցիաները ցույց տվող `$-`-ում հայտնվում է `a`-ն.

## Իրական DevOps իրավիճակ

### Ախտանիշ

Production ինժեները գրել է backup/log-rotation script, որը պետք է ամեն գիշեր cron-ից գնա `/var/log/application`-ը, արխիվացնի log ֆայլերը և փոխանցի `/backup/logs`-ին։ Փորձարկման ժամանակ նկատվում է արխիվացված ֆայլերը հայտնվում են սխալ directory-ում, և script-ը չի գտնում log ֆայլերը, քանի որ աշխատում է սխալ directory-ում.

### Ախտորոշում

1. `ps -f`-ով և `echo $$`, `echo $SHLVL`-ով ստուգիր script-ն աշխատում է subshell-ում, իսկ `cd`-ն script-ի ավարտից հետո կորչում է,
2. Ստուգիր script-ը կախված է ընթացիկ working directory-ից (relative paths, մենակ `cd`),
3. Հաշվի առ cron-ը script-ը գործարկում է minimal environment-ով առանց `.bashrc`, բայց interactive shell-ում դու PATH-ն ու relative path-ները սովոր ես տեսնել.

### Լուծում

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

### Կանխարգելում

- Բոլոր scripts-ներում օգտագործիր absolute paths. անկախ գործարկման միջավայրից (cron, systemd, CI/CD),
- Սկզբում `set -euo pipefail`,
- Դիր idempotent միջոց `mkdir -p` և այլն,
- Cron/systemd-ի համար unit-ում նշիր WorkingDirectory-ը, script-ի աշխատանքը գրանցիր syslog-ում.

## Հարցազրույցի հարցեր և պատասխաններ

### Ինչու՞ է `./script.sh`-ով գործարկելիս directory-ի փոփոխությունը չպահպանվում, և ինչպե՞ս լուծել

`./script.sh`-ն նոր subshell (child process) է ստեղծում, որտեղ կատարվում է `cd`-ը. subshell-ի ավարտից հետո փոփոխությունները կորչում են, և ծնող shell-ը մնում է նախկին directory-ում. Պահպանելու համար `source script.sh` կամ `. script.sh`-ը որը կատարում է ընթացիկ shell-ում. Production-ում սակայն, նախընտրում են absolute paths-ով գործարկման եղանակից անկախ.

### Ինչպե՞ս ստուգել script-ը subshell-ում է, թե source-ով

`$$`-ը համեմատիր `$PPID`-ի հետ. source-ով `$$`-ը ծնողի PID-ն է նույն shell-ը. subshell-ում `$$`-ը նոր PID է, և `$SHLVL`-ը 1-ով մեծ է. Բացի դա `ps -f`-ի PID/PPID/CMD սյունակներն են ցույց տալիս որտեղ է աշխատում script-ը.

### Ի՞նչ ռիսկեր կան subshell-ի և source-ի հետ ավտոմատացված միջավայրերում

Ենթադրությունները որ script-ն աշխատում է որոշակի directory-ում կամ environment-ով. cron-ը minimal environment-ով է առանց `.bashrc`-ի. source-ը կարող է անսպասելիորեն փոխել CI/CD pipeline-ի shell-ի միջավայրը. Լուծում absolute paths, PATH-ի սահմանում script-ի սկզբում, `set -euo pipefail`, WorkingDirectory unit-ում, logging, և pipeline-ում subshell-ի նախընտրում եթե script-ը չպետք է ազդի shell-ի վրա.

### Ի՞նչ է idempotency, և ինչո՞ւ այն կարևոր DevOps-ում

Գործողությունը idempotent է, եթե կրկնակի կատարումը տալիս է նույն արդյունքը առանց կողմնակի ազդեցության. Այն թույլ է տալիս անվտանգորեն կրկնել cron-ը, CI/CD-ն, recovery-ն առանց միջավայրը փչացնելու. ոչ idempotent-ը վտանգավոր. Օրինակ `mkdir -p`-ի և `mkdir`-ի տարբերությունը.

### Ի՞նչ է անում `set -a`-ն, և ի՞նչ ռիսկ ունի

`set -a`-ն (այլ `set -o allexport`) ստեղծված կամ փոփոխված բոլոր փոփոխականներն ու ֆունկցիաները ավտոմատ կերպով արտահանում է child process-ներին, այնպես որ պետք չէ յուրաքանչյուրին առանձին `export` գրել։ Ռիսկն այն է, որ գաղտնիքները (tokens, passwords) ևս արտահանվում են բոլոր child process-ներին, իսկ պատահական փոփոխականներն արտահոսում են։ Դրա պատճառով օգտագործումից հետո անպայման անջատիր այն `set +a`-ով, և ցանկալի է` օգտագործել այն միայն փոքր, խմբավորված scope-ում։

## Ինքնաստուգում

1. Ի՞նչ է shebang-ը և ինչո՞ւ այն դրվում է սկրիպտի առաջին տողում
2. Ի՞նչ տարբերություն է subshell-ի և source-ի միջև. ե՞րբ կկորցնես `cd`-ի փոփոխությունը
3. Ի՞նչ է ցույց տալիս `$?`-ը, և ե՞րբ է «ոչ զրոյական» exit code-ը համարվում սխալ
4. Ի՞նչ է անում `set -euo pipefail`-ը և ինչո՞ւ է այն կարևոր
5. Ինչպե՞ս կդարձնես directory-ի ստեղծումը idempotent
6. Ի՞նչ է անում `set -a`-ն, և ինչո՞ւ պետք է ավարտից հետո `set +a`-ով անջատել

## Հաջորդ քայլեր

Շարունակիր filesystem permissions (1.4) կամ systemd (1.5) թեմայով. կիրառիր ձեր գրած scripts-ում `set -euo pipefail`, absolute paths և idempotent մեթոդներ (տես նաև [Linux Processes](processes.md))։

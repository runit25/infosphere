# System Scanners

## FRST

**FRST**

FRST logs contain no personal information except for your username; we use it to collect system logs, start-up entries, tasks, drivers, etc., used by the likes of Malwarebytes.

Download FRST here: <https://www.bleepingcomputer.com/download/farbar-recovery-scan-tool/dl/82/>

- Run `frst.exe`, accept the **(UAC)** prompt
- Accept the **disclaimer**
- Check mark `90 Days Files` if you began noticing problems more than 30 Days ago
- Click **Scan**
- Two logs named `FRST.txt` and `Addition.txt` will be created, upload them here further analysis

***

**Fixlist**

Download `fixlist.txt` into the same directory as `frst.exe` usually located within your download folder
- Right-click `frst.exe` select, run as administrator if the previous instance has been closed
- Click **Fix**
- Once complete, upload `fixlog.txt` here for further analysis
  - __Note:__ during the fixing process, *FRST** will close active processes (this is normal)

## Windows

**Microsoft Defender Logs**

Head to: `C:\ProgramData\Microsoft\Windows Defender\Support\`
- Upload the following `.txt` logs here for further analysis:
```
MPDectection
MPLog
```

***

**Upload your process list**

- Press `CTRL` + `ALT` then `C`
- Accept the **(UAC)** prompt
- Type the following: `Get-Process > Documents\proclist.txt`
- Open file explorer > **Documents**, upload `proclist.txt` here
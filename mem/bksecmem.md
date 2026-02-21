	vol -q -f memdump.mem -r csv windows.pslist > triage/pslist.csv
	vol -q -f memdump.mem -r csv windows.pstree > triage/pstree.csv
	vol -q -f memdump.mem -r csv windows.psscan > triage/psscan.csv
	vol -q -f memdump.mem -r csv windows.psxview > triage/psxview.csv
	vol -q -f memdump.mem -r csv windows.cmdline > triage/cmdline.csv
	vol -q -f memdump.mem -r csv windows.netscan > triage/netscan.csv
	vol -q -f memdump.mem -r csv windows.malware.malfind > triage/malfind.csv

	head -n 2 triage/pslist.csv
	head -n 2 triage/pstree.csv
	head -n 2 triage/psxview.csv
	head -n 2 triage/cmdline.csv
	head -n 2 triage/netscan.csv
	head -n 2 triage/malfind.csv
	
	PID=8144
	grep -En "(^|,)$PID(,|$)" triage/*.csv


--- 
What is the Windows version of the system being dump? (format: number)

10

---
When did the user perform memory dump? (format: YYYY-MM-DD HH:MM:SS - make sure to convert to Vietnam time)

2025-02-20 22:05:26

---
There is a suspicious process running. Determine its image name and process ID. (format: name/pid)

svcost.exe/8144

---
Which process is the parent of the suspicious process? Determine its image name and process ID. (format: name/pid)

wscript.exe/8268

---
I've been telling my friend not to pass secret/password in the command line. But he didn't think that's a major problem. Please enlighten him by finding his secret.

BKSEC{d0_n0t_p4sS_s3cr3TZ_0n_th3_cmDljn3}

---
The suspicious process is making outbound connection to the Internet. Quick, find where it connects to. (format: IP:PORT)

103.69.97.144:31337

---
Final challenge: Find the hidden flag (there are 2 parts, it's in 2 seperate 'file' - they are really easy to be found, don't worry.)





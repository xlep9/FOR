	# export ra các file 
	vol -q -f memdump.mem -r csv windows.pslist > triage/pslist.csv
	vol -q -f memdump.mem -r csv windows.pstree > triage/pstree.csv
	vol -q -f memdump.mem -r csv windows.psscan > triage/psscan.csv
	vol -q -f memdump.mem -r csv windows.psxview > triage/psxview.csv
	vol -q -f memdump.mem -r csv windows.cmdline > triage/cmdline.csv
	vol -q -f memdump.mem -r csv windows.netscan > triage/netscan.csv
	vol -q -f memdump.mem -r csv windows.malware.malfind > triage/malfind.csv

	# xem các cột của file
	head -n 2 triage/pslist.csv
	head -n 2 triage/pstree.csv
	head -n 2 triage/psxview.csv
	head -n 2 triage/cmdline.csv
	head -n 2 triage/netscan.csv
	head -n 2 triage/malfind.csv

	# tìm tất cả thông tin của một PID cụ thể  
	PID=8144
	grep -En "(^|,)$PID(,|$)" triage/*.csv

	# lọc các tiến trình có “tên lạ”
	grep -Fvxf triage/windows_core_allowlist.txt triage/all_names.txt

	# quét tất cả file thấy trong RAM và lưu ra triage/filescan.txt
	vol -q -f memdump.mem windows.filescan > triage/filescan.txt
	# lọc ra những file có tên/đuôi/từ khóa đáng nghi (.vbs, .ps1, flag, ...)
	grep -Ei 'loader\.vbs|call-remote-server\.ps1|flag2\.exe|\.vbs|\.ps1|flag' triage/filescan.txt

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

BKSEC{l00k_ljk3_w3_h4v3_a_n3w_vol4tilitY_m4st3r_c0mjn9_in2_t0wn}



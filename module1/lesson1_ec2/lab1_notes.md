### Notes

Launch EC2 from templates from Lab 0.5
Make a new version; change the instance type from t2.micro to t2.small
Use that updated launch template to start a new ec2
SSH in, run steps 0 to 4 from the runbook
Before starting uvicorn, run the command below:	
	
		whoami && TZ='Your/Zone' date && ec2-metadata -i && uptime -s

Take your screenshot showing both the ID line output and the "Application startup complete." message.


### COBOL

COBOL stands for Common Business Oriented Language. It was invented back in 1959, before PC existed. 

**How normal apps save information vs. COBOL**

Modern SOftware: When you look at information nowadays, the interface shows you only what you ask for in a clean, polished way. It grabs your balance or your name, puts it in a neat box, and keeps all the underlying file structure completely hidden behind the scenes.

COBOL programs: COBOL does not hide anything. It opens a plain text file sitting directly on the server's hard drive. It's literally a massive block of raw text that you could open and read in a basic text editor.

### The COBOL Practice

In this lab I executed a series of practical commands inside the 

		/opt/balerica-bank directory 

directory to test how the old-school COBOL program processes accounts, handles errors, and records transactions.

**Posting Transaction**
To record money moving into an account, you use the payment post command. You provide your transaction ID (starting with your initials), the account number, the amount, currency, and invoice number:

		./cobol/bin/payment_post TXN-XX-0001 100001 50.00 USD INV-101

The program processes the input and writes a new entry straight into the raw journal file (data/payments.dat), confirming the payment status and approval receipt.


**Testing account lookups and return codes**
 
Tested how the program responds to different situations by checking its status code (echo $?) immediately after running a command:

**Return Code 0 (Success):** You looked up a valid account number (like 100001). The program found the account and printed the details, leaving behind a 0 to signal that everything went perfectly.

<img width="1381" height="820" alt="cobol_rc0" src="https://github.com/user-attachments/assets/a72eec72-167c-4c73-a991-ab4c20ceceb0" />

**Return Code 4 (Account Not Found):** You searched for an account number that did not exist (like 999999). Instead of crashing, the program gracefully reported that the account was missing and returned a 4 as a warning code.

<img width="1381" height="820" alt="cobol_rc4" src="https://github.com/user-attachments/assets/ad0a5d5e-fd41-4c4b-afee-437548431d1a" />


**Return Code 8 (Business Rejection):** You tried to post a payment with an invalid rule violation, such as a zero-amount transaction. The bank rules blocked it, resulting in an 8 to indicate a business logic rejection.


<img width="1381" height="820" alt="cobol_rc8" src="https://github.com/user-attachments/assets/aabb7235-af61-4e27-872e-803d1537b37a" />


**Return Code 12 (Environment Failure):** You deliberately ran a command from the wrong folder (~ instead of the bank directory). Because the program could not find the files it needed, it failed with a 12 to show an environment or path error.

<img width="1381" height="820" alt="cobol_rc12" src="https://github.com/user-attachments/assets/c6f241e1-e674-4d30-a875-fe823c18d359" />

**Decode the fixed-width ledger by hand using cut -c:** You used the Linux cut command with the -c flag (such as cut -c 1-20,35-50 data/accounts.dat) to slice specific character columns out of the raw text ledger, allowing you to manually read account names and balances without formatting.

<img width="1381" height="820" alt="cobol_ledger_decode" src="https://github.com/user-attachments/assets/d583ef6e-a255-458b-9ee3-51dd0cdbb912" />


### Mistakes I made
* None

### Commands I now know by heart
* pwd (print working directory to see where I am standing)
* cd (change directory to move into folders like /opt/balerica-bank)
* ls (list files and folders to see what is inside the current directory)


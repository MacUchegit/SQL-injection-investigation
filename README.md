![Untitled](https://github.com/user-attachments/assets/49691d1b-e4c1-4cec-aadb-e3e619348489)
---
# 🚨 SOC165 Alert: Uncovering a Possible SQL Injection Attempt
---

## **Introduction**  
When working in a SOC environment, speed and precision matter. Alerts fly in constantly, and it's your job to separate the noise from the real threats.  

In this article, I’ll walk you through how I investigated a possible SQL Injection attack using the Let’s Defend SIEM tool — breaking down the steps, the clues, and what I uncovered.  

🧠 *Before we dive in, here’s a quick primer:*  

SQL (Structured Query Language) is how websites communicate with their databases—basically, it’s how data is stored, retrieved, or updated. But when attackers tamper with this language using cleverly crafted inputs, they can trick the system into exposing sensitive data, bypassing login screens, or even altering the database itself. That’s SQL Injection — and yes, it can be **dangerous**.  

Whether you're deep in cybersecurity or just curious about how digital investigations unfold, buckle up — you’re in for a behind-the-scenes ride!  

---  
### **🛑 Step 1: Why Did the Alert Even Trigger?**  
Before diving in headfirst, we need to understand what tripped the alarm. It’s like smelling smoke — before calling the fire department, let’s check the toaster.  

![1745068990814](https://github.com/user-attachments/assets/bb39c47d-26b8-4a57-8463-a7e2039cccdd)

Here’s the alert breakdown:  
- **EventID**: 115  
- **Time**: Feb 25, 2022, 11:34 AM  
- **Rule Triggered**: SOC165 - Possible SQL Injection Payload Detected  
- **Hostname**: WebServer1001  
- **Destination IP**: 172.16.17.18  
- **Source IP**: 167.99.169.17  
- **Request Method**: GET  
- **URL**: `https://172.16.17.18/search/?q=%22%20OR%201%20%3D%201%20--%20-`  
- **User-Agent**: Firefox 40.1  
- **Trigger Reason**: URL contains `OR 1 = 1`  
- **Action**: Allowed  

🚨 *“OR 1=1” — classic SQL injection move! It’s like trying to sneak through the back door using a skeleton key.*  

---  
### **🧪 Step 2: Let’s Examine the HTTP Traffic**  
Now it’s time to get our hands dirty and look at the traffic. Hackers don’t always knock politely — they can embed their payloads anywhere in the request.  

So, I headed over to the Log Management page and filtered logs from the suspicious IP: **167.99.169.17**.  

![1745069361918](https://github.com/user-attachments/assets/1fd01bd1-1901-440d-b189-ac8caf66d935)

Guess what I found? Requests filled with SQL-flavored words like `ORDER BY`, `OR`, and `'`.  

Here’s one of the juicy requests:  
`https://172.16.17.18/search/?q=1%27%20ORDER%20BY%203--%2B`  

I ran this through CyberChef — a handy tool for decoding and analyzing data — and got this:  
`https://172.16.17.18/search/?q=1' ORDER BY 3--+`  

![1745077731143](https://github.com/user-attachments/assets/b3fe86a0-5cfb-4895-b021-74b8a1b9e780)

Let’s break it down:  
1. `1'` → Trying to break out of the SQL query.  
2. `ORDER BY 3` → A probe to check if the database accepts SQL commands.  
3. `--+` → A SQL comment to nullify the rest of the query (*sneaky, huh?*).  

🕵️♂️ *All signs point to an SQL injection probe — a common tactic hackers use to test if your website’s database is vulnerable.*  

---  
### **❓ Step 3: Is This Actually Malicious?**  

![1745070526456](https://github.com/user-attachments/assets/1edca095-d512-494a-a75e-949c0f3d82b7)

Spoiler alert: Yes.  

Why? Because these requests mirror known SQL injection patterns. And they’re not coming from a browser casually surfing cat memes — they’re deliberately crafted, with intent to exploit.  

---  
### **🪤 Step 4: Identify the Attack Type**  

![1745070616653](https://github.com/user-attachments/assets/be038825-687f-4831-8452-717f32f56496)

This one had all the signs of a classic SQL Injection attack, where the attacker tries to manipulate a website's database by injecting malicious input into a query.  

🔍 *Things to watch out for when spotting SQL Injections:*  
- `OR 1=1`, `'1'='1'`, or similar logic — often used to bypass authentication.  
- Use of SQL keywords like `UNION`, `SELECT`, `INSERT`, `ORDER BY`, etc.  
- Suspicious characters in URLs: `'`, `--`, `%27`, `%20`, `/*`, `*/`  
- Error messages like `500 Internal Server Error`, especially after unusual inputs.  
- Input in query parameters that breaks syntax or looks like part of a SQL statement.  

In our case, the attack vector was the query parameter in a GET request containing payloads like:  
`?q=1%27%20ORDER%20BY%203--%2B`  

It’s a textbook signature — someone was clearly probing for SQL injection weaknesses.  

---  
### **🧪 Step 5: Could This Be a Test?**  
Good question. Sometimes companies run penetration tests that mimic real attacks. To check, I searched for terms like:  
- “Test”  
- “Planned”  
- “WebServer1001”  
- "SQL"  

Result? Nothing. No test was planned or announced. So this is not a drill.  

![1745070439252](https://github.com/user-attachments/assets/6d0bcbb1-db15-479a-ab1e-6f145f89613d)

---  
### **🌍 Step 6: Where’s This Coming From?**  

![1745077974894](https://github.com/user-attachments/assets/b00ef09f-2308-4c97-8005-53a7edd6be26)

The IP address **167.99.169.17** caught my eye. Is it internal? Nope. It falls outside of private ranges, which makes it a public IP.  

So I ran it through VirusTotal... and boom 💥 — 4 vendors flagged it as malicious. The IP belongs to DigitalOcean, a cloud hosting provider. That doesn't mean it's always evil, but in this context — it's not looking good.  

![1745078002800](https://github.com/user-attachments/assets/3edc253b-f77d-4ef4-9e3b-5d705e4cc0da)

---  
### **❌ Step 7: Was the Attack Successful?**  

![1745078171249](https://github.com/user-attachments/assets/ffc14655-3902-4c98-b1fa-27699146dcec)

Let’s check the HTTP responses. Normally, a healthy request gets a `200 OK`. But these SQL-injected requests? They all returned `500 Internal Server Error` — a red flag, but ironically a good one.  

It means the server didn’t handle the malicious payload gracefully (*no leaks!*). So, likely, the attack failed.  

![1745078776600](https://github.com/user-attachments/assets/6b878d30-e00c-48a5-a437-4024249a6efb)

---  
### **📌 Step 8: Add Artifacts**  
In cybersecurity, we document everything. Why? To create a paper trail — in case it escalates or we need to analyze trends.  

So I logged the malicious IP **167.99.169.17** as an artifact.  

![1745078249040](https://github.com/user-attachments/assets/9ec963e0-c9df-4ad0-b79d-d81f7e65dbb3)

---  
### **🧑💻 Step 9: Do We Escalate?**  
Since the attack didn’t succeed, and there’s no sign of compromise, there’s no need to escalate this to Tier 2 support. But we’ll definitely keep an eye on it.  

---  
### **📝 Step 10: Write a Comment Summary**  
Lastly, I wrote a short but clear summary of the incident for the SOC logs. Here’s a sample:  
> *"Detected a possible SQL injection attempt from external IP 167.99.169.17. Investigation confirms malicious intent, but the attack was unsuccessful. No further escalation required at this time."*  

---  
### **✅ Conclusion: Another Day, Another Blocked Attack**  
This little investigation shows how important it is to follow a structured approach when dealing with alerts. With tools like Let’s Defend SIEM, a curious mind, and a bit of detective work, you can sniff out even the sneakiest intrusions.  

*Stay curious. Stay alert. Stay safe. 🔐 And remember — not all heroes wear capes, some wear hoodies and analyze logs.*  

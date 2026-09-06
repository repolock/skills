# Tutorial 22: Browser Security, Anti-Tamper & API Protection Hardening

A common requirement for enterprise admission portals is:
> *"How do we stop users from inspecting the code, tampering with form data, or discovering our backend APIs?"*

This guide explains **both the frontend defense tactics** (disabling inspect, obfuscation) and the **bulletproof backend defenses** that ensure your system remains secure even if someone analyzes your network traffic.

---

## 1. The Hard Truth: Client-Side vs. Backend Security

Every senior security architect knows this law:
> **"Anything sent to the user's browser is already in the user's hands."**

Even if you block the F12 key in Chrome:
* An attacker can use **cURL**, **Postman**, or **Python scripts**.
* An attacker can use an intercepting proxy like **Burp Suite** or **OWASP ZAP**.
* An attacker can open DevTools before loading the page.

Therefore, security is built in **Two Rings**:
1. **Ring 1 (Frontend Obfuscation)**: Deters 95% of casual users, competitors, and script-kiddies from poking around.
2. **Ring 2 (Zero-Trust Backend)**: Guarantees 100% security even if an attacker dissects every single API call!

---

## 2. Ring 1: Frontend Hardening & Anti-Inspect Tactics

### Tactic A: Disabling Right-Click & DevTools Keyboard Shortcuts
You can prevent casual users from right-clicking or pressing `F12`, `Ctrl+Shift+I`, `Ctrl+Shift+J`, and `Ctrl+U`:

```javascript
// Disable Right-Click Context Menu
document.addEventListener('contextmenu', (e) => e.preventDefault());

// Block Inspection Keyboard Shortcuts
document.addEventListener('keydown', (e) => {
    // F12 key
    if (e.keyCode === 123) {
        e.preventDefault();
        return false;
    }
    // Ctrl + Shift + I (Inspect)
    // Ctrl + Shift + J (Console)
    // Ctrl + Shift + C (Element selector)
    // Ctrl + U (View Source)
    if (e.ctrlKey && (e.shiftKey && (e.key === 'I' || e.key === 'J' || e.key === 'C') || e.key === 'u')) {
        e.preventDefault();
        return false;
    }
});
```

### Tactic B: DevTools Detection & Anti-Debugging Trap
When a developer opens Chrome DevTools, the browser pauses if it encounters a `debugger` statement. We can run an infinite anti-debugging loop:

```javascript
(function() {
    function detectDevTools() {
        const start = performance.now();
        debugger; // If DevTools is open, execution freezes here!
        const end = performance.now();
        if (end - start > 100) {
            // DevTools was open! Clear screen or redirect
            document.body.innerHTML = "<h2 style='text-align:center;margin-top:50px;'>⚠️ Security Warning: Inspection tools are disabled on this portal.</h2>";
        }
    }
    setInterval(detectDevTools, 1000);
})();
```

### Tactic C: Code Obfuscation & Minification
Before deploying JavaScript to production, run it through tools like **Terser** or **JavaScript Obfuscator**.
* It turns meaningful function names like `function calculateDiscount(studentId)` into unreadable scrambled code:
  `function _0x4a1b(_0x2e8f){return _0x1c3d[_0x9b](_0x2e8f);}`
* It strips all comments, variable names, and string literals.

---

## 3. Ring 2: The Real Defense (Zero-Trust Backend)

Regardless of what happens on the browser, your **Java backend** must enforce strict security guardrails.

### 🛡️ Defense 1: Never Trust Client-Side Calculations (Parameter Tampering)
* **The Attack**: On the admission form, the frontend displays `Fee: ₹25,000`. A user opens network tools and modifies the HTTP payload to:
  `{"applicationId": "101", "amount": 1.00}`
* **The Backend Guardrail**:
  * **Never accept the price from the client!**
  * The client only sends `{"applicationId": "101"}`.
  * The Java backend queries the database, calculates the authoritative fee based on grade and category, and creates the order on Razorpay for the correct amount.

### 🛡️ Defense 2: Prevent IDOR (Insecure Direct Object References)
* **The Attack**: A counselor for School A changes the URL from `GET /api/v1/leads/1001` to `GET /api/v1/leads/1002`, attempting to snoop on another school's student.
* **The Backend Guardrail**:
  ```java
  @GetMapping("/leads/{id}")
  public Lead getLead(@PathVariable String id) {
      String currentTenant = TenantContextHolder.getContext().tenantId();
      
      // CRITICAL: Always enforce tenant ownership!
      return leadRepository.findByIdAndTenantId(id, currentTenant)
          .orElseThrow(() -> new ResourceNotFoundException("Lead not found or access denied"));
  }
  ```

### 🛡️ Defense 3: Content Security Policy (CSP) Headers
Configure Spring Security to instruct the browser never to load unauthorized scripts:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.headers(headers -> headers
        // Prevents Clickjacking (cannot embed SimplyAdmission inside an iframe on another site)
        .frameOptions(frame -> frame.deny())
        
        // Strict Content Security Policy
        .contentSecurityPolicy(csp -> csp
            .policyDirectives("default-src 'self'; script-src 'self' https://checkout.razorpay.com; object-src 'none';")
        )
    );
    return http.build();
}
```

### 🛡️ Defense 4: Rate Limiting & DDoS Shield (Redis Token Bucket)
To prevent competitors or bots from scraping inquiry forms or attempting brute-force password guessing:
* Track IP addresses in Redis.
* Allow max **5 login attempts per minute per IP**.
* Return `429 Too Many Requests` if the threshold is exceeded.

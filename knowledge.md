# Helmet - Security Middleware for NestJS/Express

## Helmet kya hai?

Helmet ek security middleware hai jo Node.js applications (Express aur NestJS) ko secure karne mein madad karta hai. Ye HTTP headers ko properly configure karke aapki application ko common web vulnerabilities se bachata hai.

## Helmet Kaise Kaam Karta Hai? (Backend se Frontend Control)

### **Important Concept: Backend Headers → Browser Enforcement**

Bahut log confuse hote hain ki "attack to frontend pe hota hai, backend kaise protect karega?"

**Answer:** Backend attacks ko detect nahi karta, balki browser ko **instructions** deta hai security headers ke through.

### Request-Response Flow:

```
┌──────────────┐                    ┌──────────────┐
│   Browser    │ ──── Request ────→ │   Server     │
│  (Frontend)  │                    │  (Backend)   │
│              │                    │   + Helmet   │
│              │ ←── Response ───── │              │
│              │   (with Headers)   │              │
└──────────────┘                    └──────────────┘
        │
        ↓
  Headers Read
        │
        ↓
  Rules Follow
```

### Step-by-Step Process:

**Step 1: Backend Response Headers Set karta hai**
```typescript
// Backend (NestJS/Express + Helmet)
app.use(helmet.frameguard({ action: 'deny' }));

// Jab bhi response jata hai, ye header automatically add hota hai
```

**Step 2: Server Response Bhejta hai**
```http
HTTP/1.1 200 OK
Content-Type: text/html
X-Frame-Options: DENY          ← Helmet ne add kiya
X-Content-Type-Options: nosniff ← Helmet ne add kiya
...

<html>...</html>
```

**Step 3: Browser Headers Ko Padhta hai**
```
Browser: "Arre, X-Frame-Options: DENY dikha!"
Browser: "Matlab ye page ko iframe mein load nahi karna hai"
```

**Step 4: Browser Rules Follow Karta hai**
```javascript
// Attacker ki site pe
<iframe src="your-site.com">  // Ye load karne ki koshish

// Browser checks headers
// X-Frame-Options: DENY dekha
// ❌ Browser refuses to load iframe
// Console error: "Refused to display in a frame"
```

---

## Real Examples se Samjho:

### Example 1: Clickjacking Protection

**Scenario:** Attacker aapki site ko apne iframe mein load karna chahta hai

```html
<!-- Attacker ki malicious website -->
<iframe src="https://yourbank.com/transfer">
  <!-- Ye load nahi hoga kyunki... -->
</iframe>
```

**Backend Code:**
```typescript
// main.ts (Backend)
app.use(helmet.frameguard({ action: 'deny' }));
```

**Kya hota hai:**

1️⃣ **User attacker ki site pe jaata hai**
2️⃣ **Attacker ka HTML browser mein load hota hai** (iframe tag ke saath)
3️⃣ **Browser iframe load karne ki koshish karta hai**
4️⃣ **Browser request bhejta hai**: `GET https://yourbank.com/transfer`
5️⃣ **Backend response deta hai WITH HEADERS**:
```http
HTTP/1.1 200 OK
X-Frame-Options: DENY  ← Helmet ne set kiya
Content-Type: text/html

<html>Your Bank Page</html>
```
6️⃣ **Browser header padhta hai**: "X-Frame-Options: DENY"
7️⃣ **Browser refuses**: "Nahi, main is page ko iframe mein nahi dikhaunga!"
8️⃣ **Iframe blank rahta hai** ❌

---

### Example 2: XSS Protection

**Scenario:** Attacker malicious script inject karne ki koshish karta hai

```typescript
// Vulnerable endpoint (backend)
app.get('/search', (req, res) => {
  res.send(`<h1>Results: ${req.query.q}</h1>`);
});

// Attack URL:
// https://yoursite.com/search?q=<script>alert('XSS')</script>
```

**Helmet Protection:**
```typescript
// main.ts (Backend)
app.use(helmet.contentSecurityPolicy({
  directives: {
    scriptSrc: ["'self'"]  // Sirf apni site ke scripts allow
  }
}));
```

**Kya hota hai:**

1️⃣ **Attacker malicious URL share karta hai**
2️⃣ **User URL open karta hai**
3️⃣ **Backend response bhejta hai**:
```http
HTTP/1.1 200 OK
Content-Security-Policy: script-src 'self'  ← Helmet ne set kiya
Content-Type: text/html

<h1>Results: <script>alert('XSS')</script></h1>
```
4️⃣ **Browser HTML parse karta hai**
5️⃣ **Browser script tag dekh kar check karta hai CSP header**
6️⃣ **CSP says: "script-src 'self'" - matlab sirf same origin se scripts**
7️⃣ **Inline script ko block kar deta hai** ❌
8️⃣ **Console error**: "Refused to execute inline script (CSP violation)"

---

## Key Points (Confusion Clear karne ke liye):

### ❓ Backend kaise frontend attack stop karta hai?

**Answer:** Backend directly attack nahi rokta. Backend browser ko **instructions (headers)** deta hai. Browser un instructions ko follow karke attack rok deta hai.

```
Backend ka kaam: "Instructions dena"
Browser ka kaam: "Instructions follow karna"
```

---

### ❓ Agar backend headers set kare, to attacker browser settings change kar sakta hai?

**Answer:** Nahi! Ye headers **server se aate hain**, attacker inhe modify nahi kar sakta.

```
Attacker Control: ❌ Cannot modify response headers
Browser Built-in: ✅ Headers ko respect karta hai
```

---

### ❓ Helmet backend pe hai, frontend pe kyu nahi?

**Answer:** Kyunki **HTTP headers** response ke saath aate hain. Aap browser ko sirf server se hi bata sakte ho ki "ye page kaise handle karna hai".

```
Frontend JavaScript: Attacker modify kar sakta hai
HTTP Headers: Server hi control karta hai (secure)
```

---

### ❓ Kya agar main directly HTML file open karu (no server)?

**Answer:** Tab headers nahi honge, protection nahi milega!

```
file:///path/to/index.html  ← No server, no headers, no Helmet protection
http://localhost:3000       ← Server hai, headers hai, Helmet protection ✅
```

---

## Visual Flow - Clickjacking Example:

```
┌─────────────────────────────────────────────────┐
│ Attacker's Website (evil.com)                   │
│                                                  │
│ <iframe src="https://yourbank.com/transfer">    │
│                          │                       │
└──────────────────────────┼───────────────────────┘
                           │
                           ↓
         ┌─────────────────────────────┐
         │   1. Browser Request        │
         │   GET /transfer             │
         │   Host: yourbank.com        │
         └─────────────────────────────┘
                           │
                           ↓
         ┌─────────────────────────────┐
         │   2. Backend (Helmet)       │
         │   Adds X-Frame-Options      │
         └─────────────────────────────┘
                           │
                           ↓
         ┌─────────────────────────────┐
         │   3. Server Response        │
         │   X-Frame-Options: DENY     │
         │   <html>Bank Page</html>    │
         └─────────────────────────────┘
                           │
                           ↓
         ┌─────────────────────────────┐
         │   4. Browser Checks Header  │
         │   "DENY dikha? Toh iframe   │
         │    mein load nahi karunga"  │
         └─────────────────────────────┘
                           │
                           ↓
         ┌─────────────────────────────┐
         │   5. Browser Blocks         │
         │   ❌ Iframe stays blank     │
         │   Console: "Refused..."     │
         └─────────────────────────────┘
```

---

## Summary:

| Question | Answer |
|----------|--------|
| Attack kahan hota hai? | Frontend (Browser) |
| Protection kahan se aata hai? | Backend (Headers) |
| Enforce kaun karta hai? | Browser |
| Helmet kya karta hai? | Response headers set karta hai |
| Kya attacker headers change kar sakta? | ❌ Nahi |
| Kya headers real attack rok sakte hain? | ✅ Haan, browser enforcement se |

---

## Helmet kyu use hota hai?

### 1. **Security Headers ko Set karta hai**
Helmet automatically kai important HTTP security headers set karta hai jo browser ko batate hain ki application ko kaise handle karna hai:

- **X-Content-Type-Options**: MIME type sniffing ko prevent karta hai
- **X-Frame-Options**: Clickjacking attacks se bachata hai
- **X-XSS-Protection**: Cross-Site Scripting (XSS) attacks ko mitigate karta hai
- **Strict-Transport-Security (HSTS)**: HTTPS connections ko enforce karta hai
- **Content-Security-Policy (CSP)**: XSS aur data injection attacks ko prevent karta hai

### 2. **Common Vulnerabilities se Protection**

```
┌─────────────────────────────────────────┐
│         Without Helmet                  │
├─────────────────────────────────────────┤
│  ❌ Clickjacking attacks possible       │
│  ❌ MIME sniffing vulnerabilities       │
│  ❌ XSS attacks easier                  │
│  ❌ Insecure connections allowed        │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         With Helmet                     │
├─────────────────────────────────────────┤
│  ✅ Frame protection enabled            │
│  ✅ MIME types locked down              │
│  ✅ XSS protection active               │
│  ✅ HTTPS enforced                      │
└─────────────────────────────────────────┘
```

## Helmet Kis Kis Attack Se Protect Karta Hai?

Helmet aapki application ko kai critical web attacks se bachata hai. Yahan har attack ka detail mein explanation hai:

### 1. **Cross-Site Scripting (XSS) Attacks**

**Attack kya hai?**
- Attacker malicious JavaScript code ko aapki website mein inject karta hai
- Ye code user ke browser mein execute hota hai
- User ka data chori ho sakta hai (cookies, session tokens, passwords)

**Helmet kaise protect karta hai?**
```typescript
// Content Security Policy se XSS prevent hota hai
app.use(helmet.contentSecurityPolicy({
  directives: {
    scriptSrc: ["'self'"],  // Sirf apni website se scripts allow
    objectSrc: ["'none'"],  // Objects block kar deta hai
  }
}));

// X-XSS-Protection header
app.use(helmet.xssFilter());  // Browser ka built-in XSS filter enable
```

**Example Attack Scenario:**
```
Bina Helmet: <script>alert(document.cookie)</script> execute hoga
Helmet ke saath: CSP block kar dega malicious script ko
```

---

### 2. **Clickjacking Attacks**

**Attack kya hai?**
- Attacker aapki website ko invisible iframe mein load karta hai
- User kuch aur click kar raha hai, lekin actually aapke page par click ho raha hai
- User unknowingly actions perform kar deta hai (example: password change, fund transfer)

**Helmet kaise protect karta hai?**
```typescript
// X-Frame-Options header set karta hai
app.use(helmet.frameguard({
  action: 'deny'  // Page ko kisi bhi iframe mein load nahi hone dega
}));

// Ya specific domains allow kar sakte ho
app.use(helmet.frameguard({
  action: 'sameorigin'  // Sirf same origin se iframe allow
}));
```

**Visual Explanation:**
```
┌─────────────────────────────────┐
│  Attacker's Malicious Site      │
│  ┌───────────────────────────┐  │
│  │ Invisible Iframe          │  │
│  │ (Your Banking Site)       │  │
│  │ [Transfer Money] ←Click   │  │
│  └───────────────────────────┘  │
│  [Click Here for Prize!] ←User  │
└─────────────────────────────────┘

Helmet ke saath: Iframe load hi nahi hoga
```

---

### 3. **Man-in-the-Middle (MITM) Attacks**

**Attack kya hai?**
- Attacker HTTP connection ko intercept kar leta hai
- User aur server ke beech data read/modify kar sakta hai
- Passwords, credit card details leak ho sakte hain

**Helmet kaise protect karta hai?**
```typescript
// HSTS (HTTP Strict Transport Security)
app.use(helmet.hsts({
  maxAge: 31536000,        // 1 year tak HTTPS force karega
  includeSubDomains: true, // Subdomains par bhi apply
  preload: true            // Browser's HSTS preload list mein
}));
```

**Example:**
```
Bina HSTS: http://example.com → HTTP pe vulnerable
HSTS ke saath: http://example.com → Automatically https://example.com redirect
```

---

### 4. **MIME Type Sniffing Attacks**

**Attack kya hai?**
- Browser file ke actual content ko dekh kar type guess karta hai
- Attacker image file mein malicious script chupa deta hai
- Browser use script samajh kar execute kar deta hai

**Helmet kaise protect karta hai?**
```typescript
// X-Content-Type-Options header
app.use(helmet.noSniff());
```

**Example Scenario:**
```javascript
// Bina noSniff:
// File: innocent.jpg (actually contains JavaScript)
// Browser: "Ye JavaScript lagta hai, chalo execute kar dete hain"

// noSniff ke saath:
// Browser: "Content-Type: image/jpeg hai, toh image hi rahega"
// Malicious code execute nahi hoga
```

---

### 5. **Information Disclosure Attacks**

**Attack kya hai?**
- Headers se technology stack pata chal jata hai
- Attacker ko known vulnerabilities exploit karne mein aasani
- Example: "X-Powered-By: Express" se pata chal gaya ki Express use ho raha hai

**Helmet kaise protect karta hai?**
```typescript
// X-Powered-By header ko hide karta hai
app.use(helmet.hidePoweredBy());
```

**Before/After:**
```bash
# Bina Helmet
X-Powered-By: Express

# Helmet ke saath
# X-Powered-By header hi nahi hoga
```

---

### 6. **DNS Prefetch Attacks**

**Attack kya hai?**
- Browser automatically DNS queries bhejta hai links ke liye
- Privacy leak ho sakti hai (user ne page visit nahi kiya, phir bhi DNS query)
- Attacker track kar sakta hai user behavior

**Helmet kaise protect karta hai?**
```typescript
app.use(helmet.dnsPrefetchControl({
  allow: false  // DNS prefetching disable
}));
```

---

### 7. **Cross-Site Request Forgery (CSRF) Related**

**Attack kya hai?**
- Malicious website user ke credentials use karke unauthorized requests bhejti hai
- Example: User logged in hai bank mein, malicious site transaction trigger karta hai

**Helmet kaise help karta hai?**
```typescript
// Referrer Policy se origin information control hoti hai
app.use(helmet.referrerPolicy({
  policy: 'no-referrer'  // Koi referrer info share nahi hogi
}));
```

---

### 8. **Code Injection Attacks**

**Attack kya hai?**
- Attacker malicious code (scripts, plugins, objects) inject karta hai
- Plugin vulnerabilities exploit ho sakti hain
- Unauthorized code execution

**Helmet kaise protect karta hai?**
```typescript
app.use(helmet.contentSecurityPolicy({
  directives: {
    objectSrc: ["'none'"],           // Flash, Java plugins block
    scriptSrc: ["'self'"],           // Sirf trusted scripts
    styleSrc: ["'self'", "'unsafe-inline'"],
    upgradeInsecureRequests: [],     // HTTP ko HTTPS mein upgrade
  }
}));
```

---

### 9. **Cross-Origin Attacks**

**Attack kya hai?**
- Malicious websites aapke resources ko unauthorized access kar sakti hain
- Data leakage cross-origin requests se

**Helmet kaise protect karta hai?**
```typescript
app.use(helmet({
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: true,
  crossOriginResourcePolicy: { policy: "same-origin" }
}));
```

---

## Attack Protection Summary Table

| Attack Type | Helmet Feature | Header Set | Risk Level |
|-------------|----------------|------------|------------|
| XSS | Content Security Policy | `Content-Security-Policy` | 🔴 Critical |
| XSS | XSS Filter | `X-XSS-Protection` | 🔴 Critical |
| Clickjacking | Frame Guard | `X-Frame-Options` | 🟠 High |
| MITM | HSTS | `Strict-Transport-Security` | 🔴 Critical |
| MIME Sniffing | No Sniff | `X-Content-Type-Options` | 🟡 Medium |
| Information Leak | Hide Powered By | (removes `X-Powered-By`) | 🟢 Low |
| DNS Tracking | DNS Prefetch Control | `X-DNS-Prefetch-Control` | 🟢 Low |
| CSRF Related | Referrer Policy | `Referrer-Policy` | 🟡 Medium |
| Cross-Origin | COEP/COOP/CORP | Multiple headers | 🟠 High |

---

## Real Attack Example aur Protection

### Example 1: XSS Attack Scenario
```typescript
// Vulnerable Code (Bina Helmet)
app.get('/search', (req, res) => {
  const query = req.query.q;
  res.send(`<h1>Search results for: ${query}</h1>`);
});

// Attack: http://site.com/search?q=<script>alert('Hacked!')</script>
// Result: Script execute hoga
```

```typescript
// Protected Code (Helmet ke saath)
app.use(helmet.contentSecurityPolicy({
  directives: {
    scriptSrc: ["'self'"]
  }
}));

// Same attack try karo
// Result: CSP violation, script execute nahi hoga
```

### Example 2: Clickjacking Attack
```html
<!-- Attacker ka site (bina Helmet protection) -->
<iframe src="http://victim-site.com/transfer-money"
        style="opacity:0; position:absolute; top:0; left:0;">
</iframe>
<button style="position:absolute; top:0; left:0;">
  Click to Win Prize!
</button>

<!-- User "prize" button click kar raha hai
     But actually "transfer money" button click ho raha hai -->
```

```typescript
// Protection
app.use(helmet.frameguard({ action: 'deny' }));
// Ab iframe load hi nahi hoga
```

---

## NestJS mein Helmet kaise use karein?

### Installation

```bash
npm install --save helmet
# ya
yarn add helmet
```

### Basic Implementation

**Method 1: Global Middleware (Recommended)**

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Helmet ko apply karein
  app.use(helmet());

  await app.listen(3000);
}
bootstrap();
```

**Method 2: Custom Configuration**

```typescript
// main.ts
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Custom helmet configuration
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        scriptSrc: ["'self'"],
        imgSrc: ["'self'", 'data:', 'https:'],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
  }));

  await app.listen(3000);
}
bootstrap();
```

## Helmet ke Features

### 1. **Content Security Policy (CSP)**
```typescript
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "trusted-cdn.com"],
  }
}));
```
- XSS attacks ko prevent karta hai
- Resource loading ko control karta hai

### 2. **HSTS (HTTP Strict Transport Security)**
```typescript
app.use(helmet.hsts({
  maxAge: 31536000,
  includeSubDomains: true
}));
```
- HTTP ko HTTPS mein force karta hai
- Man-in-the-middle attacks se bachata hai

### 3. **X-Frame-Options**
```typescript
app.use(helmet.frameguard({ action: 'deny' }));
```
- Clickjacking attacks ko prevent karta hai
- Page ko iframe mein load hone se rokta hai

### 4. **X-Content-Type-Options**
```typescript
app.use(helmet.noSniff());
```
- MIME type confusion attacks ko rokta hai

## Real-World Production Example

```typescript
// main.ts - Production Ready Setup
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Comprehensive Helmet Configuration
  app.use(helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        baseUri: ["'self'"],
        fontSrc: ["'self'", "https:", "data:"],
        formAction: ["'self'"],
        frameAncestors: ["'self'"],
        imgSrc: ["'self'", "data:", "https:"],
        objectSrc: ["'none'"],
        scriptSrc: ["'self'"],
        scriptSrcAttr: ["'none'"],
        styleSrc: ["'self'", "https:", "'unsafe-inline'"],
        upgradeInsecureRequests: [],
      },
    },

    // Cross-Origin Policies
    crossOriginEmbedderPolicy: true,
    crossOriginOpenerPolicy: true,
    crossOriginResourcePolicy: { policy: "same-origin" },

    // DNS Prefetch Control
    dnsPrefetchControl: { allow: false },

    // Frame Options
    frameguard: { action: "deny" },

    // Hide Powered By
    hidePoweredBy: true,

    // HSTS
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },

    // IE No Open
    ieNoOpen: true,

    // No Sniff
    noSniff: true,

    // Origin Agent Cluster
    originAgentCluster: true,

    // Permitted Cross Domain Policies
    permittedCrossDomainPolicies: { permittedPolicies: "none" },

    // Referrer Policy
    referrerPolicy: { policy: "no-referrer" },

    // XSS Filter
    xssFilter: true,
  }));

  await app.listen(3000);
}
bootstrap();
```

## Benefits Summary

| Feature | Benefit | Risk Mitigated |
|---------|---------|----------------|
| CSP | Resource loading control | XSS, Code Injection |
| HSTS | Force HTTPS | Man-in-the-Middle |
| X-Frame-Options | Prevent framing | Clickjacking |
| X-Content-Type-Options | MIME type safety | MIME Confusion |
| Hide X-Powered-By | Hide tech stack | Information Disclosure |
| XSS Filter | Browser XSS protection | Cross-Site Scripting |

## Best Practices

### ✅ DO:
- Hamesha production mein Helmet use karein
- CSP policies ko apni requirements ke according configure karein
- Regular basis par security headers ko audit karein
- HTTPS ke saath HSTS enable karein

### ❌ DON'T:
- Development aur production mein same configuration use na karein
- Bina testing ke strict CSP policies lagu na karein
- Default configuration par blindly rely na karein
- Security headers ko completely disable na karein

## Testing Helmet Headers

```bash
# Headers ko curl se check karein
curl -I http://localhost:3000

# Expected output (with Helmet):
# X-Content-Type-Options: nosniff
# X-Frame-Options: DENY
# Strict-Transport-Security: max-age=31536000; includeSubDomains
# Content-Security-Policy: default-src 'self'
```

## Performance Impact

- **Minimal overhead**: Headers chhoti hoti hain (~1KB)
- **One-time configuration**: Runtime mein koi calculation nahi
- **Network overhead**: Negligible (headers har response mein jate hain)
- **Security benefit**: Performance cost se kahin zyada

## Common Issues & Solutions

### Issue 1: CSP blocking resources
```typescript
// Solution: Trusted domains add karein
contentSecurityPolicy: {
  directives: {
    scriptSrc: ["'self'", "cdn.example.com"],
    styleSrc: ["'self'", "'unsafe-inline'"], // Agar zarurat ho
  }
}
```

### Issue 2: CORS conflicts
```typescript
// @nestjs/cors alag se use karein
app.enableCors({
  origin: 'https://example.com',
  credentials: true,
});
app.use(helmet());
```

### Issue 3: Development vs Production
```typescript
// Conditional helmet configuration
const helmetConfig = process.env.NODE_ENV === 'production'
  ? { /* strict config */ }
  : { contentSecurityPolicy: false };

app.use(helmet(helmetConfig));
```

## Conclusion

Helmet ek essential security middleware hai jo minimal configuration ke saath maximum security provide karta hai. Ye OWASP ke top security recommendations ko implement karta hai aur production applications ke liye industry standard hai.

**Key Takeaway**: Hamesha production NestJS applications mein Helmet use karein. Ye aapki application ko common web vulnerabilities se bachane ka sabse aasan aur effective tarika hai.

## Additional Resources

- Official Docs: https://helmetjs.github.io/
- NestJS Security Guide: https://docs.nestjs.com/security/helmet
- OWASP Security Headers: https://owasp.org/www-project-secure-headers/

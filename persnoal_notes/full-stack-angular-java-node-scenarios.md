# Full-Stack Interview Scenarios: Angular with Java (Spring Boot) & Node.js

When interviewing for a full-stack role, the focus shifts heavily to the boundary where the frontend (Angular) meets the backend (Java/Spring Boot or Node.js). 

Here are the most critical full-stack concepts and questions you will face, categorized simply.

---

## 1. The Holy Grail: JWT (JSON Web Tokens) Authentication Flow

**The Interview Question:** "Walk me through the exact process of logging a user in and keeping them authenticated through subsequent requests using a frontend Angular app and a backend Node.js or Java server."

### How it Works (The Flow):
1. **Login:** Angular sends `POST /login` with username/password.
2. **Backend Validation:** Node.js (via Passport or bcrypt) or Java (via Spring Security) checks the DB.
3. **Token Generation:** Upon success, the backend signs a JWT using a private secret key (`HS256` or `RS256`) and returns it.
4. **Storage:** Angular receives the token. *(See "The Storage Debate" below).*
5. **Subsequent API Calls:** Angular attaches the JWT as a header: `Authorization: Bearer <token>`.
6. **Backend Verification:** Java/Node.js reads the incoming header, verifies the signature using the secret key (proving it wasn't tampered with), and allows access.

### The Storage Debate (Crucial to know):
* **Where do you store the token in the browser?**
  * **Option A `localStorage` (Most Common, Least Secure):** Easy to read via JS. Vulnerable to **XSS** (Cross-Site Scripting) attacks where injected malicious scripts steal the token.
  * **Option B `HttpOnly Cookies` (Industry Standard):** The backend sets a cookie flag `HttpOnly`. JavaScript (and Angular) *cannot* physically read this token anymore. It is automatically attached by the browser on every API request. Completely defeats XSS token theft!

### Angular Implementation (The JWT Interceptor)
If using `localStorage`, you MUST use an interceptor to append the token.
```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('jwt_token');
  if (token) {
    const cloned = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next(cloned);
  }
  return next(req);
};
```
*(If using HttpOnly Cookies, use `withCredentials: true` in your HttpClient so the browser includes the cookie).*

---

## 2. Dealing with CORS (Cross-Origin Resource Sharing)

**The Interview Question:** "You are running Angular on `localhost:4200` and Spring Boot on `localhost:8080`. Your API call fails with a CORS error. What is CORS and how do you fix it?"

### What is CORS?
CORS is a **browser security mechanism**, not a backend error. The browser blocks web pages from making HTTP requests to a different domain/port than the one that served the web page, to prevent malicious sites from draining data.

### The Fix:
You do NOT fix CORS in Angular. You fix it on the backend... OR bypass it locally for dev.

* **Backend Fix (Java/Spring Boot):**
  Add the `@CrossOrigin(origins = "http://localhost:4200")` annotation to your Controller, or configure a global CORS bean.
* **Backend Fix (Node.js/Express):**
  `app.use(cors({ origin: 'http://localhost:4200' }));`
* **Angular Dev Bypass (proxy.conf.json):**
  Create a proxy so Angular's dev server pretends to serve the API itself, tricking the browser.
  ```json
  {
    "/api/*": {
      "target": "http://localhost:8080",
      "secure": false
    }
  }
  ```

---

## 3. Refresh Tokens (Session Extension)

**The Interview Question:** "JWTs should have short lifespans (e.g., 15 minutes) for security. But we don't want the user logging in every 15 minutes. How do we solve this?"

**The Solution:**
1. The backend issues two tokens on login: A short-lived **Access Token** (15 mins) and a long-lived **Refresh Token** (7 days, stored in HTTP-only DB/cookie).
2. When the Access Token expires, the Java/Node backend returns a `401 Unauthorized`.
3. In Angular, the `globalErrorHandlerInterceptor` catches the `401` error.
4. The Interceptor uses `switchMap` to pause all outgoing API requests, hits a special `/refresh` endpoint sending the Refresh Token, receives a new Access Token, updates the `localStorage`, and instantly replays the previously failed API requests.

---

## 4. Anti-CSRF (Cross-Site Request Forgery)

**The Interview Question:** "You properly secured your app by storing the JWT in an HttpOnly cookie against XSS. But now you are vulnerable to CSRF. How do you mitigate it?"

**The Solution (Double Submit Cookie Pattern or CSRF Token):**
1. The Java/Node backend sends a CSRF Token header/cookie.
2. An attacker's website can automatically trigger credentials via cookies, but the attacker *cannot read* the token due to CORS strictness.
3. Therefore, Angular is configured to read the `XSRF-TOKEN` cookie (which is NOT HttpOnly) and copy its value into an HTTP Header `X-XSRF-TOKEN` on all POST/PUT/DELETE requests.
4. The Backend checks: Does the header match the Cookie? An attacker's forged site won't have the header because they couldn't read the cookie.
*(Fun fact: Angular's `HttpClient` does this exact pattern automatically if the backend sets an `XSRF-TOKEN` cookie!)*

---

## 5. WebSockets & Real-Time Data

**The Interview Question:** "Our application needs to show live stock prices hitting our Java backend. HTTP GET polling is too slow and resource-heavy. How do we integrate?"

**The Solution:**
1. **Java/Spring:** Use `spring-boot-starter-websocket` using STOMP protocol.
2. **Node.js:** Use `Socket.io`.
3. **Angular Integration:** Create a dedicated Singleton Service. Don't manipulate the DOM directly. Instead, map the incoming WebSocket push events into an RxJS **Subject/Observable**.
4. Now, any Angular component can just `this.socketService.livePrices$.subscribe(...)` securely.

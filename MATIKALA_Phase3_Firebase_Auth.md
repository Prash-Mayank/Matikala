# MATIKALA — Phase 3: Firebase Authentication
### Complete Step-by-Step Integration Guide

---

## What You're Wiring

| Feature | File | Current State | After Phase 3 |
|---|---|---|---|
| Email + Password Login | account.html | Demo stub | Real Firebase Auth |
| Signup + Phone OTP | account.html | Accepts any 6 digits | Firebase Phone Auth |
| Google OAuth | account.html | Stub toast | Real Google popup |
| Admin Login | admin.html | Fixed hardcoded users | Firebase Auth + Firestore role check |
| Delivery Login | delivery.html | Fixed hardcoded users | Firebase Auth + Firestore role check |
| Forgot Password | account.html | Stub toast | Firebase sendPasswordResetEmail |
| Delete Account | account.html | Stub toast | Firebase deleteUser |
| Persist Login | all pages | Lost on refresh | Firebase onAuthStateChanged |

---

## STEP 1 — Create Your Firebase Project

### 1.1 Go to Firebase Console
1. Open [console.firebase.google.com](https://console.firebase.google.com)
2. Click **"Add project"**
3. Name: `matikala-ecommerce`
4. Disable Google Analytics (not needed on free tier)
5. Click **"Create project"**

### 1.2 Register a Web App
1. Inside your project → click **`</>`** (Web icon)
2. App nickname: `matikala-web`
3. ✅ Check **"Also set up Firebase Hosting"**
4. Click **"Register app"**
5. **Copy the config object** — you'll need it in Step 3:

```javascript
// You will get something like this — yours will have real values
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "matikala-ecommerce.firebaseapp.com",
  projectId: "matikala-ecommerce",
  storageBucket: "matikala-ecommerce.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef"
};
```

---

## STEP 2 — Enable Authentication Methods

In Firebase Console → **Authentication** → **Sign-in method** tab:

### 2.1 Enable Email/Password
- Click **Email/Password** → Toggle **Enable** → Save

### 2.2 Enable Phone (for OTP signup)
- Click **Phone** → Toggle **Enable** → Save
- ⚠️ Phone auth requires a **real domain** or localhost for reCAPTCHA
- For localhost testing: add `localhost` to **Authorized domains** (Auth → Settings → Authorized domains)

### 2.3 Enable Google OAuth
1. Click **Google** → Toggle **Enable**
2. Set **Project support email** → your email
3. Save
4. In **Google Cloud Console** → OAuth consent screen → add your domain when you go live

### 2.4 Add Authorized Domains
Auth → Settings → Authorized domains → Add:
- `localhost`
- `matikala-ecommerce.web.app` (Firebase Hosting default)
- `matikala.com` (when you have a custom domain)

---

## STEP 3 — Create `firebase.js` (Your Service Layer)

Create a new file: `firebase.js` — place it next to your HTML files.

```javascript
// ═══════════════════════════════════════════════
//  firebase.js  —  MATIKALA Firebase Service Layer
//  Handles: Auth, Firestore, Storage
// ═══════════════════════════════════════════════

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import {
  getAuth,
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  signInWithPhoneNumber,
  GoogleAuthProvider,
  signInWithPopup,
  signOut,
  sendPasswordResetEmail,
  deleteUser,
  updateProfile,
  onAuthStateChanged,
  RecaptchaVerifier
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
import {
  getFirestore,
  doc,
  setDoc,
  getDoc,
  updateDoc,
  serverTimestamp
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

// ── YOUR CONFIG (replace with your real values from Step 1.2) ──
const firebaseConfig = {
  apiKey:            "REPLACE_ME",
  authDomain:        "REPLACE_ME.firebaseapp.com",
  projectId:         "REPLACE_ME",
  storageBucket:     "REPLACE_ME.appspot.com",
  messagingSenderId: "REPLACE_ME",
  appId:             "REPLACE_ME"
};

// ── Init ──
const app  = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db   = getFirestore(app);
auth.languageCode = 'en';   // set to 'hi' for Hindi OTP SMS

// ═══════════════════════════════════════════════
//  A. EMAIL / PASSWORD AUTH
// ═══════════════════════════════════════════════

/**
 * Login with email + password
 * Returns: { user } on success, throws on failure
 */
export async function loginWithEmail(email, password) {
  return await signInWithEmailAndPassword(auth, email, password);
}

/**
 * Create account with email + password
 * Also creates user document in Firestore users collection
 */
export async function signupWithEmail(name, email, phone, password) {
  const cred = await createUserWithEmailAndPassword(auth, email, password);
  await updateProfile(cred.user, { displayName: name });
  await createUserDoc(cred.user, { name, email, phone, role: 'user' });
  return cred;
}

// ═══════════════════════════════════════════════
//  B. PHONE OTP AUTH (for signup verification)
// ═══════════════════════════════════════════════

let confirmationResult = null;

/**
 * Send OTP to phone number
 * containerId: ID of the invisible reCAPTCHA div in your HTML
 * phoneNumber: must be in E.164 format, e.g. "+919876543210"
 */
export async function sendPhoneOTP(phoneNumber, containerId = 'recaptcha-container') {
  // Create invisible reCAPTCHA (only once)
  if (!window._recaptchaVerifier) {
    window._recaptchaVerifier = new RecaptchaVerifier(auth, containerId, {
      size: 'invisible',
      callback: () => {}
    });
  }
  confirmationResult = await signInWithPhoneNumber(auth, phoneNumber, window._recaptchaVerifier);
  return confirmationResult;
}

/**
 * Verify OTP entered by user
 * Returns Firebase credential on success
 */
export async function verifyPhoneOTP(otp) {
  if (!confirmationResult) throw new Error('No OTP sent yet. Call sendPhoneOTP first.');
  return await confirmationResult.confirm(otp);
}

// ═══════════════════════════════════════════════
//  C. GOOGLE OAUTH
// ═══════════════════════════════════════════════

const googleProvider = new GoogleAuthProvider();
googleProvider.setCustomParameters({ prompt: 'select_account' });

/**
 * Sign in with Google popup
 * Creates Firestore user doc if first time
 */
export async function loginWithGoogle() {
  const result = await signInWithPopup(auth, googleProvider);
  await ensureUserDoc(result.user);
  return result;
}

// ═══════════════════════════════════════════════
//  D. SIGN OUT & ACCOUNT MANAGEMENT
// ═══════════════════════════════════════════════

export async function logoutUser() {
  await signOut(auth);
}

export async function resetPassword(email) {
  await sendPasswordResetEmail(auth, email);
}

export async function deleteAccount() {
  const user = auth.currentUser;
  if (!user) throw new Error('No user logged in');
  await deleteUser(user);
}

// ═══════════════════════════════════════════════
//  E. AUTH STATE OBSERVER
//  Call this on every page to persist sessions
// ═══════════════════════════════════════════════

/**
 * onAuthStateChanged wrapper
 * callback(user) — user is null if logged out
 */
export function watchAuthState(callback) {
  return onAuthStateChanged(auth, callback);
}

export function getCurrentUser() {
  return auth.currentUser;
}

// ═══════════════════════════════════════════════
//  F. FIRESTORE USER DOCUMENTS
// ═══════════════════════════════════════════════

/**
 * Create a new user document in Firestore
 * Called after email signup or first Google login
 */
export async function createUserDoc(firebaseUser, extraData = {}) {
  const ref = doc(db, 'users', firebaseUser.uid);
  await setDoc(ref, {
    uid:       firebaseUser.uid,
    name:      extraData.name  || firebaseUser.displayName || '',
    email:     extraData.email || firebaseUser.email || '',
    phone:     extraData.phone || firebaseUser.phoneNumber || '',
    photoURL:  firebaseUser.photoURL || '',
    role:      extraData.role || 'user',   // 'user' | 'admin' | 'delivery'
    createdAt: serverTimestamp(),
    addresses: [],
    wishlist:  []
  });
}

/**
 * Create user doc only if it doesn't already exist
 * Safe to call on every Google login
 */
export async function ensureUserDoc(firebaseUser) {
  const ref  = doc(db, 'users', firebaseUser.uid);
  const snap = await getDoc(ref);
  if (!snap.exists()) {
    await createUserDoc(firebaseUser);
  }
  return snap.exists() ? snap.data() : null;
}

/**
 * Fetch user document from Firestore
 * Returns user data object or null
 */
export async function getUserDoc(uid) {
  const snap = await getDoc(doc(db, 'users', uid));
  return snap.exists() ? snap.data() : null;
}

/**
 * Update user profile fields in Firestore
 */
export async function updateUserDoc(uid, fields) {
  await updateDoc(doc(db, 'users', uid), fields);
}

/**
 * Get user role from Firestore
 * Returns: 'user' | 'admin' | 'delivery' | null
 */
export async function getUserRole(uid) {
  const data = await getUserDoc(uid);
  return data ? data.role : null;
}

export { auth, db };
```

---

## STEP 4 — Add reCAPTCHA Container to account.html

Find the `<body>` tag in **account.html** and add this invisible div right after it:

```html
<body>
  <!-- ⬇ ADD THIS — required for Firebase Phone Auth reCAPTCHA ⬇ -->
  <div id="recaptcha-container"></div>
  <!-- rest of your page... -->
```

---

## STEP 5 — Wire account.html Auth Functions

Replace the 5 stub functions in **account.html** with these Firebase-wired versions.

Add this at the **top of your `<script>` block** (the import):

```html
<script type="module">
  import {
    loginWithEmail, signupWithEmail, sendPhoneOTP, verifyPhoneOTP,
    loginWithGoogle, logoutUser, resetPassword, deleteAccount,
    watchAuthState, getUserDoc, updateUserDoc
  } from './firebase.js';
```

> ⚠️ Change your `<script>` tag to `<script type="module">` — required for ES module imports.

---

### 5.1 — Replace `doLogin()`

**Find and replace:**
```javascript
// BEFORE (stub)
function doLogin() {
    const em = document.getElementById('li-email').value.trim();
    const pw = document.getElementById('li-pw').value;
    if (!em || !pw) { showToast('Please fill in all fields'); return; }
    doLoginSuccess('Shivank Singh', em);
}
```

```javascript
// AFTER (Firebase)
async function doLogin() {
    const em = document.getElementById('li-email').value.trim();
    const pw = document.getElementById('li-pw').value;
    if (!em || !pw) { showToast('Please fill in all fields', 'error'); return; }
    const btn = document.querySelector('#tab-login .lw-btn');
    btn.disabled = true;
    btn.innerHTML = '<i class="fa-solid fa-spinner fa-spin"></i> Signing in…';
    try {
        const cred = await loginWithEmail(em, pw);
        const userData = await getUserDoc(cred.user.uid);
        doLoginSuccess(userData?.name || cred.user.displayName || em.split('@')[0], em);
        showToast('Welcome back!', 'success');
    } catch (err) {
        showToast(firebaseErrMsg(err.code), 'error');
    } finally {
        btn.disabled = false;
        btn.innerHTML = '<i class="fa-solid fa-arrow-right"></i> Sign In';
    }
}
```

---

### 5.2 — Replace `sendSignupOtp()`

**Find and replace:**
```javascript
// BEFORE (stub) — the long validation function that ends with:
//   pendingSignup = { name: nm, email: em, phone: ... }
//   showToast('Sending OTP...')  ← fake
```

```javascript
// AFTER (Firebase)
async function sendSignupOtp() {
    const nm = document.getElementById('su-name').value.trim();
    const em = document.getElementById('su-email').value.trim();
    const ph = document.getElementById('su-phone').value.trim();
    const pw = document.getElementById('su-pw').value;
    const agreed = document.getElementById('su-agree')?.checked;

    if (!nm)  { showToast('Please enter your full name', 'error'); return; }
    if (!em || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(em)) { showToast('Please enter a valid email', 'error'); return; }
    if (!ph || !/^\d{10}$/.test(ph)) { showToast('Please enter a valid 10-digit mobile number', 'error'); return; }
    if (!pw || pw.length < 8) { showToast('Password must be at least 8 characters', 'error'); return; }
    if (!agreed) { showToast('Please agree to the Terms & Privacy Policy', 'error'); return; }

    const btn = document.getElementById('su-submit-btn');
    btn.disabled = true;
    btn.innerHTML = '<i class="fa-solid fa-spinner fa-spin"></i> Sending OTP…';

    try {
        pendingSignup = { name: nm, email: em, phone: ph, password: pw };
        const phoneE164 = '+91' + ph;            // ← India prefix, adjust if needed
        await sendPhoneOTP(phoneE164, 'recaptcha-container');
        // Show the OTP screen
        document.getElementById('signup-form-step').style.display = 'none';
        document.getElementById('signup-otp-step').style.display  = 'block';
        startOtpTimer();
        showToast('OTP sent to +91 ' + ph, 'success');
    } catch (err) {
        showToast(firebaseErrMsg(err.code), 'error');
        // Reset reCAPTCHA on error so user can retry
        if (window._recaptchaVerifier) {
            window._recaptchaVerifier.clear();
            window._recaptchaVerifier = null;
        }
    } finally {
        btn.disabled = false;
        btn.innerHTML = '<i class="fa-solid fa-mobile-screen-button"></i> Send OTP & Verify';
    }
}
```

---

### 5.3 — Replace `verifySignupOtp()`

```javascript
// BEFORE (stub — accepts any 6 digits except 000000)

// AFTER (Firebase)
async function verifySignupOtp() {
    const otp = [0,1,2,3,4,5].map(i => document.getElementById('otp' + i).value).join('');
    if (otp.length < 6) { showOtpErr('Please enter all 6 digits'); return; }

    const btn = document.getElementById('otp-verify-btn');
    btn.disabled = true;
    btn.innerHTML = '<i class="fa-solid fa-spinner fa-spin"></i> Verifying…';

    try {
        // 1. Verify the OTP (signs in with phone number)
        await verifyPhoneOTP(otp);
        // 2. Now create the email/password account (primary login method)
        await signupWithEmail(
            pendingSignup.name,
            pendingSignup.email,
            pendingSignup.phone,
            pendingSignup.password
        );
        clearInterval(otpTimer);
        doLoginSuccess(pendingSignup.name, pendingSignup.email);
        showToast('Account created! Welcome to MATIKALA 🎉', 'success');
        pendingSignup = null;
    } catch (err) {
        showOtpErr(firebaseErrMsg(err.code));
    } finally {
        btn.disabled = false;
        btn.innerHTML = '<i class="fa-solid fa-circle-check"></i> Verify & Create Account';
    }
}
```

---

### 5.4 — Replace `socialLogin()`

```javascript
// BEFORE (stub)
function socialLogin(provider) {
    showToast(`${provider} login — Phase 3 (Firebase Auth)`);
    setTimeout(() => doLoginSuccess('Demo User', 'demo@matikala.in'), 800);
}

// AFTER (Firebase)
async function socialLogin(provider) {
    if (provider !== 'Google') { showToast(`${provider} login coming soon`, 'warn'); return; }
    const btn = document.querySelector(`[onclick="socialLogin('Google')"]`);
    if (btn) { btn.disabled = true; btn.style.opacity = '.6'; }
    try {
        const result = await loginWithGoogle();
        const u = result.user;
        doLoginSuccess(u.displayName || u.email.split('@')[0], u.email);
        showToast('Signed in with Google!', 'success');
    } catch (err) {
        if (err.code !== 'auth/popup-closed-by-user') {
            showToast(firebaseErrMsg(err.code), 'error');
        }
    } finally {
        if (btn) { btn.disabled = false; btn.style.opacity = '1'; }
    }
}
```

---

### 5.5 — Replace `logout()`

```javascript
// BEFORE
function logout() { isLoggedIn = false; ... }

// AFTER
async function logout() {
    try {
        await logoutUser();
    } catch (e) {}
    isLoggedIn = false;
    currentUser = null;
    document.getElementById('authWall').style.display = 'flex';
    document.getElementById('accountView').style.display = 'none';
    showToast('Signed out successfully', 'success');
}
```

---

### 5.6 — Replace `showForgot()` / password reset

```javascript
// BEFORE — stub toast

// AFTER
async function sendPasswordReset() {
    const em = document.getElementById('forgot-email')?.value.trim()
               || document.getElementById('li-email').value.trim();
    if (!em) { showToast('Please enter your email address', 'error'); return; }
    try {
        await resetPassword(em);
        showToast('Password reset email sent! Check your inbox.', 'success');
    } catch (err) {
        showToast(firebaseErrMsg(err.code), 'error');
    }
}
```

---

### 5.7 — Replace `confirmDeleteAccount()`

```javascript
// BEFORE — stub toast

// AFTER
async function confirmDeleteAccount() {
    const confirm = window.confirm('Are you sure? This cannot be undone.');
    if (!confirm) return;
    try {
        await deleteAccount();
        showToast('Account deleted.', 'success');
        logout();
    } catch (err) {
        // Firebase requires recent login before deletion
        if (err.code === 'auth/requires-recent-login') {
            showToast('Please sign out and sign back in, then try again.', 'error');
        } else {
            showToast(firebaseErrMsg(err.code), 'error');
        }
    }
}
```

---

### 5.8 — Add Session Persistence (onAuthStateChanged)

Add this at the **bottom of your script**, after all function definitions:

```javascript
// ── Persist login across page refreshes ──
watchAuthState(async (user) => {
    if (user) {
        const userData = await getUserDoc(user.uid);
        doLoginSuccess(
            userData?.name || user.displayName || user.email.split('@')[0],
            user.email
        );
    } else {
        // Not logged in — show auth wall
        document.getElementById('authWall').style.display = 'flex';
        document.getElementById('accountView').style.display = 'none';
    }
});
```

---

### 5.9 — Add Error Message Helper

Add this utility function anywhere in your script:

```javascript
function firebaseErrMsg(code) {
    const map = {
        'auth/user-not-found':         'No account found with this email.',
        'auth/wrong-password':         'Incorrect password. Please try again.',
        'auth/email-already-in-use':   'An account with this email already exists.',
        'auth/weak-password':          'Password must be at least 6 characters.',
        'auth/invalid-email':          'Please enter a valid email address.',
        'auth/too-many-requests':      'Too many attempts. Please wait and try again.',
        'auth/network-request-failed': 'Network error. Please check your connection.',
        'auth/popup-closed-by-user':   'Sign-in popup was closed.',
        'auth/invalid-verification-code': 'Incorrect OTP. Please try again.',
        'auth/code-expired':           'OTP has expired. Please request a new one.',
        'auth/quota-exceeded':         'SMS quota exceeded. Please try email login.',
        'auth/requires-recent-login':  'Please sign in again to perform this action.',
    };
    return map[code] || 'Something went wrong. Please try again.';
}
```

---

## STEP 6 — Wire admin.html to Firebase (Role Check)

### 6.1 Add import to admin.html script

```html
<script type="module">
import { loginWithEmail, logoutUser, watchAuthState, getUserRole } from './firebase.js';
```

### 6.2 Replace `adminLogin()` in admin.html

```javascript
// BEFORE — checks against hardcoded ADMIN_USERS array

// AFTER (Firebase)
async function adminLogin() {
    const em  = document.getElementById('lwEmail').value.trim().toLowerCase();
    const pw  = document.getElementById('lwPw').value;
    const err = document.getElementById('lwErr');
    if (!em) { err.textContent = 'Please enter your email address'; return; }
    if (!pw) { err.textContent = 'Please enter your password'; return; }
    err.textContent = '';

    const btn = document.querySelector('.lw-btn');
    btn.disabled = true;
    btn.innerHTML = '<i class="fa-solid fa-spinner fa-spin"></i> Signing in…';

    try {
        const cred = await loginWithEmail(em, pw);
        const role = await getUserRole(cred.user.uid);

        if (role !== 'admin') {
            await logoutUser();
            err.textContent = 'Access denied. This account does not have admin privileges.';
            return;
        }

        // Get name from ADMIN_USERS array fallback or Firestore
        const found = ADMIN_USERS.find(u => u.email === em) || { name: cred.user.displayName, avatar: cred.user.displayName?.[0] || 'A', role: 'Admin' };
        adminUser = found;
        document.getElementById('loginWall').style.display = 'none';
        document.getElementById('sbName').textContent   = found.name;
        document.getElementById('sbAvatar').textContent = found.avatar;
        document.getElementById('sbRole').textContent   = found.role;
        document.getElementById('tbAvatar').textContent = found.avatar;
        document.getElementById('tbName').textContent   = found.name.split(' ')[0];
        document.getElementById('tbRole').textContent   = found.role;
        initDashboard();
        showAdmin('dashboard');
        showToast('Welcome back, ' + found.name.split(' ')[0] + '!', 'success');
    } catch (err2) {
        err.textContent = firebaseErrMsg(err2.code);
    } finally {
        btn.disabled = false;
        btn.innerHTML = '<i class="fa-solid fa-lock"></i> Sign In as Admin';
    }
}

async function adminLogout() {
    await logoutUser();
    adminUser = null;
    document.getElementById('loginWall').style.display = 'flex';
}
```

---

## STEP 7 — Wire delivery.html to Firebase

### 7.1 Replace `doLogin()` in delivery.html

```javascript
// AFTER (Firebase)
async function doLogin() {
    const inp = document.getElementById('lwEmail').value.trim().toLowerCase();
    const pw  = document.getElementById('lwPw').value;
    const err = document.getElementById('lwErr');
    if (!inp) { err.textContent = 'Please enter your email'; return; }
    if (!pw)  { err.textContent = 'Please enter your password'; return; }
    err.textContent = '';

    const btn = document.querySelector('.lw-btn');
    btn.disabled = true;
    btn.innerHTML = '<i class="fa-solid fa-spinner fa-spin"></i> Signing in…';

    try {
        const cred = await loginWithEmail(inp, pw);
        const role = await getUserRole(cred.user.uid);

        if (role !== 'delivery') {
            await logoutUser();
            err.textContent = 'Access denied. This is not a delivery partner account.';
            return;
        }

        const userData = await getUserDoc(cred.user.uid);
        currentUser = { ...userData, avatar: userData.name?.[0] || 'D' };
        document.getElementById('loginWall').style.display = 'none';
        setUserUI(currentUser);
        initDashboard();
        goto('dashboard');
        showToast('Welcome back, ' + userData.name.split(' ')[0] + '! 🚀', 'success');
    } catch (err2) {
        err.textContent = firebaseErrMsg(err2.code);
    } finally {
        btn.disabled = false;
        btn.innerHTML = '<i class="fa-solid fa-motorcycle"></i> Sign In';
    }
}
```

---

## STEP 8 — Firestore Security Rules

In Firebase Console → **Firestore Database** → **Rules** tab.
Replace the default rules with:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ── Helper functions ──
    function isSignedIn() {
      return request.auth != null;
    }
    function isOwner(uid) {
      return request.auth.uid == uid;
    }
    function getUserRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    function isAdmin() {
      return isSignedIn() && getUserRole() == 'admin';
    }
    function isDelivery() {
      return isSignedIn() && getUserRole() == 'delivery';
    }

    // ── Users collection ──
    match /users/{userId} {
      allow read:   if isOwner(userId) || isAdmin();
      allow create: if isSignedIn() && isOwner(userId);
      allow update: if isOwner(userId) || isAdmin();
      allow delete: if isAdmin();
    }

    // ── Products collection (public read, admin write) ──
    match /products/{productId} {
      allow read:  if true;
      allow write: if isAdmin();
    }

    // ── Categories (public read, admin write) ──
    match /categories/{catId} {
      allow read:  if true;
      allow write: if isAdmin();
    }

    // ── Orders ──
    match /orders/{orderId} {
      // Customer can read/create their own orders
      allow read:   if isAdmin()
                    || isOwner(resource.data.userId)
                    || (isDelivery() && resource.data.deliveryPartnerId == request.auth.uid);
      allow create: if isSignedIn() && request.resource.data.userId == request.auth.uid;
      // Only admin or assigned delivery partner can update status
      allow update: if isAdmin()
                    || (isDelivery() && resource.data.deliveryPartnerId == request.auth.uid);
      allow delete: if isAdmin();
    }

    // ── Cart (user owns their cart) ──
    match /cart/{userId} {
      allow read, write: if isOwner(userId);
    }

    // ── Wishlists ──
    match /wishlists/{userId} {
      allow read, write: if isOwner(userId);
    }

    // ── Reviews ──
    match /reviews/{reviewId} {
      allow read:   if true;
      allow create: if isSignedIn();
      allow update, delete: if isAdmin()
                            || isOwner(resource.data.userId);
    }

    // ── Delivery Partners ──
    match /deliveryPartners/{partnerId} {
      allow read:   if isAdmin() || isOwner(partnerId);
      allow write:  if isAdmin();
      allow update: if isOwner(partnerId);   // partner updates their own availability/status
    }

    // ── Notifications ──
    match /notifications/{notifId} {
      allow read:   if isOwner(resource.data.userId) || isAdmin();
      allow write:  if isAdmin();
    }

    // ── Analytics (admin only) ──
    match /analytics/{doc} {
      allow read, write: if isAdmin();
    }
  }
}
```

---

## STEP 9 — Create Admin & Delivery Users in Firebase

You can't create admin/delivery users through the normal signup form (they'd get `role: 'user'`). Create them manually:

### Option A — Firebase Console (easiest)
1. **Authentication** → **Users** → **Add user**
2. Enter email + password (e.g. `shivank@matikala.com` / strong password)
3. Copy the **UID** shown
4. **Firestore** → **users** collection → **Add document**
   - Document ID = the UID you copied
   - Fields:
     ```
     uid:       "paste-uid-here"
     name:      "Shivank Singh"
     email:     "shivank@matikala.com"
     role:      "admin"             ← this is what grants access
     phone:     "+919458509527"
     createdAt: (timestamp)
     ```

5. Repeat for Mayank → `role: "admin"`
6. Repeat for delivery partners → `role: "delivery"`

### Option B — One-time Setup Script (run once, then delete)
Create `setup-users.html` temporarily:

```html
<script type="module">
import { signupWithEmail } from './firebase.js';
import { doc, setDoc } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";
import { db } from './firebase.js';

// Run this ONCE to create admin users, then delete this file
async function createAdmin(name, email, password, phone) {
    const cred = await signupWithEmail(name, email, phone, password);
    // Override role to 'admin'
    await setDoc(doc(db, 'users', cred.user.uid), { role: 'admin' }, { merge: true });
    console.log('Created admin:', email, '| UID:', cred.user.uid);
}

createAdmin('Shivank Singh',  'shivank@matikala.com', 'shivank@123', '9458509527');
createAdmin('Mayank Prashar', 'mayank@matikala.com',  'mayank@123',  '9458509527');
</script>
```

> ⚠️ **Delete this file immediately after running it once.**

---

## STEP 10 — Test Checklist

Before going live, test each flow:

```
□ Email signup → OTP → account created in Firebase Auth + Firestore
□ Email login → correct password → logged in
□ Email login → wrong password → error message shown (not console error)
□ Google login → popup → user doc created in Firestore
□ Forgot password → email received → can reset
□ Page refresh → still logged in (onAuthStateChanged)
□ Logout → auth wall shown
□ Admin login → shivank@matikala.com / shivank@123 → dashboard loads
□ Admin login → regular user email → "Access denied" shown
□ Delivery login → rajan@matikala.com / rajan@123 → delivery dashboard loads
□ Delivery login → regular user email → "Access denied" shown
□ Firestore rules → user can't read another user's data (test in Rules Simulator)
```

---

## STEP 11 — Deploy to Firebase Hosting

```bash
# Install Firebase CLI (once)
npm install -g firebase-tools

# Login
firebase login

# In your project folder
firebase init hosting

# When prompted:
# - Public directory: . (dot — your HTML files are at root)
# - Single-page app: No
# - Overwrite index.html: No

# Deploy
firebase deploy --only hosting
```

Your site will be live at:
`https://matikala-ecommerce.web.app`

---

## Quick Reference — Files Changed

| File | Changes |
|---|---|
| `firebase.js` | **NEW** — service layer, create this first |
| `account.html` | Add `type="module"` to script, add `#recaptcha-container`, replace 7 functions |
| `admin.html` | Add `type="module"` to script, replace `adminLogin()` + `adminLogout()` |
| `delivery.html` | Add `type="module"` to script, replace `doLogin()` + `doLogout()` |
| Firebase Console | Enable Email, Phone, Google auth methods |
| Firestore Rules | Replace default rules with RBAC rules from Step 8 |

---

## What's Next After Phase 3

| Phase | What it covers |
|---|---|
| **Phase 4** | Cart persistence in Firestore + Razorpay checkout |
| **Phase 5** | Order system — real-time status updates, order tracking |
| **Phase 6** | Admin panel wired to Firestore — live products, orders, users |
| **Phase 7** | Delivery panel wired — assigned orders from Firestore, proof of delivery |
| **Phase 8** | FCM push notifications + automated emails via Firebase Functions |

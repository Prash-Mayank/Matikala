# MATIKALA — Handcrafted Clay & Terracotta E-Commerce Platform

> *"Where Earth Meets Art"* — A full-featured e-commerce platform for handcrafted clay, terracotta & sustainable decor products.

![MATIKALA Banner](./other/imag.png)

---

## Overview

**MATIKALA** is a production-ready e-commerce web platform built with **HTML5, Tailwind CSS, JavaScript** and powered by **Firebase** (free tier). It supports three user roles — Customers, Admins, and Delivery Partners — with real-time order tracking, multi-language support, and Razorpay payment integration.

---

## Features

### Customer Panel
- Email OTP & Google OAuth login
- Browse, search & filter products
- Cart system with quantity management
- Checkout with UPI, Cards, Wallets & Net Banking
- Real-time order tracking
- Wishlist management
- Profile & address management
- Multi-language support (EN, HI, TA, TE, KN, ML, BN, MR)

### Admin Panel
- Sales analytics & revenue graphs (Chart.js)
- Product & category management
- Order management & status updates
- Delivery partner assignment
- User management & blocking
- CMS — Edit banners & featured products
- Invoice & report generation

### Delivery Partner Panel
- View assigned orders
- Pickup & delivery navigation
- OTP verification & proof of delivery
- Earnings dashboard

---

## Project Structure

```
matikala/
│
├── index.html                  # Homepage
├── account.html                # User Account
├── cart.html                   # Shopping Cart
├── checkout.html               # Checkout Flow
├── order-history.html          # Order History
├── order-tracking.html         # Real-time Order Tracking
├── delivery.html               # Delivery Partner Panel
├── admin.html                  # Admin Dashboard
├── payment-success.html        # Payment Success Page
├── payment-failed.html         # Payment Failed Page
├── 404.html                    # 404 Error Page
│
├── other/
│   ├── about.html              # About Us
│   ├── contact.html            # Contact Page
│   ├── faq.html                # FAQ
│   ├── shipping.html           # Shipping Policy
│   ├── return-refund.html      # Return & Refund Policy
│   ├── cancellation.html       # Cancellation Policy
│   ├── privacy-policy.html     # Privacy Policy
│   ├── terms-of-service.html   # Terms of Service
│   ├── cookies.html            # Cookie Policy
│   └── care-instructions.html  # Product Care Instructions
│
└── README.md
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | HTML5, Tailwind CSS, JavaScript |
| Animations | Framer Motion |
| i18n | i18next (8 languages) |
| Backend | Firebase (Auth, Firestore, Storage, Functions) |
| Payments | Razorpay |
| Charts | Chart.js |
| Emails | Firebase Functions + SendGrid |
| Notifications | Firebase Cloud Messaging |

---

Live at: `https://matikala.web.app` or `matikala.com`

---

## Development Roadmap

- [x] Phase 1 — UI Base Layout
- [x] Phase 2 — Authentication System
- [x] Phase 3 — Product & Category System
- [x] Phase 4 — Cart, Checkout & Payments
- [x] Phase 5 — Order System & Tracking
- [x] Phase 6 — Admin Dashboard
- [x] Phase 7 — Delivery Partner System
- [ ] Phase 8 — Notifications, Emails & Analytics

---

## License

  MIT License

---

## Author

**Mayank Prashar**

- GitHub: [github.com/prash-mayank](https://github.com/prash-mayank)
- LinkedIn: [linkedin.com/in/prashmayank](https://www.linkedin.com/in/prashmayank)
- Email: [mayank.prash@gmail.com](mailto:mayank.prash@gmail.com)

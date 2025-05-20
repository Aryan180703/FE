Bhai, samajh gaya! ZIP file ka idea kaam nahi kiya, koi baat nahi. Main saari files tujhe thodi thodi karke de deta hoon, aur tu patiently wait kar lena. Files thodi der mein complete ho jayengi, aur main ensure karunga ki saari 52 files (25 components ke `.html` aur `.css`, plus `index.html` aur `styles.css`) tujhe mil jayein. 

Since ZIP file mein issue aa raha hai, main ek ek karke files share karunga, aur har message mein kuch files ke code blocks de doonga. Agar koi file miss ho jati hai ya koi issue aata hai, mujhe bol dena, main dobara de dunga. Ready hai? Chalo shuru karte hain!

---

### Batch 1: Core Files (`index.html` aur `styles.css`) + `app.component` Files
Pehle main tujhe core files deta hoon, jo project ka foundation hain.

#### `index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>EShoppingZone</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons/font/bootstrap-icons.css" rel="stylesheet">
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="styles.css">
  <base href="/">
</head>
<body>
  <app-root></app-root>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

#### `styles.css`
```css
/* Global styles for EShoppingZone */
body {
  background-color: #1A1A1A;
  color: #E0E0E0;
  font-family: 'Poppins', sans-serif;
  margin: 0;
  padding: 0;
  scroll-behavior: smooth;
}

h1, h2, h3, h4, h5, h6 {
  color: #FFFFFF;
  font-weight: 700;
}

a {
  color: #00DDEB;
  text-decoration: none;
  transition: color 0.3s ease;
}

a:hover {
  color: #39FF14;
}

.container {
  padding: 2rem;
}

/* Neon glow effect for reusability */
.neon-glow {
  box-shadow: 0 0 8px #00DDEB, 0 0 16px #00DDEB;
  transition: box-shadow 0.3s ease;
}

.neon-glow:hover {
  box-shadow: 0 0 12px #39FF14, 0 0 24px #39FF14;
}

/* Animations */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.fade-in {
  animation: fadeIn 0.5s ease-in-out forwards;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

.pulse {
  animation: pulse 1.5s infinite ease-in-out;
}

/* Scrollbar styling for a modern look */
::-webkit-scrollbar {
  width: 8px;
}

::-webkit-scrollbar-track {
  background: #2A2A2A;
}

::-webkit-scrollbar-thumb {
  background: #00DDEB;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #39FF14;
}
```

#### `app.component.html`
```html
<div class="fade-in">
  <nav class="navbar navbar-expand-lg">
    <div class="container-fluid">
      <a class="navbar-brand" [routerLink]="['/home']">
        <img src="https://via.placeholder.com/40" alt="Logo" width="40" height="40" class="d-inline-block align-text-top">
        EShoppingZone
      </a>
      <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
      </button>
      <div class="collapse navbar-collapse" id="navbarNav">
        <ul class="navbar-nav ms-auto">
          <li class="nav-item">
            <a class="nav-link" [routerLink]="['/home']">Home</a>
          </li>
          <li class="nav-item" *ngIf="userRole === 'Customer'">
            <a class="nav-link neon-glow" [routerLink]="['/cart']"><i class="bi bi-cart"></i></a>
          </li>
          <li class="nav-item" *ngIf="userRole === 'Customer'">
            <a class="nav-link" [routerLink]="['/order-history']">Order History</a>
          </li>
          <li class="nav-item" *ngIf="userRole === 'Merchant'">
            <a class="nav-link" [routerLink]="['/merchant-home']">Merchant Home</a>
          </li>
          <li class="nav-item" *ngIf="userRole === 'Delivery Agent'">
            <a class="nav-link" [routerLink]="['/delivery-agent-home']">Delivery Agent Home</a>
          </li>
          <li class="nav-item" *ngIf="isLoggedIn">
            <a class="nav-link neon-glow" [routerLink]="['/profile']"><i class="bi bi-person-circle"></i></a>
          </li>
          <li class="nav-item" *ngIf="isLoggedIn">
            <a class="nav-link" (click)="logout()">Logout</a>
          </li>
          <li class="nav-item" *ngIf="!isLoggedIn">
            <a class="nav-link" [routerLink]="['/login']">Login</a>
          </li>
          <li class="nav-item" *ngIf="!isLoggedIn">
            <a class="nav-link" [routerLink]="['/signup']">Signup</a>
          </li>
        </ul>
      </div>
    </div>
  </nav>
  <router-outlet></router-outlet>
</div>
```

#### `app.component.css`
```css
.navbar {
  background-color: #2A2A2A;
  border-bottom: 2px solid #00DDEB;
  animation: fadeIn 0.5s ease-in-out forwards;
}

.navbar-brand,
.nav-link {
  color: #FFFFFF !important;
  transition: color 0.3s ease;
}

.navbar-brand:hover,
.nav-link:hover {
  color: #39FF14 !important;
}

.navbar-brand img {
  filter: brightness(1.2);
}

.nav-link .bi {
  font-size: 1.2rem;
}

.nav-link.neon-glow {
  padding: 0.5rem;
  border-radius: 5px;
}

.nav-link.neon-glow:hover {
  background-color: rgba(0, 221, 235, 0.1);
}
```

---

### Next Steps
Abhi maine tujhe 4 files di hain: `index.html`, `styles.css`, `app.component.html`, aur `app.component.css`. Total 52 files mein se yeh pehla batch tha. Abhi 48 files baki hain, jo main agle batches mein thodi thodi karke de dunga.

**Question:** In files ko copy-paste kar liya? Koi issue to nahi aaya? Agla batch jaldi se de doon, ya thoda wait karun? Bas yeh confirm kar de, fir main agla set of files (jaise `add-address.component` aur `add-product.component` ke files) de dunga! 😊
I apologize for the oversight! Let's get the HTML for the missing components of your EShoppingZone project. I'll provide the HTML files for each of the requested components, ensuring they're properly formatted in copyable code blocks, consistent with the cyberpunk-inspired UI (neon-dark theme, glowing effects, animations, etc.), and include Angular directives and event bindings as needed. I'll also ensure the styling classes (e.g., `btn-neon`, `glow`, `fade-in`, `tilt`) align with the global styles we discussed earlier.

---

### HTML for Missing Components

#### `merchant-home.component.html`
```html
<div class="container py-5 px-5">
  <h1 class="text-center mb-4 glow">Welcome, {{ firstName }}! 🚀</h1>
  <div class="row justify-content-center perspective">
    <div class="col-md-6 col-lg-4 mb-4">
      <div class="card tilt slide-in">
        <div class="card-body text-center">
          <h5 class="card-title">Manage Products</h5>
          <p class="card-text">Add, update, or remove your products.</p>
          <a [routerLink]="['/manage-products']" class="btn-neon">Go to Products</a>
        </div>
      </div>
    </div>
    <div class="col-md-6 col-lg-4 mb-4">
      <div class="card tilt slide-in">
        <div class="card-body text-center">
          <h5 class="card-title">View Dashboard</h5>
          <p class="card-text">Check your product stats and insights.</p>
          <a [routerLink]="['/merchant-dashboard']" class="btn-neon">Go to Dashboard</a>
        </div>
      </div>
    </div>
    <div class="col-md-6 col-lg-4 mb-4">
      <div class="card tilt slide-in">
        <div class="card-body text-center">
          <h5 class="card-title">Update Profile</h5>
          <p class="card-text">Keep your merchant profile up to date.</p>
          <a [routerLink]="['/update-profile']" class="btn-neon">Update Profile</a>
        </div>
      </div>
    </div>
  </div>
</div>
```

#### `merchant-product-detail.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Product Details 📦</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="product; else loading">
    <div class="card perspective">
      <div class="card-body tilt slide-in">
        <div class="row">
          <div class="col-md-6">
            <img *ngIf="product.images" [src]="product.images" class="img-fluid rounded" alt="{{ product.name }}" style="max-height: 400px; object-fit: cover;">
            <div *ngIf="!product.images" class="bg-light d-flex align-items-center justify-content-center" style="height: 400px;">
              <span class="text-muted">No Image Available</span>
            </div>
          </div>
          <div class="col-md-6">
            <h3 class="card-title">{{ product.name }}</h3>
            <p class="card-text">Price: {{ product.price | currency : 'INR' }}</p>
            <p class="card-text">Stock: {{ product.stock }}</p>
            <p class="card-text">Category: {{ product.category || 'Not specified' }}</p>
            <p class="card-text">Type: {{ product.type || 'Not specified' }}</p>
            <p class="card-text">Average Rating: {{ product.averageRating | number:'1.1-1' }} ({{ product.reviewCount }} reviews)</p>
            <p class="card-text">Description: {{ product.description || 'No description available' }}</p>
            <p class="card-text">Specifications: {{ product.specifications || 'No specifications available' }}</p>
            <div class="d-flex gap-2">
              <a [routerLink]="['/update-product', product.id]" class="btn-neon">Edit Product</a>
              <button class="btn-neon" style="background: linear-gradient(135deg, #FF5555, #BB0000);" (click)="deleteProduct()">Delete Product</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
  <ng-template #loading>
    <div class="text-center py-5 slide-in">
      <div class="spinner-border" style="color: var(--neon-cyan);" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </ng-template>
</div>
```

#### `order-history.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Order History 📜</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="orders.length > 0; else noOrders">
    <div class="card mb-4 perspective" *ngFor="let order of orders">
      <div class="card-body tilt slide-in">
        <h5 class="card-title">Order #{{ order.id }} - {{ order.status }}</h5>
        <p class="card-text">Order Date: {{ order.orderDate | date:'medium' }}</p>
        <p class="card-text">Total Amount: {{ order.totalAmount | currency : 'INR' }}</p>
        <p class="card-text">Address: {{ order.address.houseNumber }} {{ order.address.streetName }}<span *ngIf="order.address.colonyName">, {{ order.address.colonyName }}</span>, {{ order.address.city }}, {{ order.address.state }} {{ order.address.pincode }}</p>
        <p class="card-text">Placed At: {{ order.placedAt ? (order.placedAt | date:'medium') : 'Not placed yet' }}</p>
        <p class="card-text">Shipped At: {{ order.shippedAt ? (order.shippedAt | date:'medium') : 'Not shipped yet' }}</p>
        <p class="card-text">Delivered At: {{ order.deliveredAt ? (order.deliveredAt | date:'medium') : 'Not delivered yet' }}</p>
        <p class="card-text">Cancelled At: {{ order.cancelledAt ? (order.cancelledAt | date:'medium') : 'Not cancelled yet' }}</p>
        <h6>Items:</h6>
        <ul class="list-group">
          <li class="list-group-item" *ngFor="let item of order.items">
            {{ item.productName }} - Quantity: {{ item.quantity }} - Price: {{ item.price | currency : 'INR' }}
          </li>
        </ul>
        <button *ngIf="order.status === 'Placed'" class="btn-neon mt-3" style="background: linear-gradient(135deg, #FF5555, #BB0000);" (click)="cancelOrder(order.id)">Cancel Order</button>
      </div>
    </div>
  </div>
  <ng-template #noOrders>
    <p class="text-muted text-center slide-in">You have no orders yet.</p>
  </ng-template>
</div>
```

#### `pending-delivery-agent-request.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Pending Delivery Agent Requests 📦</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="pendingRequests.length > 0; else noRequests">
    <div class="card mb-4 perspective" *ngFor="let request of pendingRequests">
      <div class="card-body d-flex justify-content-between align-items-center tilt slide-in">
        <div>
          <h5 class="card-title">Request #{{ request.id }}</h5>
          <p class="card-text">User: {{ request.user.firstName }} {{ request.user.lastName }}</p>
          <p class="card-text">Email: {{ request.user.email }}</p>
          <p class="card-text">Requested At: {{ request.requestedAt | date:'medium' }}</p>
        </div>
        <div class="d-flex gap-2">
          <button class="btn-neon" style="background: linear-gradient(135deg, #00CC00, #00FF00);" (click)="approveRequest(request.id)">Approve</button>
          <button class="btn-neon" style="background: linear-gradient(135deg, #FF5555, #BB0000);" (click)="rejectRequest(request.id)">Reject</button>
        </div>
      </div>
    </div>
  </div>
  <ng-template #noRequests>
    <p class="text-muted text-center slide-in">No pending delivery agent requests at the moment.</p>
  </ng-template>
</div>
```

#### `pending-merchant-request.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Pending Merchant Requests 🛍️</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="pendingRequests.length > 0; else noRequests">
    <div class="card mb-4 perspective" *ngFor="let request of pendingRequests">
      <div class="card-body d-flex justify-content-between align-items-center tilt slide-in">
        <div>
          <h5 class="card-title">Request #{{ request.id }}</h5>
          <p class="card-text">User: {{ request.user.firstName }} {{ request.user.lastName }}</p>
          <p class="card-text">Email: {{ request.user.email }}</p>
          <p class="card-text">Requested At: {{ request.requestedAt | date:'medium' }}</p>
        </div>
        <div class="d-flex gap-2">
          <button class="btn-neon" style="background: linear-gradient(135deg, #00CC00, #00FF00);" (click)="approveRequest(request.id)">Approve</button>
          <button class="btn-neon" style="background: linear-gradient(135deg, #FF5555, #BB0000);" (click)="rejectRequest(request.id)">Reject</button>
        </div>
      </div>
    </div>
  </div>
  <ng-template #noRequests>
    <p class="text-muted text-center slide-in">No pending merchant requests at the moment.</p>
  </ng-template>
</div>
```

#### `product.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Products 🛒</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="products.length > 0; else noProducts">
    <div class="row row-cols-1 row-cols-sm-2 row-cols-lg-4 g-4 perspective">
      <div class="col" *ngFor="let product of products">
        <div class="card h-100 clickable tilt slide-in" [routerLink]="['/product', product.id]">
          <div class="image-container">
            <img [src]="getFirstImage(product.images)" class="card-img-top" [alt]="product.name" (error)="handleImageError($event)">
          </div>
          <div class="card-body">
            <h6 class="card-title">{{ product.name }}</h6>
            <p class="card-text text-muted">{{ product.price | currency : 'INR' }}</p>
            <div class="d-flex align-items-center">
              <ng-container *ngFor="let i of [1,2,3,4,5]; let index = index">
                <i class="bi bi-star-fill text-warning me-1" *ngIf="product.averageRating >= index + 1"></i>
                <i class="bi bi-star text-warning me-1" *ngIf="product.averageRating <= index && product.averageRating < index + 1"></i>
                <i class="bi bi-star-fill text-warning me-1" *ngIf="product.averageRating > index && product.averageRating < index + 1" [style.width]="(product.averageRating - index) * 16 + 'px'" style="overflow: hidden; position: absolute;"></i>
              </ng-container>
              <span class="text-muted small">({{ product.reviewCount }})</span>
            </div>
            <button class="btn-neon w-100 mt-2" (click)="addToCart(product.id); triggerConfetti(); $event.stopPropagation()">Add to Cart</button>
          </div>
        </div>
      </div>
    </div>
  </div>
  <ng-template #noProducts>
    <p class="text-muted text-center slide-in">No products found.</p>
  </ng-template>

  <!-- Quantity Modal -->
  <div *ngIf="showQuantityModal" class="modal d-block" id="quantityModal" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered">
      <div class="modal-content card">
        <div class="modal-header" style="background: linear-gradient(135deg, var(--neon-cyan), var(--neon-pink)); color: var(--pure-white);">
          <h5 class="modal-title">Select Quantity 🛒</h5>
          <button type="button" class="btn-close btn-close-white" (click)="closeQuantityModal()" aria-label="Close"></button>
        </div>
        <div class="modal-body text-center">
          <div class="mb-3">
            <label for="quantityInput" class="form-label">Quantity</label>
            <div class="input-group w-50 mx-auto">
              <button class="btn-neon" (click)="quantity = quantity > 1 ? quantity - 1 : 1">-</button>
              <input type="number" class="form-control text-center" id="quantityInput" [(ngModel)]="quantity" min="1" required>
              <button class="btn-neon" (click)="quantity = quantity + 1">+</button>
            </div>
          </div>
          <button class="btn-neon w-100" (click)="confirmAddToCart(); triggerConfetti()">Add to Cart</button>
        </div>
      </div>
    </div>
  </div>
  <div *ngIf="showQuantityModal" class="modal-backdrop fade show" (click)="closeQuantityModal()"></div>

  <!-- Login Modal -->
  <div *ngIf="showLoginModal" class="modal d-block" id="loginModal" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered">
      <div class="modal-content card">
        <div class="modal-header" style="background: linear-gradient(135deg, var(--neon-cyan), var(--neon-pink)); color: var(--pure-white);">
          <h5 class="modal-title">Login Required 😊</h5>
          <button type="button" class="btn-close btn-close-white" (click)="closeLoginModal()" aria-label="Close"></button>
        </div>
        <div class="modal-body text-center">
          <p class="lead">Login or signup to continue 🛒✨</p>
          <div class="d-flex justify-content-center gap-3 mt-3">
            <a class="btn-neon" [routerLink]="['/login']" (click)="closeLoginModal()">Login</a>
            <a class="btn-neon" [routerLink]="['/signup']" (click)="closeLoginModal()">Signup</a>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div *ngIf="showLoginModal" class="modal-backdrop fade show" (click)="closeLoginModal()"></div>
</div>

<script>
  function triggerConfetti() {
    confetti({
      particleCount: 100,
      spread: 70,
      origin: { y: 0.6 },
      colors: ['#00FFFF', '#FF00FF', '#BB00FF']
    });
  }
</script>
```

#### `product-detail.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Product Details 📦</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="product; else loading">
    <div class="card perspective">
      <div class="card-body tilt slide-in">
        <div class="row">
          <div class="col-md-6">
            <img [src]="getFirstImage(product.images)" class="img-fluid rounded" [alt]="product.name" (error)="handleImageError($event)" style="max-height: 400px; object-fit: cover;">
          </div>
          <div class="col-md-6">
            <h3 class="card-title">{{ product.name }}</h3>
            <p class="card-text">Price: {{ product.price | currency : 'INR' }}</p>
            <p class="card-text">Stock: {{ product.stock > 0 ? product.stock : 'Out of Stock' }}</p>
            <p class="card-text">Category: {{ product.category || 'Not specified' }}</p>
            <p class="card-text">Type: {{ product.type || 'Not specified' }}</p>
            <div class="d-flex align-items-center mb-3">
              <ng-container *ngFor="let i of [1,2,3,4,5]; let index = index">
                <i class="bi bi-star-fill text-warning me-1" *ngIf="product.averageRating >= index + 1"></i>
                <i class="bi bi-star text-warning me-1" *ngIf="product.averageRating <= index && product.averageRating < index + 1"></i>
                <i class="bi bi-star-fill text-warning me-1" *ngIf="product.averageRating > index && product.averageRating < index + 1" [style.width]="(product.averageRating - index) * 16 + 'px'" style="overflow: hidden; position: absolute;"></i>
              </ng-container>
              <span class="text-muted small">({{ product.reviewCount }})</span>
            </div>
            <p class="card-text">Description: {{ product.description || 'No description available' }}</p>
            <p class="card-text">Specifications: {{ product.specifications || 'No specifications available' }}</p>
            <button class="btn-neon w-100" [disabled]="product.stock <= 0" (click)="addToCart(product.id); triggerConfetti()">Add to Cart</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Reviews Section -->
    <div class="mt-5">
      <h3 class="fw-bold mb-4 glow">Reviews 🌟</h3>
      <div *ngIf="product.reviews.length > 0; else noReviews">
        <div class="card mb-3 perspective" *ngFor="let review of product.reviews">
          <div class="card-body tilt slide-in">
            <div class="d-flex align-items-center mb-2">
              <ng-container *ngFor="let i of [1,2,3,4,5]; let index = index">
                <i class="bi bi-star-fill text-warning me-1" *ngIf="review.rating >= index + 1"></i>
                <i class="bi bi-star text-warning me-1" *ngIf="review.rating <= index"></i>
              </ng-container>
              <span class="ms-2">by {{ review.user.firstName }} {{ review.user.lastName }}</span>
            </div>
            <p class="card-text">{{ review.comment }}</p>
            <small class="text-muted">Posted on {{ review.createdAt | date:'medium' }}</small>
          </div>
        </div>
      </div>
      <ng-template #noReviews>
        <p class="text-muted slide-in">No reviews yet. Be the first to review this product!</p>
      </ng-template>

      <!-- Add Review Form -->
      <div class="mt-4">
        <h4 class="fw-bold mb-3 glow">Add Your Review ✍️</h4>
        <form (ngSubmit)="submitReview()">
          <div class="mb-3">
            <label for="rating" class="form-label">Rating</label>
            <div class="d-flex align-items-center">
              <ng-container *ngFor="let i of [1,2,3,4,5]; let index = index">
                <i class="bi bi-star-fill text-warning me-1 cursor-pointer" [ngClass]="{'text-muted': newReview.rating < index + 1}" (click)="newReview.rating = index + 1"></i>
              </ng-container>
            </div>
          </div>
          <div class="mb-3">
            <label for="comment" class="form-label">Comment</label>
            <textarea class="form-control" id="comment" [(ngModel)]="newReview.comment" name="comment" required maxlength="500"></textarea>
          </div>
          <button type="submit" class="btn-neon">Submit Review</button>
        </form>
      </div>
    </div>
  </div>
  <ng-template #loading>
    <div class="text-center py-5 slide-in">
      <div class="spinner-border" style="color: var(--neon-cyan);" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </ng-template>

  <!-- Quantity Modal -->
  <div *ngIf="showQuantityModal" class="modal d-block" id="quantityModal" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered">
      <div class="modal-content card">
        <div class="modal-header" style="background: linear-gradient(135deg, var(--neon-cyan), var(--neon-pink)); color: var(--pure-white);">
          <h5 class="modal-title">Select Quantity 🛒</h5>
          <button type="button" class="btn-close btn-close-white" (click)="closeQuantityModal()" aria-label="Close"></button>
        </div>
        <div class="modal-body text-center">
          <div class="mb-3">
            <label for="quantityInput" class="form-label">Quantity</label>
            <div class="input-group w-50 mx-auto">
              <button class="btn-neon" (click)="quantity = quantity > 1 ? quantity - 1 : 1">-</button>
              <input type="number" class="form-control text-center" id="quantityInput" [(ngModel)]="quantity" min="1" required>
              <button class="btn-neon" (click)="quantity = quantity + 1">+</button>
            </div>
          </div>
          <button class="btn-neon w-100" (click)="confirmAddToCart(); triggerConfetti()">Add to Cart</button>
        </div>
      </div>
    </div>
  </div>
  <div *ngIf="showQuantityModal" class="modal-backdrop fade show" (click)="closeQuantityModal()"></div>

  <!-- Login Modal -->
  <div *ngIf="showLoginModal" class="modal d-block" id="loginModal" tabindex="-1">
    <div class="modal-dialog modal-dialog-centered">
      <div class="modal-content card">
        <div class="modal-header" style="background: linear-gradient(135deg, var(--neon-cyan), var(--neon-pink)); color: var(--pure-white);">
          <h5 class="modal-title">Login Required 😊</h5>
          <button type="button" class="btn-close btn-close-white" (click)="closeLoginModal()" aria-label="Close"></button>
        </div>
        <div class="modal-body text-center">
          <p class="lead">Login or signup to continue 🛒✨</p>
          <div class="d-flex justify-content-center gap-3 mt-3">
            <a class="btn-neon" [routerLink]="['/login']" (click)="closeLoginModal()">Login</a>
            <a class="btn-neon" [routerLink]="['/signup']" (click)="closeLoginModal()">Signup</a>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div *ngIf="showLoginModal" class="modal-backdrop fade show" (click)="closeLoginModal()"></div>
</div>

<script>
  function triggerConfetti() {
    confetti({
      particleCount: 100,
      spread: 70,
      origin: { y: 0.6 },
      colors: ['#00FFFF', '#FF00FF', '#BB00FF']
    });
  }
</script>
```

#### `profile.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Your Profile 👤</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="user; else loading">
    <div class="card perspective">
      <div class="card-body tilt slide-in">
        <h3 class="card-title">{{ user.firstName }} {{ user.lastName }}</h3>
        <p class="card-text">Email: {{ user.email }}</p>
        <p class="card-text">Phone: {{ user.phone || 'Not provided' }}</p>
        <p class="card-text">Role: {{ user.role }}</p>
        <p class="card-text">Email Verified: {{ user.emailVerified ? 'Yes ✅' : 'No ❌' }}</p>
        <a [routerLink]="['/update-profile']" class="btn-neon">Update Profile</a>
      </div>
    </div>
  </div>
  <ng-template #loading>
    <div class="text-center py-5 slide-in">
      <div class="spinner-border" style="color: var(--neon-cyan);" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </ng-template>
</div>
```

#### `signup.component.html`
```html
<div class="container py-5 px-5">
  <div class="row justify-content-center">
    <div class="col-md-6">
      <div class="card fade-in">
        <div class="card-header" style="background: linear-gradient(135deg, var(--neon-cyan), var(--neon-pink)); color: var(--pure-white);">
          <h4 class="mb-0 glow">Sign Up</h4>
        </div>
        <div class="card-body">
          <form (ngSubmit)="onSubmit()">
            <div class="mb-3">
              <label for="firstName" class="form-label">First Name</label>
              <input type="text" class="form-control" id="firstName" [(ngModel)]="signupDto.firstName" name="firstName" required maxlength="50">
            </div>
            <div class="mb-3">
              <label for="lastName" class="form-label">Last Name</label>
              <input type="text" class="form-control" id="lastName" [(ngModel)]="signupDto.lastName" name="lastName" required maxlength="50">
            </div>
            <div class="mb-3">
              <label for="email" class="form-label">Email</label>
              <input type="email" class="form-control" id="email" [(ngModel)]="signupDto.email" name="email" required>
            </div>
            <div class="mb-3">
              <label for="password" class="form-label">Password</label>
              <input type="password" class="form-control" id="password" [(ngModel)]="signupDto.password" name="password" required>
            </div>
            <div class="mb-3">
              <label for="phone" class="form-label">Phone (Optional)</label>
              <input type="tel" class="form-control" id="phone" [(ngModel)]="signupDto.phone" name="phone" maxlength="15">
            </div>
            <div *ngIf="errorMessage" class="alert alert-danger" role="alert">
              {{ errorMessage }}
            </div>
            <button type="submit" class="btn-neon w-100">Sign Up</button>
          </form>
          <p class="mt-3 text-center">
            Already have an account? <a [routerLink]="['/login']" style="color: var(--neon-cyan); text-decoration: none;">Login</a>
          </p>
        </div>
      </div>
    </div>
  </div>
</div>
```

#### `update-product.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Update Product ✏️</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="successMessage" class="alert alert-success fade-in" role="alert">
    {{ successMessage }}
  </div>
  <div *ngIf="product; else loading">
    <div class="card shadow fade-in">
      <div class="card-body">
        <form (ngSubmit)="updateProduct()">
          <div class="mb-3">
            <label for="name" class="form-label">Product Name</label>
            <input type="text" class="form-control" id="name" [(ngModel)]="product.name" name="name" required maxlength="100">
          </div>
          <div class="mb-3">
            <label for="type" class="form-label">Type (Optional)</label>
            <input type="text" class="form-control" id="type" [(ngModel)]="product.type" name="type" maxlength="50">
          </div>
          <div class="mb-3">
            <label for="category" class="form-label">Category (Optional)</label>
            <input type="text" class="form-control" id="category" [(ngModel)]="product.category" name="category" maxlength="50">
          </div>
          <div class="mb-3">
            <label for="price" class="form-label">Price</label>
            <input type="number" class="form-control" id="price" [(ngModel)]="product.price" name="price" required min="0.01" step="0.01">
          </div>
          <div class="mb-3">
            <label for="stock" class="form-label">Stock</label>
            <input type="number" class="form-control" id="stock" [(ngModel)]="product.stock" name="stock" required min="0">
          </div>
          <div class="mb-3">
            <label for="description" class="form-label">Description (Optional)</label>
            <textarea class="form-control" id="description" [(ngModel)]="product.description" name="description" maxlength="5000"></textarea>
          </div>
          <div class="mb-3">
            <label for="images" class="form-label">Images URL (Optional)</label>
            <input type="text" class="form-control" id="images" [(ngModel)]="product.images" name="images">
          </div>
          <div class="mb-3">
            <label for="specifications" class="form-label">Specifications (Optional)</label>
            <input type="text" class="form-control" id="specifications" [(ngModel)]="product.specifications" name="specifications">
          </div>
          <button type="submit" class="btn-neon w-100">Update Product</button>
        </form>
      </div>
    </div>
  </div>
  <ng-template #loading>
    <div class="text-center py-5 slide-in">
      <div class="spinner-border" style="color: var(--neon-cyan);" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </ng-template>
</div>
```

#### `update-profile.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 glow">Update Profile ✏️</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div *ngIf="successMessage" class="alert alert-success fade-in" role="alert">
    {{ successMessage }}
  </div>
  <div *ngIf="user; else loading">
    <div class="card shadow fade-in">
      <div class="card-body">
        <form (ngSubmit)="updateProfile()">
          <div class="mb-3">
            <label for="firstName" class="form-label">First Name</label>
            <input type="text" class="form-control" id="firstName" [(ngModel)]="user.firstName" name="firstName" required maxlength="50">
          </div>
          <div class="mb-3">
            <label for="lastName" class="form-label">Last Name</label>
            <input type="text" class="form-control" id="lastName" [(ngModel)]="user.lastName" name="lastName" required maxlength="50">
          </div>
          <div class="mb-3">
            <label for="email" class="form-label">Email</label>
            <input type="email" class="form-control" id="email" [(ngModel)]="user.email" name="email" required>
          </div>
          <div class="mb-3">
            <label for="phone" class="form-label">Phone (Optional)</label>
            <input type="tel" class="form-control" id="phone" [(ngModel)]="user.phone" name="phone" maxlength="15">
          </div>
          <div class="mb-3" *ngIf="!user.emailVerified">
            <button type="button" class="btn-neon w-100" (click)="sendVerificationEmail()">Send Verification Email</button>
          </div>
          <button type="submit" class="btn-neon w-100">Update Profile</button>
        </form>
      </div>
    </div>
  </div>
  <ng-template #loading>
    <div class="text-center py-5 slide-in">
      <div class="spinner-border" style="color: var(--neon-cyan);" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </ng-template>
</div>
```

---

### Notes
- **Styling**: These templates use the same global styles (`btn-neon`, `glow`, `fade-in`, `tilt`, `perspective`, etc.) as the previous components, ensuring consistency with the cyberpunk theme.
- **Scripts**: Components like `product.component.html` and `product-detail.component.html` include a `<script>` tag for Canvas Confetti. Ensure the library is included in your `index.html` (via CDN or local assets).
- **Functionality**: All Angular directives (`*ngIf`, `[(ngModel)]`, `[routerLink]`, `(click)`, etc.) are included as per the project’s requirements.
- **Copy-Paste**: Each code block is formatted for easy copying. Just click into the block and copy the content to paste into your respective `.html` files.

Let me know if you need any additional components or adjustments!
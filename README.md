Bhai, bilkul! Main 10-10 files ke batch mein de sakta hoon, koi issue nahi hai. Lekin agar tujhe 5-5 ke batch mein chahiye, to woh bhi perfectly fine hai. Kaam acche se hona chahiye, yahi priority hai. Since total 52 files hain aur pehle batch mein maine 4 files de di hain (`index.html`, `styles.css`, `app.component.html`, aur `app.component.css`), ab 48 files baki hain.

Chalo, ab 5-5 ke batch mein hi proceed karte hain, taaki tu aaram se manage kar sake aur koi confusion na ho. Har batch mein 5 files doonga, aur 48 files ke liye total 10 batches banenge (last batch mein 3 files rahenge). Ready hai? Shuru karte hain!

---

### Batch 2: 5 Files (`add-address.component` aur `add-product.component` ke Files)
#### `add-address.component.html`
```html
<div class="container py-5 px-5">
  <div class="row justify-content-center">
    <div class="col-md-6">
      <div class="card shadow fade-in">
        <div class="card-header bg-primary text-white">
          <h4 class="mb-0">Add Address 📍</h4>
        </div>
        <div class="card-body">
          <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
            {{ errorMessage }}
          </div>
          <div *ngIf="successMessage" class="alert alert-success fade-in" role="alert">
            {{ successMessage }}
          </div>
          <form (ngSubmit)="onSubmit()">
            <div class="mb-3">
              <label for="houseNumber" class="form-label">House Number</label>
              <input type="number" class="form-control" id="houseNumber" [(ngModel)]="addressRequest.houseNumber" name="houseNumber" required>
            </div>
            <div class="mb-3">
              <label for="streetName" class="form-label">Street Name</label>
              <input type="text" class="form-control" id="streetName" [(ngModel)]="addressRequest.streetName" name="streetName" required>
            </div>
            <div class="mb-3">
              <label for="colonyName" class="form-label">Colony Name (Optional)</label>
              <input type="text" class="form-control" id="colonyName" [(ngModel)]="addressRequest.colonyName" name="colonyName">
            </div>
            <div class="mb-3">
              <label for="city" class="form-label">City</label>
              <input type="text" class="form-control" id="city" [(ngModel)]="addressRequest.city" name="city" required>
            </div>
            <div class="mb-3">
              <label for="state" class="form-label">State</label>
              <input type="text" class="form-control" id="state" [(ngModel)]="addressRequest.state" name="state" required>
            </div>
            <div class="mb-3">
              <label for="pincode" class="form-label">Pincode</label>
              <input type="number" class="form-control" id="pincode" [(ngModel)]="addressRequest.pincode" name="pincode" required>
            </div>
            <button type="submit" class="btn btn-primary w-100 pulse">Add Address</button>
          </form>
        </div>
      </div>
    </div>
  </div>
</div>
```

#### `add-address.component.css`
```css
.card {
  background-color: #2A2A2A;
  border: 1px solid #00DDEB;
  border-radius: 10px;
  animation: fadeIn 0.5s ease-in-out forwards;
}

.card-header {
  background-color: #333;
  color: #FFFFFF;
}

.form-control {
  background-color: #333;
  color: #E0E0E0;
  border: 1px solid #00DDEB;
  border-radius: 5px;
}

.form-control:focus {
  box-shadow: 0 0 10px #00DDEB;
  border-color: #39FF14;
}

.btn-primary {
  background-color: #00DDEB;
  border: none;
  transition: background-color 0.3s ease, box-shadow 0.3s ease;
}

.btn-primary:hover {
  background-color: #39FF14;
  box-shadow: 0 0 10px #39FF14, 0 0 20px #39FF14;
}
```

#### `add-product.component.html`
```html
<div class="container py-5 px-5">
  <div class="row justify-content-center">
    <div class="col-md-6">
      <div class="card shadow fade-in">
        <div class="card-header bg-primary text-white">
          <h4 class="mb-0">Add Product 🛍️</h4>
        </div>
        <div class="card-body">
          <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
            {{ errorMessage }}
          </div>
          <div *ngIf="successMessage" class="alert alert-success fade-in" role="alert">
            {{ successMessage }}
          </div>
          <form (ngSubmit)="onSubmit()">
            <div class="mb-3">
              <label for="name" class="form-label">Product Name</label>
              <input type="text" class="form-control" id="name" [(ngModel)]="productRequest.name" name="name" maxlength="100" required>
            </div>
            <div class="mb-3">
              <label for="type" class="form-label">Type</label>
              <input type="text" class="form-control" id="type" [(ngModel)]="productRequest.type" name="type" maxlength="50">
            </div>
            <div class="mb-3">
              <label for="category" class="form-label">Category</label>
              <input type="text" class="form-control" id="category" [(ngModel)]="productRequest.category" name="category" maxlength="50">
            </div>
            <div class="mb-3">
              <label for="price" class="form-label">Price</label>
              <input type="number" class="form-control" id="price" [(ngModel)]="productRequest.price" name="price" required min="0.01" step="0.01">
            </div>
            <div class="mb-3">
              <label for="stock" class="form-label">Stock</label>
              <input type="number" class="form-control" id="stock" [(ngModel)]="productRequest.stock" name="stock" required min="0">
            </div>
            <div class="mb-3">
              <label for="description" class="form-label">Description</label>
              <textarea class="form-control" id="description" [(ngModel)]="productRequest.description" name="description" maxlength="500"></textarea>
            </div>
            <div class="mb-3">
              <label for="images" class="form-label">Images URL</label>
              <input type="text" class="form-control" id="images" [(ngModel)]="productRequest.images" name="images">
            </div>
            <div class="mb-3">
              <label for="specifications" class="form-label">Specifications</label>
              <input type="text" class="form-control" id="specifications" [(ngModel)]="productRequest.specifications" name="specifications" maxlength="500">
            </div>
            <button type="submit" class="btn btn-primary w-100 pulse">Add Product</button>
          </form>
        </div>
      </div>
    </div>
  </div>
</div>
```

#### `add-product.component.css`
```css
.card {
  background-color: #2A2A2A;
  border: 1px solid #00DDEB;
  border-radius: 10px;
  animation: fadeIn 0.5s ease-in-out forwards;
}

.card-header {
  background-color: #333;
  color: #FFFFFF;
}

.form-control {
  background-color: #333;
  color: #E0E0E0;
  border: 1px solid #00DDEB;
  border-radius: 5px;
}

.form-control:focus {
  box-shadow: 0 0 10px #00DDEB;
  border-color: #39FF14;
}

.btn-primary {
  background-color: #00DDEB;
  border: none;
  transition: background-color 0.3s ease, box-shadow 0.3s ease;
}

.btn-primary:hover {
  background-color: #39FF14;
  box-shadow: 0 0 10px #39FF14, 0 0 20px #39FF14;
}
```

#### `address.component.html`
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4 fade-in">My Addresses 📍</h2>
  <div *ngIf="errorMessage" class="alert alert-danger fade-in" role="alert">
    {{ errorMessage }}
  </div>
  <div class="row">
    <div class="col-md-4 mb-3" *ngFor="let address of addresses">
      <div class="card shadow fade-in">
        <div class="card-body">
          <p class="mb-1"><strong>Address:</strong> {{ address.houseNumber }}, {{ address.streetName }}, {{ address.colonyName || '' }}</p>
          <p class="mb-1">{{ address.city }}, {{ address.state }} - {{ address.pincode }}</p>
          <div class="d-flex gap-2 mt-2">
            <button class="btn btn-primary" [routerLink]="['/update-address', address.id]">Update</button>
            <button class="btn btn-danger" (click)="deleteAddress(address.id)">Delete</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

### Progress Update
Ab tak maine tujhe total 9 files de di hain:
- Batch 1: `index.html`, `styles.css`, `app.component.html`, `app.component.css` (4 files)
- Batch 2: `add-address.component.html`, `add-address.component.css`, `add-product.component.html`, `add-product.component.css`, `address.component.html` (5 files)

**Baki files:** 52 - 9 = 43 files abhi baki hain. Agle batch mein 5 aur files doonga, jisme `address.component.css` aur agle components ke files shamil honge.

**Question:** Yeh 5 files copy kar li hain? Koi issue to nahi hai? Agla batch jaldi se doon, ya thoda wait karun? 😊
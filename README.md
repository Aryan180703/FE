**It’s 4:10 PM IST on Sunday, May 18, 2025.**

Thank you for sharing the backend files and DTOs. I’ll update the necessary files to implement the `Delivery Agent` functionality with the following requirements:
- Add `PlacedAt`, `ShippedAt`, `DeliveredAt`, and `CancelledAt` timestamps to the `Order` model.
- Update `OrderService` to set these timestamps when the status changes.
- Add `OrderDate`, `PlacedAt`, `ShippedAt`, `DeliveredAt`, and `CancelledAt` to `DeliveryAgentOrderResponse` DTO.
- Add status transition validation in `UpdateOrderStatusAsync`.
- Optimize `GetAllOrdersForDeliveryAgentAsync` by moving filtering and sorting to the repository.
- Since all `Delivery Agents` can modify all non-`Delivered` orders, we won’t add `DeliveryAgentId` or assignment logic.

Let’s update the files step-by-step.

---

### Step 1: Update `Order.cs`
**Add timestamp fields to track status changes.**

**`Models/Order.cs` (Updated)**:
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Threading.Tasks;

namespace EShoppingZone.Models
{
    public class Order
    {
        [Key]
        public int Id { get; set; }
        [Required]
        public int CustomerId { get; set; }
        [Required]
        public int AddressId { get; set; }
        [Required]
        public decimal TotalPrice { get; set; }
        public string Status { get; set; } = "Placed";
        public DateTime OrderDate { get; set; }
        // New timestamp fields
        public DateTime? PlacedAt { get; set; }
        public DateTime? ShippedAt { get; set; }
        public DateTime? DeliveredAt { get; set; }
        public DateTime? CancelledAt { get; set; }
        public UserProfile Customer { get; set; } = null!;
        public Address Address { get; set; } = null!;
        public List<OrderItem> Items { get; set; } = new();
    }
}
```

- **Changes**:
  - Added `PlacedAt`, `ShippedAt`, `DeliveredAt`, and `CancelledAt` as nullable `DateTime` fields.

**Migration Command**:
Run the following to update the database schema:
```bash
dotnet ef migrations add AddOrderTimestamps
dotnet ef database update
```

---

### Step 2: Update `DeliveryAgentOrderResponse.cs`
**Add `OrderDate` and the new timestamp fields to the DTO.**

**`DTOs/OrderDTOs/DeliveryAgentOrderResponse.cs` (Updated)**:
```csharp
using System;
using EShoppingZone.DTOs.AddressDTOs;

namespace EShoppingZone.DTOs.OrderDTOs
{
    public class DeliveryAgentOrderResponse
    {
        public int Id { get; set; }
        public AddressResponse Address { get; set; }
        public string Status { get; set; } = string.Empty;
        public DateTime OrderDate { get; set; }
        public DateTime? PlacedAt { get; set; }
        public DateTime? ShippedAt { get; set; }
        public DateTime? DeliveredAt { get; set; }
        public DateTime? CancelledAt { get; set; }
    }
}
```

- **Changes**:
  - Added `OrderDate`, `PlacedAt`, `ShippedAt`, `DeliveredAt`, and `CancelledAt` to the DTO.

---

### Step 3: Update `AutoMapperProfile.cs`
**Ensure AutoMapper maps the new fields.**

**`Mapping/AutoMapperProfile.cs` (Updated)**:
```csharp
using AutoMapper;
using EShoppingZone.DTOs.OrderDTOs;
using EShoppingZone.Models;

namespace EShoppingZone.Mapping
{
    public class AutoMapperProfile : Profile
    {
        public AutoMapperProfile()
        {
            CreateMap<Order, OrderResponse>();
            CreateMap<OrderItem, OrderItemResponse>();

            CreateMap<Order, DeliveryAgentOrderResponse>()
                .ForMember(dest => dest.OrderDate, opt => opt.MapFrom(src => src.OrderDate))
                .ForMember(dest => dest.PlacedAt, opt => opt.MapFrom(src => src.PlacedAt))
                .ForMember(dest => dest.ShippedAt, opt => opt.MapFrom(src => src.ShippedAt))
                .ForMember(dest => dest.DeliveredAt, opt => opt.MapFrom(src => src.DeliveredAt))
                .ForMember(dest => dest.CancelledAt, opt => opt.MapFrom(src => src.CancelledAt));
        }
    }
}
```

- **Changes**:
  - Added mappings for `OrderDate`, `PlacedAt`, `ShippedAt`, `DeliveredAt`, and `CancelledAt`.

---

### Step 4: Update `IOrderService.cs`
**Adjust the `GetAllOrdersForDeliveryAgentAsync` method signature to remove the `deliveryAgentId` parameter since all `Delivery Agents` can access all orders.**

**`Interfaces/IOrderService.cs` (Updated)**:
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.DTOs.OrderDTOs;

namespace EShoppingZone.Interfaces
{
    public interface IOrderService
    {
        Task<ResponseDTO<OrderResponse>> PlaceOrderAsync(int profileId, OrderRequest orderRequest);
        Task<ResponseDTO<OrderResponse>> GetOrderAsync(int profileId, int orderId);
        Task<ResponseDTO<List<OrderResponse>>> GetAllOrdersAsync(int profileId);
        Task<ResponseDTO<OrderResponse>> UpdateOrderStatusAsync(int profileId, int orderId, UpdateOrderStatusRequest updateRequest);
        Task<ResponseDTO<OrderResponse>> CancelOrderAsync(int profileId, int orderId);
        Task<ResponseDTO<List<DeliveryAgentOrderResponse>>> GetAllOrdersForDeliveryAgentAsync();
    }
}
```

- **Changes**:
  - Ensured `GetAllOrdersForDeliveryAgentAsync` doesn’t take a `deliveryAgentId` parameter.

---

### Step 5: Update `IOrderRepository.cs`
**Add a new method to fetch non-`Delivered` orders with sorting.**

**`Interfaces/IOrderRepository.cs` (Updated)**:
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using EShoppingZone.Models;

namespace EShoppingZone.Interfaces
{
    public interface IOrderRepository
    {
        Task<Cart> GetCartAsync(int cartId);
        Task<Address> GetAddressAsync(int addressId);
        Task<Product> GetProductAsync(int productId);
        Task<Order> CreateOrderAsync(Order order);
        Task ClearCartAsync(Cart cart);
        Task<Order> GetOrderAsync(int orderId);
        Task<List<Order>> GetAllOrdersAsync(int profileId, bool isMerchant);
        Task UpdateOrderAsync(Order order);
        Task<bool> IsMerchantAsync(int profileId);
        Task<bool> IsDeliveryAgentAsync(int profileId);
        Task<List<Order>> GetAllOrdersForDeliveryAgentAsync();
        Task<List<Order>> GetNonDeliveredOrdersForDeliveryAgentAsync(); // New method
    }
}
```

- **Changes**:
  - Added `GetNonDeliveredOrdersForDeliveryAgentAsync` to fetch non-`Delivered` orders with sorting.

---

### Step 6: Update `OrderRepository.cs`
**Implement the new method to fetch non-`Delivered` orders, sorted by `OrderDate`.**

**`Repositories/OrderRepository.cs` (Updated)**:
```csharp
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using EShoppingZone.Data;
using EShoppingZone.Interfaces;
using EShoppingZone.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Repositories
{
    public class OrderRepository : IOrderRepository
    {
        private readonly EShoppingZoneDBContext _context;
        private readonly RoleManager<IdentityRole<int>> _roleManager;

        public OrderRepository(EShoppingZoneDBContext context, RoleManager<IdentityRole<int>> roleManager)
        {
            _context = context;
            _roleManager = roleManager;
        }

        public async Task<Order> CreateOrderAsync(Order order)
        {
            await _context.Orders.AddAsync(order);
            await _context.SaveChangesAsync();
            return order;
        }

        public async Task<Order> GetOrderAsync(int orderId)
        {
            return await _context.Orders
                .Include(o => o.Items)
                .Include(o => o.Address)
                .FirstOrDefaultAsync(o => o.Id == orderId);
        }

        public async Task<List<Order>> GetAllOrdersAsync(int profileId, bool isMerchant)
        {
            if (isMerchant)
            {
                return await _context.Orders
                    .Include(o => o.Items)
                    .Include(o => o.Address)
                    .ToListAsync();
            }
            return await _context.Orders
                .Include(o => o.Items)
                .Include(o => o.Address)
                .Where(o => o.CustomerId == profileId)
                .ToListAsync();
        }

        public async Task UpdateOrderAsync(Order order)
        {
            _context.Orders.Update(order);
            await _context.SaveChangesAsync();
        }

        public async Task<Cart> GetCartAsync(int cartId)
        {
            return await _context.Carts
                .Include(c => c.Items)
                .FirstOrDefaultAsync(c => c.Id == cartId);
        }

        public async Task<Address> GetAddressAsync(int addressId)
        {
            return await _context.Addresses.FindAsync(addressId);
        }

        public async Task<Product> GetProductAsync(int productId)
        {
            return await _context.Products.FindAsync(productId);
        }

        public async Task ClearCartAsync(Cart cart)
        {
            cart.Items.Clear();
            cart.TotalPrice = 0;
            _context.Carts.Update(cart);
            await _context.SaveChangesAsync();
        }

        public async Task<bool> IsMerchantAsync(int profileId)
        {
            var userRoles = await _context.UserRoles
                .Where(ur => ur.UserId == profileId)
                .Select(ur => ur.RoleId)
                .ToListAsync();
            var merchantRole = await _roleManager.FindByNameAsync("Merchant");
            return userRoles.Contains(merchantRole.Id);
        }

        public async Task<bool> IsDeliveryAgentAsync(int profileId)
        {
            var userRoles = await _context.UserRoles
                .Where(ur => ur.UserId == profileId)
                .Select(ur => ur.RoleId)
                .ToListAsync();
            var deliveryAgentRole = await _roleManager.FindByNameAsync("Delivery Agent");
            return userRoles.Contains(deliveryAgentRole.Id);
        }

        public async Task<List<Order>> GetAllOrdersForDeliveryAgentAsync()
        {
            return await _context.Orders
                .Include(o => o.Address)
                .Include(o => o.Items)
                .ToListAsync();
        }

        public async Task<List<Order>> GetNonDeliveredOrdersForDeliveryAgentAsync()
        {
            return await _context.Orders
                .Include(o => o.Address)
                .Include(o => o.Items)
                .Where(o => o.Status != "Delivered")
                .OrderByDescending(o => o.OrderDate)
                .ToListAsync();
        }
    }
}
```

- **Changes**:
  - Implemented `GetNonDeliveredOrdersForDeliveryAgentAsync` to fetch non-`Delivered` orders, sorted by `OrderDate` (most recent to oldest).
  - Added `Include(o => o.Address)` in `GetOrderAsync` and `GetAllOrdersAsync` to ensure the `Address` is always loaded (required for mapping to `DeliveryAgentOrderResponse`).

---

### Step 7: Update `OrderService.cs`
**Set timestamps, add status transition validation, and optimize `GetAllOrdersForDeliveryAgentAsync`.**

**`Services/OrderService.cs` (Updated)**:
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using AutoMapper;
using EShoppingZone.Data;
using EShoppingZone.DTOs;
using EShoppingZone.DTOs.AddressDTOs;
using EShoppingZone.DTOs.OrderDTOs;
using EShoppingZone.Interfaces;
using EShoppingZone.Models;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Services
{
    public class OrderService : IOrderService
    {
        private readonly IOrderRepository _repository;
        private readonly IMapper _mapper;
        private readonly EShoppingZoneDBContext _context;

        public OrderService(IOrderRepository repository, IMapper mapper, EShoppingZoneDBContext context)
        {
            _repository = repository;
            _mapper = mapper;
            _context = context;
        }

        public async Task<ResponseDTO<OrderResponse>> PlaceOrderAsync(int profileId, OrderRequest orderRequest)
        {
            var cart = await _repository.GetCartAsync(orderRequest.CartId);
            if (cart == null || cart.UserProfileId != profileId || !cart.Items.Any())
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Invalid or empty cart"
                };
            }

            var address = await _repository.GetAddressAsync(orderRequest.AddressId);
            if (address == null || address.UserProfileId != profileId)
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Invalid address"
                };
            }

            foreach (var item in cart.Items)
            {
                var product = await _repository.GetProductAsync(item.ProductId);
                if (product == null || product.Stock < item.Quantity)
                {
                    return new ResponseDTO<OrderResponse>
                    {
                        Success = false,
                        Message = $"Insufficient stock for product: {item.ProductName}"
                    };
                }
            }

            var order = new Order
            {
                CustomerId = profileId,
                AddressId = orderRequest.AddressId,
                TotalPrice = cart.TotalPrice,
                Status = "Placed",
                OrderDate = DateTime.UtcNow,
                PlacedAt = DateTime.UtcNow // Set PlacedAt timestamp
            };

            foreach (var item in cart.Items)
            {
                var product = await _repository.GetProductAsync(item.ProductId);
                if (product == null)
                {
                    return new ResponseDTO<OrderResponse>
                    {
                        Success = false,
                        Message = $"Product {item.ProductId} not found"
                    };
                }
                order.Items.Add(new OrderItem
                {
                    ProductId = item.ProductId,
                    ProductName = item.ProductName,
                    Price = item.Price,
                    Quantity = item.Quantity
                });
            }

            foreach (var item in cart.Items)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                product.Stock -= item.Quantity;
                _context.Products.Update(product);
            }

            var createdOrder = await _repository.CreateOrderAsync(order);
            await _repository.ClearCartAsync(cart);

            var response = await GetOrderResponseAsync(createdOrder);
            return new ResponseDTO<OrderResponse>
            {
                Success = true,
                Message = "Order placed successfully",
                Data = response
            };
        }

        public async Task<ResponseDTO<OrderResponse>> GetOrderAsync(int profileId, int orderId)
        {
            var order = await _repository.GetOrderAsync(orderId);
            if (order == null || (order.CustomerId != profileId && !await _repository.IsMerchantAsync(profileId)))
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Order not found or unauthorized"
                };
            }

            var response = await GetOrderResponseAsync(order);
            return new ResponseDTO<OrderResponse>
            {
                Success = true,
                Message = "Order retrieved",
                Data = response
            };
        }

        public async Task<ResponseDTO<List<OrderResponse>>> GetAllOrdersAsync(int profileId)
        {
            var isMerchant = await _repository.IsMerchantAsync(profileId);
            var orders = await _repository.GetAllOrdersAsync(profileId, isMerchant);

            if (!orders.Any())
            {
                return new ResponseDTO<List<OrderResponse>>
                {
                    Success = false,
                    Message = "No orders found"
                };
            }

            var response = new List<OrderResponse>();
            foreach (var order in orders)
            {
                response.Add(await GetOrderResponseAsync(order));
            }

            return new ResponseDTO<List<OrderResponse>>
            {
                Success = true,
                Message = "Orders retrieved",
                Data = response
            };
        }

        public async Task<ResponseDTO<OrderResponse>> UpdateOrderStatusAsync(int profileId, int orderId, UpdateOrderStatusRequest updateRequest)
        {
            var isDeliveryAgent = await _repository.IsDeliveryAgentAsync(profileId);
            if (!isDeliveryAgent)
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Unauthorized"
                };
            }

            var order = await _repository.GetOrderAsync(orderId);
            if (order == null)
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Order not found"
                };
            }

            var validStatuses = new[] { "Placed", "Shipped", "Delivered", "Cancelled" };
            if (!validStatuses.Contains(updateRequest.Status))
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Invalid status"
                };
            }

            // Validate status transitions
            if (order.Status == "Cancelled")
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Cannot change status of a cancelled order"
                };
            }

            if (order.Status == "Delivered")
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Cannot change status of a delivered order"
                };
            }

            if (order.Status == "Placed" && updateRequest.Status != "Shipped" && updateRequest.Status != "Cancelled")
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Placed orders can only be changed to Shipped or Cancelled"
                };
            }

            if (order.Status == "Shipped" && updateRequest.Status != "Delivered" && updateRequest.Status != "Cancelled")
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Shipped orders can only be changed to Delivered or Cancelled"
                };
            }

            // Set the appropriate timestamp based on the new status
            order.Status = updateRequest.Status;
            switch (updateRequest.Status)
            {
                case "Placed":
                    order.PlacedAt = DateTime.UtcNow;
                    break;
                case "Shipped":
                    order.ShippedAt = DateTime.UtcNow;
                    break;
                case "Delivered":
                    order.DeliveredAt = DateTime.UtcNow;
                    break;
                case "Cancelled":
                    order.CancelledAt = DateTime.UtcNow;
                    break;
            }

            await _repository.UpdateOrderAsync(order);

            var response = await GetOrd
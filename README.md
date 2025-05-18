Bhai, samajh gaya! 😊 Tune bola ki role request se related saare DTOs, `RoleRequestController`, service, aur repository ke code chahiye. Chuki hum EShoppingZone project pe kaam kar rahe hain aur tune migration (`AddRoleRequestTable`) already apply kar diya hai, main ab sirf role request feature ke liye relevant files doonga. Yeh files backend ke hain (.NET 8 Web API with ASP.NET Core Identity), aur hum `UserId` ko `int` use kar rahe hain (jaise tune migration ke time fix kiya tha).

---

### **Role Request se Related Files**

#### **DTOs**
##### **File 1: `RoleRequestDTO.cs`**
**Path: `EShoppingZone.Models/RoleRequestDTO.cs`**  
(Yeh DTO role request submit karne ke liye hai, jab customer "Become a Merchant" ya "Become a Delivery Agent" button click karta hai.)
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequestDTO
    {
        public string RequestedRole { get; set; } = string.Empty; // Merchant or DeliveryAgent
    }
}
```

##### **File 2: `RoleRequestResponseDTO.cs`**
**Path: `EShoppingZone.Models/RoleRequestResponseDTO.cs`**  
(Yeh DTO admin ke liye hai jab woh role request ko approve ya reject karta hai.)
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequestResponseDTO
    {
        public int RequestId { get; set; }
        public string Status { get; set; } = string.Empty; // Approved or Rejected
    }
}
```

#### **Controller**
##### **File 3: `RoleRequestController.cs`**
**Path: `EShoppingZone.Controllers/RoleRequestController.cs`**  
(Yeh controller role request ke endpoints handle karta hai: submit request, pending requests dekhna, approve/reject request, aur user ka apna request status check karna.)
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using EShoppingZone.Models;
using EShoppingZone.Services;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class RoleRequestController : ControllerBase
    {
        private readonly IRepository _repository;
        private readonly IEmailService _emailService;
        private readonly UserManager<UserProfile> _userManager;
        private readonly EShoppingZoneDbContext _context;

        public RoleRequestController(
            IRepository repository,
            IEmailService emailService,
            UserManager<UserProfile> userManager,
            EShoppingZoneDbContext context)
        {
            _repository = repository;
            _emailService = emailService;
            _userManager = userManager;
            _context = context;
        }

        [HttpPost("SubmitRoleRequest")]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> SubmitRoleRequest([FromBody] RoleRequestDTO request)
        {
            if (request.RequestedRole != "Merchant" && request.RequestedRole != "DeliveryAgent")
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid role. Only Merchant or DeliveryAgent allowed.",
                });
            }

            var userIdString = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userIdString) || !int.TryParse(userIdString, out int userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated or invalid user ID.",
                });
            }

            var existingRequest = await _repository.GetRoleRequestByUserIdAsync(userId);
            if (existingRequest != null)
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "You already have a pending role request.",
                });
            }

            var roleRequest = new RoleRequest
            {
                UserId = userId,
                RequestedRole = request.RequestedRole,
                Status = "Pending",
                RequestedAt = DateTime.UtcNow,
            };

            await _repository.AddRoleRequestAsync(roleRequest);

            await _emailService.SendEmailAsync(
                User.Identity.Name,
                $"Your application to become a {request.RequestedRole} has been accepted. We will review it and get back to you soon.",
                "Role Request Submitted"
            );

            return Ok(new ResponseDTO<string>
            {
                Success = true,
                Message = "Role request submitted successfully.",
            });
        }

        [HttpGet("PendingRequests")]
        [Authorize(Roles = "Admin")]
        public async Task<IActionResult> GetPendingRequests()
        {
            var requests = await _repository.GetPendingRoleRequestsAsync();
            return Ok(new ResponseDTO<List<RoleRequest>>
            {
                Success = true,
                Message = "Pending role requests retrieved successfully.",
                Data = requests
            });
        }

        [HttpPost("ReviewRequest")]
        [Authorize(Roles = "Admin")]
        public async Task<IActionResult> ReviewRequest([FromBody] RoleRequestResponseDTO response)
        {
            if (response.Status != "Approved" && response.Status != "Rejected")
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid status. Status must be 'Approved' or 'Rejected'.",
                });
            }

            var roleRequest = await _context.RoleRequests
                .Include(rr => rr.User)
                .FirstOrDefaultAsync(rr => rr.Id == response.RequestId);
            if (roleRequest == null)
            {
                return NotFound(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Role request not found.",
                });
            }

            roleRequest.Status = response.Status;
            roleRequest.ReviewedAt = DateTime.UtcNow;
            await _repository.UpdateRoleRequestAsync(roleRequest);

            if (response.Status == "Approved")
            {
                var user = roleRequest.User;
                var existingRoles = await _userManager.GetRolesAsync(user);
                await _userManager.RemoveFromRolesAsync(user, existingRoles);
                await _userManager.AddToRoleAsync(user, roleRequest.RequestedRole);

                await _emailService.SendEmailAsync(
                    user.Email,
                    $"Congratulations! Your request to become a {roleRequest.RequestedRole} has been approved!",
                    "Role Request Approved"
                );
            }
            else
            {
                await _emailService.SendEmailAsync(
                    roleRequest.User.Email,
                    "We regret to inform you that your request to become a " +
                    $"{roleRequest.RequestedRole} has been rejected. " +
                    "Currently, we are looking to expand our team in other areas. " +
                    "Thank you for your interest!",
                    "Role Request Rejected"
                );
            }

            return Ok(new ResponseDTO<string>
            {
                Success = true,
                Message = $"Role request {response.Status.ToLower()} successfully.",
            });
        }

        [HttpGet("MyRequest")]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> GetMyRoleRequest()
        {
            var userIdString = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userIdString) || !int.TryParse(userIdString, out int userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated or invalid user ID.",
                });
            }

            var roleRequest = await _repository.GetRoleRequestByUserIdAsync(userId);
            if (roleRequest == null)
            {
                return Ok(new ResponseDTO<RoleRequest>
                {
                    Success = true,
                    Message = "No pending role request found.",
                    Data = null
                });
            }

            return Ok(new ResponseDTO<RoleRequest>
            {
                Success = true,
                Message = "Role request retrieved successfully.",
                Data = roleRequest
            });
        }
    }
}
```

#### **Repository**
##### **File 4: `IRepository.cs`**
**Path: `EShoppingZone.Services/IRepository.cs`**  
(Yeh interface repository ke methods define karta hai, jo role request ke liye use hote hain.)
```csharp
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public interface IRepository
    {
        Task<UserProfile> GetUserByEmailAsync(string email);
        Task AddUserAsync(UserProfile user);
        Task AddRoleRequestAsync(RoleRequest roleRequest);
        Task<RoleRequest> GetRoleRequestByUserIdAsync(int userId);
        Task<List<RoleRequest>> GetPendingRoleRequestsAsync();
        Task UpdateRoleRequestAsync(RoleRequest roleRequest);
    }
}
```

##### **File 5: `Repository.cs`**
**Path: `EShoppingZone.Services/Repository.cs`**  
(Yeh repository implementation hai jo database operations handle karta hai role requests ke liye.)
```csharp
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Data;
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public class Repository : IRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public Repository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<UserProfile> GetUserByEmailAsync(string email)
        {
            return await _context.Profiles.FirstOrDefaultAsync(u => u.Email == email);
        }

        public async Task AddUserAsync(UserProfile user)
        {
            await _context.Profiles.AddAsync(user);
            await _context.SaveChangesAsync();
        }

        public async Task AddRoleRequestAsync(RoleRequest roleRequest)
        {
            await _context.RoleRequests.AddAsync(roleRequest);
            await _context.SaveChangesAsync();
        }

        public async Task<RoleRequest> GetRoleRequestByUserIdAsync(int userId)
        {
            return await _context.RoleRequests
                .FirstOrDefaultAsync(rr => rr.UserId == userId && rr.Status == "Pending");
        }

        public async Task<List<RoleRequest>> GetPendingRoleRequestsAsync()
        {
            return await _context.RoleRequests
                .Where(rr => rr.Status == "Pending")
                .Include(rr => rr.User)
                .ToListAsync();
        }

        public async Task UpdateRoleRequestAsync(RoleRequest roleRequest)
        {
            _context.RoleRequests.Update(roleRequest);
            await _context.SaveChangesAsync();
        }
    }
}
```

#### **Service**
Humne role request ke liye koi separate service layer nahi banayi hai, kyunki `RoleRequestController` directly `IRepository` aur `UserManager` ka use karta hai. Agar tujhe ek dedicated service layer chahiye (jaise `IRoleRequestService`), toh bolna, main bana doonga. Abhi ke liye, yeh kaam `RoleRequestController` aur `Repository` ke through ho raha hai.

---

### **Summary**
- **DTOs**:
  - `RoleRequestDTO.cs`: Role request submit karne ke liye.
  - `RoleRequestResponseDTO.cs`: Admin ke liye approve/reject ke liye.
- **Controller**:
  - `RoleRequestController.cs`: Saare role request endpoints (`SubmitRoleRequest`, `PendingRequests`, `ReviewRequest`, `MyRequest`).
- **Repository**:
  - `IRepository.cs`: Interface with role request methods.
  - `Repository.cs`: Database operations for role requests.
- **Service**:
  - Abhi service layer nahi hai, `RoleRequestController` directly repository ka use karta hai.

---

### **Test the Backend**
- Backend run karo:
  ```bash
  dotnet run
  ```
- Ek customer user ke saath signup aur login karo:
  - `POST /api/AuthController/Signup`
  - `POST /api/AuthController/Login`
- Role request submit karo:
  - `POST /api/RoleRequest/SubmitRoleRequest`:
    ```json
    {
      "RequestedRole": "Merchant"
    }
    ```
- Check pending request:
  - `GET /api/RoleRequest/MyRequest` (Customer user ke saath)
- Admin ke saath approve/reject karo:
  - `GET /api/RoleRequest/PendingRequests`
  - `POST /api/RoleRequest/ReviewRequest`

Bhai, yeh role request ke saare relevant files hain. Test karke batana kaisa chala! 😊 Agar kuch aur chahiye ya frontend pe jana hai, bolna! 🙏
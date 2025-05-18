Bhai, theek hai, chalo backend ke changes pe discuss karte hain. 😊 Tera mentor bol raha hai ki registration ke time role nahi puchna hai, by default har user jo register karta hai woh `Customer` hoga. Uske baad, logged-in customers ke liye do options chahiye: "Become a Merchant" aur "Become a Delivery Agent". Jab customer yeh request karega, toh ek request admin ko jani chahiye, aur admin verify karke confirm ya reject kar sake. Jab tak request pending hai, customer ko "Become a Merchant" ya "Become a Delivery Agent" pe click karne pe ek dialog box dikhna chahiye jo bole "Your application is being reviewed, stay tuned". Jab admin confirm karega, toh customer ka role update ho jana chahiye (jaise `Merchant` ya `DeliveryAgent`), aur emails bhi bhejne hain—pehle ek application acceptance email, phir confirmation ya rejection ke baad ek congratulatory ya rejection email.

Chuki maine tera pura backend banaya tha, mujhe project ke structure ka ache se pata hai. Humara backend .NET 8 Web API pe based hai, aur hum ASP.NET Core Identity ke saath `UserProfile` (jo `AspNetUsers` table ko map karta hai `Profiles` naam se) use kar rahe hain. Emails ke liye hum RabbitMQ queues use karte hain (`user-events` queue ke through). Toh chalo, step-by-step backend mein kya changes karne hain, discuss karte hain.

---

### **Backend Changes Overview**
1. **Remove Role Selection at Registration**: Registration ke time role puchne ki zarurat nahi, by default role `Customer` set karenge.
2. **Add Role Request Mechanism**: Ek naya table banayenge (`RoleRequests`) jo customer ki role change requests ko store karega (e.g., `Customer` se `Merchant` ya `DeliveryAgent` banne ki request).
3. **Admin Approval Workflow**: Admin ke liye endpoints banayenge jo role requests ko approve ya reject kar sake.
4. **Email Notifications**: RabbitMQ ke through teen emails bhejne hain:
   - Application acceptance email ("Your application is accepted, we will review it...").
   - Approval email ("Congratulations, your request to become a Merchant/Delivery Agent has been approved!").
   - Rejection email ("Currently we are looking to...").
5. **Role Update Logic**: Jab admin approve karega, toh user ka role update ho jana chahiye (`Customer` se `Merchant` ya `DeliveryAgent`).

---

### **Step 1: Remove Role Selection at Registration**
Pehle hum registration flow se role selection hata denge. Currently, `AuthController` ke `Signup` endpoint aur `AuthService` mein role user se pucha ja raha hai frontend ke through, aur `SignupRequest` DTO mein bhi `Role` field hai. Isko hata ke by default `Customer` set karenge.

#### **Update `SignupRequest` DTO**
**File: `EShoppingZone.Models/SignupRequest.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class SignupRequest
    {
        public string Email { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
        // Role hata diya, ab by default Customer set hoga
    }
}
```

#### **Update `AuthService`**
**File: `EShoppingZone.Services/AuthService.cs`**
`SignupAsync` method mein role ko by default `Customer` set karenge.

```csharp
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using EShoppingZone.Models;
using Microsoft.AspNetCore.Identity;

namespace EShoppingZone.Services
{
    public class AuthService : IAuthService
    {
        private readonly UserManager<UserProfile> _userManager;
        private readonly IRepository _repository;
        private readonly string _jwtSecret = "YourSuperSecretKey1234567890!@#$%^&*()";
        private readonly int _jwtExpirationInMinutes = 60;

        public AuthService(UserManager<UserProfile> userManager, IRepository repository)
        {
            _userManager = userManager;
            _repository = repository;
        }

        public async Task<ResponseDTO<string>> SignupAsync(SignupRequest request)
        {
            var existingUser = await _userManager.FindByEmailAsync(request.Email);
            if (existingUser != null)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Email already exists",
                };
            }

            var user = new UserProfile
            {
                UserName = request.Email,
                Email = request.Email,
            };

            var result = await _userManager.CreateAsync(user, request.Password);
            if (!result.Succeeded)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = string.Join(", ", result.Errors.Select(e => e.Description)),
                };
            }

            // By default Customer role assign karo
            await _userManager.AddToRoleAsync(user, "Customer");

            var token = GenerateJwtToken(user);
            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Signup successful",
                Data = token,
            };
        }

        public async Task<ResponseDTO<string>> LoginAsync(LoginRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null || !await _userManager.CheckPasswordAsync(user, request.Password))
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid email or password",
                };
            }

            var token = GenerateJwtToken(user);
            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Login successful",
                Data = token,
            };
        }

        private string GenerateJwtToken(UserProfile user)
        {
            var claims = new List<Claim>
            {
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            };

            var roles = _userManager.GetRolesAsync(user).Result;
            claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSecret));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
            var token = new JwtSecurityToken(
                issuer: null,
                audience: null,
                claims: claims,
                expires: DateTime.Now.AddMinutes(_jwtExpirationInMinutes),
                signingCredentials: creds);

            return new JwtSecurityTokenHandler().WriteToken(token);
        }
    }
}
```

- **Changes**:
  - `SignupRequest` se `Role` field hata diya.
  - `SignupAsync` mein role by default `"Customer"` set kar diya using `_userManager.AddToRoleAsync`.

#### **Update `AuthController`**
**File: `EShoppingZone.Controllers/AuthController.cs`**
`Signup` endpoint mein bhi role parameter hata denge.

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;
using EShoppingZone.Services;
using EShoppingZone.Models;

namespace EShoppingZone.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class AuthController : ControllerBase
    {
        private readonly IAuthService _authService;

        public AuthController(IAuthService authService)
        {
            _authService = authService;
        }

        [HttpPost("Login")]
        public async Task<IActionResult> Login([FromBody] LoginRequest request)
        {
            var response = await _authService.LoginAsync(request);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpPost("Signup")]
        public async Task<IActionResult> Signup([FromBody] SignupRequest request)
        {
            var response = await _authService.SignupAsync(request);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }
    }
}
```

- **Changes**:
  - `Signup` endpoint ab `SignupRequest` ke through sirf `Email` aur `Password` lega, `Role` nahi.

---

### **Step 2: Add Role Request Mechanism**
Ab hum ek naya table banayenge `RoleRequests` jo role change requests ko store karega (e.g., `Customer` se `Merchant` ya `DeliveryAgent` banne ki request). Is table mein fields honge:
- `Id`: Primary key.
- `UserId`: User ka ID (foreign key to `Profiles` table).
- `RequestedRole`: Requested role (`Merchant` ya `DeliveryAgent`).
- `Status`: Request ka status (`Pending`, `Approved`, `Rejected`).
- `RequestedAt`: Request submit hone ka timestamp.
- `ReviewedAt`: Admin ne review kiya toh uska timestamp (optional, null if not reviewed).

#### **Create `RoleRequest` Model**
**File: `EShoppingZone.Models/RoleRequest.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequest
    {
        public int Id { get; set; }
        public string UserId { get; set; } = string.Empty;
        public string RequestedRole { get; set; } = string.Empty; // Merchant ya DeliveryAgent
        public string Status { get; set; } = "Pending"; // Pending, Approved, Rejected
        public DateTime RequestedAt { get; set; }
        public DateTime? ReviewedAt { get; set; }
        public UserProfile User { get; set; } // Navigation property
    }
}
```

#### **Update `UserProfile` Model**
`UserProfile` mein `RoleRequests` ka navigation property add karenge.

**File: `EShoppingZone.Models/UserProfile.cs`**
```csharp
using Microsoft.AspNetCore.Identity;

namespace EShoppingZone.Models
{
    public class UserProfile : IdentityUser
    {
        public string? Image { get; set; }
        public List<RoleRequest> RoleRequests { get; set; } = new List<RoleRequest>();
    }
}
```

#### **Update DbContext**
`RoleRequests` table ko `EShoppingZoneDbContext` mein add karenge.

**File: `EShoppingZone.Data/EShoppingZoneDbContext.cs`**
```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Models;

namespace EShoppingZone.Data
{
    public class EShoppingZoneDbContext : IdentityDbContext<UserProfile>
    {
        public EShoppingZoneDbContext(DbContextOptions<EShoppingZoneDbContext> options)
            : base(options)
        {
        }

        public DbSet<RoleRequest> RoleRequests { get; set; }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);

            builder.Entity<UserProfile>()
                .ToTable("Profiles");

            builder.Entity<RoleRequest>()
                .HasOne(rr => rr.User)
                .WithMany(u => u.RoleRequests)
                .HasForeignKey(rr => rr.UserId);
        }
    }
}
```

- **Changes**:
  - `DbSet<RoleRequest>` add kiya.
  - `OnModelCreating` mein relationship define kiya `UserProfile` aur `RoleRequest` ke beech.

#### **Create Migration**
Naye table ke liye migration banani hogi:
```bash
dotnet ef migrations add AddRoleRequestTable
dotnet ef database update
```

---

### **Step 3: Add Role Request Endpoint**
Ab ek endpoint banayenge jahan se customer role change request submit kar sake (`Become a Merchant` ya `Become a Delivery Agent`).

#### **Create `RoleRequestDTO`**
**File: `EShoppingZone.Models/RoleRequestDTO.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequestDTO
    {
        public string RequestedRole { get; set; } = string.Empty; // Merchant ya DeliveryAgent
    }
}
```

#### **Update `IRepository`**
**File: `EShoppingZone.Services/IRepository.cs`**
```csharp
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public interface IRepository
    {
        Task<UserProfile> GetUserByEmailAsync(string email);
        Task AddUserAsync(UserProfile user);
        Task AddRoleRequestAsync(RoleRequest roleRequest);
        Task<RoleRequest> GetRoleRequestByUserIdAsync(string userId);
        Task<List<RoleRequest>> GetPendingRoleRequestsAsync();
        Task UpdateRoleRequestAsync(RoleRequest roleRequest);
    }
}
```

#### **Update `Repository`**
**File: `EShoppingZone.Services/Repository.cs`**
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

        public async Task<RoleRequest> GetRoleRequestByUserIdAsync(string userId)
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

#### **Create `RoleRequestController`**
**File: `EShoppingZone.Controllers/RoleRequestController.cs`**
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using EShoppingZone.Models;
using EShoppingZone.Services;

namespace EShoppingZone.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class RoleRequestController : ControllerBase
    {
        private readonly IRepository _repository;
        private readonly IRabbitMQProducer _rabbitMQProducer;

        public RoleRequestController(IRepository repository, IRabbitMQProducer rabbitMQProducer)
        {
            _repository = repository;
            _rabbitMQProducer = rabbitMQProducer;
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

            var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated.",
                });
            }

            // Check if user already has a pending request
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

            // Send email notification via RabbitMQ
            var emailMessage = new EmailMessage
            {
                Email = User.Identity.Name,
                Subject = "Role Request Submitted",
                Body = $"Your application to become a {request.RequestedRole} has been accepted. We will review it and get back to you soon."
            };
            _rabbitMQProducer.PublishMessage(emailMessage, "user-events");

            return Ok(new ResponseDTO<string>
            {
                Success = true,
                Message = "Role request submitted successfully.",
            });
        }
    }
}
```

- **What This Does**:
  - `SubmitRoleRequest` endpoint banaya jo `Customer` role ke users ke liye hai.
  - JWT token se user ka `UserId` extract kiya.
  - Check kiya ki user ka pehle se koi pending request toh nahi hai.
  - Naya `RoleRequest` create kiya aur database mein save kiya.
  - RabbitMQ ke through email notification bheji ("Your application is accepted, we will review it...").

#### **Add `EmailMessage` Model**
**File: `EShoppingZone.Models/EmailMessage.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class EmailMessage
    {
        public string Email { get; set; } = string.Empty;
        public string Subject { get; set; } = string.Empty;
        public string Body { get; set; } = string.Empty;
    }
}
```

#### **Update RabbitMQ Producer**
**File: `EShoppingZone.Services/IRabbitMQProducer.cs`**
```csharp
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public interface IRabbitMQProducer
    {
        void PublishMessage(EmailMessage message, string queueName);
    }
}
```

**File: `EShoppingZone.Services/RabbitMQProducer.cs`**
```csharp
using RabbitMQ.Client;
using System.Text;
using System.Text.Json;
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public class RabbitMQProducer : IRabbitMQProducer
    {
        private readonly IConnection _connection;
        private readonly IModel _channel;

        public RabbitMQProducer()
        {
            var factory = new ConnectionFactory { HostName = "localhost" };
            _connection = factory.CreateConnection();
            _channel = _connection.CreateModel();
            _channel.QueueDeclare(queue: "user-events", durable: false, exclusive: false, autoDelete: false, arguments: null);
        }

        public void PublishMessage(EmailMessage message, string queueName)
        {
            var body = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(message));
            _channel.BasicPublish(exchange: "", routingKey: queueName, basicProperties: null, body: body);
        }
    }
}
```

- **Note**: Yeh RabbitMQ setup pehle se tha, bas `EmailMessage` model ke hisaab se adjust kiya.

---

### **Step 4: Admin Approval Workflow**
Ab admin ke liye endpoints banayenge jo pending role requests ko approve ya reject kar sake.

#### **Create `RoleRequestResponseDTO`**
**File: `EShoppingZone.Models/RoleRequestResponseDTO.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequestResponseDTO
    {
        public int RequestId { get; set; }
        public string Status { get; set; } = string.Empty; // Approved ya Rejected
    }
}
```

#### **Add Admin Endpoints to `RoleRequestController`**
**File: `EShoppingZone.Controllers/RoleRequestController.cs`**
`

'''
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
        private readonly IEmailService _emailService; // Changed to IEmailService
        private readonly UserManager<UserProfile> _userManager;
        private readonly EShoppingZoneDbContext _context;

        public RoleRequestController(
            IRepository repository,
            IEmailService emailService, // Changed to IEmailService
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

            var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated.",
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

            // Send email directly using EmailService
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
                // Remove existing roles and add the new role
                var user = roleRequest.User;
                var existingRoles = await _userManager.GetRolesAsync(user);
                await _userManager.RemoveFromRolesAsync(user, existingRoles);
                await _userManager.AddToRoleAsync(user, roleRequest.RequestedRole);

                // Send approval email
                await _emailService.SendEmailAsync(
                    user.Email,
                    $"Congratulations! Your request to become a {roleRequest.RequestedRole} has been approved!",
                    "Role Request Approved"
                );
            }
            else
            {
                // Send rejection email
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
    }
}


'''
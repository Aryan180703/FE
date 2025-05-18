Bhai, sorry for the confusion! 😊 Samajh gaya, tune bataya ki tune sirf migration 

---

### **Full Files for Backend**

#### **File 1: `UserProfile.cs`**
**Path: `EShoppingZone.Models/UserProfile.cs`**
```csharp
using Microsoft.AspNetCore.Identity;

namespace EShoppingZone.Models
{
    public class UserProfile : IdentityUser<int>  // Id is int
    {
        public string? Image { get; set; }
        public List<RoleRequest> RoleRequests { get; set; } = new List<RoleRequest>();
    }
}
```

#### **File 2: `RoleRequest.cs`**
**Path: `EShoppingZone.Models/RoleRequest.cs`**  
(Tune `UserId` ko `string` se `int` mein change kiya hai, yeh wahi updated version hai.)
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequest
    {
        public int Id { get; set; }
        public int UserId { get; set; }  // Changed from string to int
        public string RequestedRole { get; set; } = string.Empty; // Merchant or DeliveryAgent
        public string Status { get; set; } = "Pending"; // Pending, Approved, Rejected
        public DateTime RequestedAt { get; set; }
        public DateTime? ReviewedAt { get; set; }
        public UserProfile User { get; set; } // Navigation property
    }
}
```

#### **File 3: `RoleRequestDTO.cs`**
**Path: `EShoppingZone.Models/RoleRequestDTO.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class RoleRequestDTO
    {
        public string RequestedRole { get; set; } = string.Empty; // Merchant or DeliveryAgent
    }
}
```

#### **File 4: `RoleRequestResponseDTO.cs`**
**Path: `EShoppingZone.Models/RoleRequestResponseDTO.cs`**
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

#### **File 5: `SignupRequest.cs`**
**Path: `EShoppingZone.Models/SignupRequest.cs`**  
(Role field hata diya, kyunki ab by default `Customer` set hoga.)
```csharp
namespace EShoppingZone.Models
{
    public class SignupRequest
    {
        public string Email { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
    }
}
```

#### **File 6: `LoginRequest.cs`**
**Path: `EShoppingZone.Models/LoginRequest.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class LoginRequest
    {
        public string Email { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
    }
}
```

#### **File 7: `ResponseDTO.cs`**
**Path: `EShoppingZone.Models/ResponseDTO.cs`**
```csharp
namespace EShoppingZone.Models
{
    public class ResponseDTO<T>
    {
        public bool Success { get; set; }
        public string Message { get; set; } = string.Empty;
        public T Data { get; set; }
    }
}
```

#### **File 8: `EShoppingZoneDbContext.cs`**
**Path: `EShoppingZone.Data/EShoppingZoneDbContext.cs`**
```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Models;

namespace EShoppingZone.Data
{
    public class EShoppingZoneDbContext : IdentityDbContext<UserProfile, IdentityRole<int>, int>
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

#### **File 9: `IEmailService.cs`**
**Path: `EShoppingZone.Interfaces/IEmailService.cs`**
```csharp
namespace EShoppingZone.Interfaces
{
    public interface IEmailService
    {
        Task SendEmailAsync(string email, string body, string subject);
    }
}
```

#### **File 10: `EmailService.cs`**
**Path: `EShoppingZone.Services/EmailService.cs`**  
(Yeh tune already diya tha, bas main yaha include kar raha hoon for completeness.)
```csharp
using System;
using System.Net;
using System.Net.Mail;
using System.Threading.Tasks;
using EShoppingZone.Data;
using EShoppingZone.Interfaces;
using EShoppingZone.Models;
using Microsoft.AspNetCore.Identity;

namespace EShoppingZone.Services
{
    public class EmailService : IEmailService 
    {
        private readonly UserManager<UserProfile> _userManager;
        private readonly SignInManager<UserProfile> _signInManager;
        private readonly EShoppingZoneDbContext _context;
        private readonly IConfiguration _configuration;
        
        public EmailService(
            UserManager<UserProfile> userManager,
            SignInManager<UserProfile> signInManager,
            EShoppingZoneDbContext context,
            IConfiguration configuration)
        {
            _userManager = userManager;
            _signInManager = signInManager;
            _context = context;
            _configuration = configuration;
        }

        public async Task SendEmailAsync(string email, string body, string subject)
        {
            var user = await _userManager.FindByEmailAsync(email);

            var smtpClient = new SmtpClient(_configuration["Smtp:Host"])
            {
                Port = int.Parse(_configuration["Smtp:Port"]),
                Credentials = new NetworkCredential(_configuration["Smtp:Username"], _configuration["Smtp:Password"]),
                EnableSsl = true
            };

            var mailMessage = new MailMessage
            {
                From = new MailAddress(_configuration["Smtp:FromEmail"], _configuration["Smtp:FromName"]),
                Subject = subject,
                Body = body,
                IsBodyHtml = true
            };
            mailMessage.To.Add(email);
            await smtpClient.SendMailAsync(mailMessage);
        }
    }
}
```

#### **File 11: `IAuthService.cs`**
**Path: `EShoppingZone.Services/IAuthService.cs`**
```csharp
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public interface IAuthService
    {
        Task<ResponseDTO<string>> SignupAsync(SignupRequest request);
        Task<ResponseDTO<string>> LoginAsync(LoginRequest request);
    }
}
```

#### **File 12: `IRepository.cs`**
**Path: `EShoppingZone.Services/IRepository.cs`**
```csharp
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public interface IRepository
    {
        Task<UserProfile> GetUserByEmailAsync(string email);
        Task AddUserAsync(UserProfile user);
        Task AddRoleRequestAsync(RoleRequest roleRequest);
        Task<RoleRequest> GetRoleRequestByUserIdAsync(int userId);  // Updated to int
        Task<List<RoleRequest>> GetPendingRoleRequestsAsync();
        Task UpdateRoleRequestAsync(RoleRequest roleRequest);
    }
}
```

#### **File 13: `AuthService.cs`**
**Path: `EShoppingZone.Services/AuthService.cs`**
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

#### **File 14: `Repository.cs`**
**Path: `EShoppingZone.Services/Repository.cs`**
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

#### **File 15: `AuthController.cs`**
**Path: `EShoppingZone.Controllers/AuthController.cs`**
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

#### **File 16: `RoleRequestController.cs`**
**Path: `EShoppingZone.Controllers/RoleRequestController.cs`**
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
                    $"{roleRequest.RequestedRole} has b



```
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
    }
}
```
# Spring Boot + React Architecture Guide

This document describes the architecture pattern used for Spring Boot and React applications, including user management, authentication, authorization, and the connection between frontend and backend.

## Table of Contents
1. [Project Structure](#project-structure)
2. [User Management Flow](#user-management-flow)
3. [Authentication & Authorization](#authentication--authorization)
4. [React-Spring Boot Connection](#react-spring-boot-connection)
5. [Security Configuration](#security-configuration)
6. [Exception Handling & Error Response](#exception-handling--error-response)
7. [Validation & Input Sanitization](#validation--input-sanitization)
8. [Database Migrations](#database-migrations)
9. [API Documentation](#api-documentation)
10. [Logging & Monitoring](#logging--monitoring)
11. [Pagination & Sorting](#pagination--sorting)
12. [CORS Configuration](#cors-configuration)
13. [Performance Optimization](#performance-optimization)
14. [Security Enhancements](#security-enhancements)
15. [Testing Strategy](#testing-strategy)
16. [Code Quality & Standards](#code-quality--standards)
17. [Additional Patterns](#additional-patterns)
18. [Key Patterns & Best Practices](#key-patterns--best-practices)

---

## Project Structure

### Spring Boot Backend Structure

```
six-sigma-api/
├── src/main/java/com/six/sigma/funnels/
│   ├── config/                    # Configuration classes
│   │   ├── SecurityConfiguration.java
│   │   ├── SecurityJwtConfiguration.java
│   │   └── WebConfigurer.java
│   ├── domain/                     # JPA Entities
│   │   ├── User.java
│   │   ├── Company.java
│   │   └── ...
│   ├── repository/                 # JPA Repositories
│   │   ├── UserRepository.java
│   │   └── ...
│   ├── service/                    # Business Logic Layer
│   │   ├── UserService.java
│   │   ├── dto/                    # Data Transfer Objects
│   │   │   ├── AdminUserDTO.java
│   │   │   ├── UserDetailsDTO.java
│   │   │   └── ...
│   │   └── impl/                   # Service Implementations
│   │       └── UserServiceImpl.java
│   ├── web/rest/                   # REST Controllers
│   │   ├── AccountResource.java    # User account operations
│   │   ├── AuthenticateController.java  # Login/Auth endpoints
│   │   ├── UserResource.java      # User CRUD operations
│   │   └── ...
│   ├── security/                   # Security Components
│   │   ├── jwt/
│   │   │   ├── JWTFilter.java     # JWT Token Filter
│   │   │   └── TokenProvider.java # Token generation/validation
│   │   ├── DomainUserDetailsService.java
│   │   └── SecurityUtils.java
│   ├── operation/util/            # Utility Classes
│   │   └── UserUtil.java
│   └── constants/                  # Constants
│       └── SixSigmaConstants.java
└── src/main/resources/
    └── config/
        └── application-dev.yml
```

### React Frontend Structure

```
six-sigma-ui/
├── src/
│   ├── app/
│   │   ├── modules/
│   │   │   └── auth/
│   │   │       ├── core/
│   │   │       │   ├── Auth.tsx           # Auth Context Provider
│   │   │       │   ├── AuthHelpers.ts    # Auth helper functions
│   │   │       │   ├── _requests.ts       # API request functions
│   │   │       │   └── _models.ts        # TypeScript models
│   │   │       └── components/
│   │   │           ├── Login.tsx
│   │   │           └── ...
│   │   ├── Constants.tsx                  # API endpoints & constants
│   │   ├── useAxiosInterceptor.ts         # Axios interceptor hook
│   │   └── pages/                        # Page components
│   ├── config/
│   │   └── helpers/                       # Helper functions
│   └── main.tsx                          # App entry point
└── package.json
```

---

## User Management Flow

### 1. User Creation Flow

#### Frontend (React)
```typescript
// src/app/modules/auth/core/_requests.ts
export function createUser(
  email: string,
  firstname: string,
  lastname: string,
  password: string
) {
  return axios.post(POST_REGISTER_USER, {
    email,
    firstName: firstname,
    lastName: lastname,
    password,
    login: email,
    role: ROLE_SITE_SUBSCRIBER,
    company: COMPANY_NAME
  });
}
```

#### Backend (Spring Boot)

**Controller Layer:**
```java
// AccountResource.java
@PostMapping("/create")
@ResponseStatus(HttpStatus.CREATED)
public void createUserAccount(@Valid @RequestBody ManagedUserVM userDTO) {
    if (isPasswordLengthInvalid(userDTO.getPassword())) {
        throw new InvalidPasswordException();
    }
    User user = userUtil.createUser(userDTO);
    Company company = companyRepository.findByName(userDTO.getCompany())
            .orElseThrow(() -> new EntityNotFoundException("Company not found"));
    commonUtil.saveCompanyUserPreferenceByKey(company, user, ...);
}
```

**Service/Util Layer:**
```java
// UserUtil.java
@Transactional
public User createUser(ManagedUserVM managedUserVM) {
    User user = userService.getUser(managedUserVM.getLogin());
    if (user != null) {
        throw new UserAlreadyExistsException("Email already registered");
    }
    user = userService.createUser(managedUserVM);
    
    // Create company-user relationship
    CompanyUserDTO companyUserDTO = new CompanyUserDTO();
    companyUserDTO.setCompany(company);
    companyUserDTO.setUser(userDTO);
    companyUserService.save(companyUserDTO);
    
    // Assign role
    RoleCompanyUserDTO roleCompanyUserDTO = new RoleCompanyUserDTO();
    roleCompanyUserDTO.setRole(roleDTO);
    roleCompanyUserDTO.setUser(userDTO);
    roleCompanyUserService.save(roleCompanyUserDTO);
    
    return user;
}
```

**Service Layer:**
```java
// UserService.java
public User createUser(ManagedUserVM userDTO) {
    // Check for existing user
    userRepository.findOneByLogin(userDTO.getLogin().toLowerCase())
        .ifPresent(existingUser -> {
            throw new UsernameAlreadyUsedException();
        });
    
    // Create new user
    User newUser = new User();
    String encryptedPassword = passwordEncoder.encode(userDTO.getPassword());
    newUser.setLogin(userDTO.getLogin().toLowerCase());
    newUser.setPassword(encryptedPassword);
    newUser.setFirstName(userDTO.getFirstName());
    newUser.setLastName(userDTO.getLastName());
    newUser.setEmail(userDTO.getEmail().toLowerCase());
    newUser.setActivated(true);
    
    // Set authorities
    Set<Authority> authorities = new HashSet<>();
    authorityRepository.findById(AuthoritiesConstants.USER).ifPresent(authorities::add);
    newUser.setAuthorities(authorities);
    
    return userRepository.save(newUser);
}
```

### 2. User Login Flow

#### Frontend (React)
```typescript
// src/app/modules/auth/components/Login.tsx
const formik = useFormik({
  initialValues,
  validationSchema: loginSchema,
  onSubmit: async (values) => {
    try {
      // Step 1: Login and get JWT token
      await login(values.email, values.password).then(
        async ({ data: auth }) => {
          saveAuth(auth); // Save token to localStorage
          
          // Step 2: Fetch user details using token
          await getUserByToken(auth.api_token).then(({ data: user }) => {
            setCurrentUser(user);
          });
        }
      );
    } catch (error) {
      // Handle error
    }
  }
});
```

```typescript
// src/app/modules/auth/core/_requests.ts
export function login(email: string, password: string) {
  return axios.post<AuthModel>(POST_FETCH_BEARER_TOKEN, {
    username: email,
    password,
    rememberMe: true
  });
}

export function getUserByToken(token: string) {
  return axios.post<UserModel>(POST_FETCH_USER, {
    api_token: token,
  });
}
```

#### Backend (Spring Boot)

**Controller Layer:**
```java
// AuthenticateController.java
@PostMapping("/login")
public ResponseEntity<AuthModel> login(@Valid @RequestBody LoginVM loginVM) {
    // Check subscription status
    User loginUser = userRepository.findOneByEmailIgnoreCase(loginVM.getUsername())
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
    
    // Authenticate user
    UsernamePasswordAuthenticationToken authenticationToken = 
        new UsernamePasswordAuthenticationToken(loginVM.getUsername(), loginVM.getPassword());
    
    Authentication authentication;
    try {
        authentication = authenticationManagerBuilder.getObject()
            .authenticate(authenticationToken);
    } catch (AuthenticationException e) {
        throw new BadRequestAlertException("Invalid Credentials", "Authenticate", "UNAUTHORIZED");
    }
    
    // Set authentication in security context
    SecurityContextHolder.getContext().setAuthentication(authentication);
    
    // Generate JWT tokens
    String jwt = tokenProvider.createToken(authentication, loginVM.isRememberMe());
    String refreshToken = tokenProvider.createRefreshToken(authentication);
    
    HttpHeaders httpHeaders = new HttpHeaders();
    httpHeaders.setBearerAuth(jwt);
    
    return new ResponseEntity<>(
        new AuthModel(jwt, refreshToken, trialExpired), 
        httpHeaders, 
        HttpStatus.OK
    );
}

@PostMapping("/get_user")
public UserDetailsDTO getUserByBearerToken(HttpServletRequest request) {
    String login = request.getUserPrincipal().getName();
    if (login != null) {
        Optional<User> userOptional = userService.getUserWithAuthoritiesByLogin(login);
        if (userOptional.isPresent()) {
            User user = userOptional.orElseThrow();
            UserDetailsDTO userModel = userUtil.getCompleteUserDetails(user);
            userModel.setAuthorities(SecurityUtils.getAuthoritiesList());
            return userModel;
        }
    }
    return null;
}
```

### 3. User Update Flow

#### Frontend (React)
```typescript
// Example: Update user profile
const updateUser = async (userData: AdminUserDTO) => {
  try {
    const response = await axios.put(PUT_UPDATE_USER, userData);
    // Handle success
  } catch (error) {
    // Handle error
  }
};
```

#### Backend (Spring Boot)
```java
// UserResource.java
@PutMapping("/updateUser")
public ResponseEntity<AdminUserDTO> updateUser(@RequestBody AdminUserDTO userDTO) {
    if (userDTO.getId() != 0 && userDTO.getId() > 0) {
        Optional<User> userOptional = userRepository.findById(userDTO.getId());
        if (userOptional.isPresent()) {
            User user = userOptional.orElseThrow();
            
            // Validate user is active
            if (user.isActivated()) {
                // Validate login and email cannot be changed
                if (!user.getLogin().equalsIgnoreCase(userDTO.getLogin())) {
                    throw new BadRequestAlertException("Login field is not editable", ...);
                }
                if (!user.getEmail().equalsIgnoreCase(userDTO.getEmail())) {
                    throw new BadRequestAlertException("Email is not editable", ...);
                }
                
                // Update user
                AdminUserDTO updateUser = userService.updateUser(userDTO)
                    .orElseThrow(() -> new UsernameNotFoundException("User not exists"));
                
                return ResponseEntity.ok().body(updateUser);
            } else {
                throw new BadRequestAlertException("Can't update inactive user details", ...);
            }
        }
    }
    throw new BadRequestAlertException("User Id required", ...);
}
```

```java
// UserService.java
public Optional<AdminUserDTO> updateUser(AdminUserDTO userDTO) {
    return userRepository.findById(userDTO.getId())
        .map(user -> {
            user.setFirstName(userDTO.getFirstName());
            user.setLastName(userDTO.getLastName());
            if (userDTO.getCountryCode() != null) {
                user.setCountryCode(userDTO.getCountryCode());
            }
            if (userDTO.getPhoneNumber() != null) {
                user.setPhoneNumber(userDTO.getPhoneNumber());
            }
            userRepository.save(user);
            return new AdminUserDTO(user);
        });
}
```

---

## Authentication & Authorization

### JWT Token Flow

1. **Token Generation:**
```java
// TokenProvider.java
public String createToken(Authentication authentication, boolean rememberMe) {
    String authorities = authentication.getAuthorities().stream()
        .map(GrantedAuthority::getAuthority)
        .collect(Collectors.joining(" "));
    
    Instant now = Instant.now();
    Instant validity = rememberMe 
        ? now.plus(tokenValidityInSecondsForRememberMe, ChronoUnit.SECONDS)
        : now.plus(tokenValidityInSeconds, ChronoUnit.SECONDS);
    
    JwtClaimsSet claims = JwtClaimsSet.builder()
        .issuedAt(now)
        .expiresAt(validity)
        .subject(authentication.getName())
        .claim(AUTHORITIES_KEY, authorities)
        .build();
    
    return jwtEncoder.encode(JwtEncoderParameters.from(jwsHeader, claims))
        .getTokenValue();
}
```

2. **Token Validation (JWT Filter):**
```java
// JWTFilter.java
@Override
protected void doFilterInternal(HttpServletRequest request, 
                                HttpServletResponse response, 
                                FilterChain filterChain) {
    String jwt = resolveToken(request); // Extract from "Authorization: Bearer <token>"
    
    if (StringUtils.hasText(jwt) && this.tokenProvider.validateToken(jwt)) {
        Authentication authentication = this.tokenProvider.getAuthentication(jwt);
        SecurityContextHolder.getContext().setAuthentication(authentication);
        
        // Check subscription status
        String userName = authentication.getName();
        UserDetailsDTO userDetailsDTO = isSubscriptionExpired(userName);
        if (Boolean.TRUE.equals(userDetailsDTO.getShouldStopServices())) {
            // Return 403 Forbidden
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            return;
        }
    }
    
    filterChain.doFilter(request, response);
}
```

### Spring Security Configuration

```java
// SecurityConfiguration.java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http, MvcRequestMatcher.Builder mvc) {
    http
        .cors(withDefaults())
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(authz -> 
            authz
                // Public endpoints
                .requestMatchers(mvc.pattern(HttpMethod.POST, "/api/login")).permitAll()
                .requestMatchers(mvc.pattern(HttpMethod.POST, "/api/create")).permitAll()
                .requestMatchers(mvc.pattern("/api/register")).permitAll()
                
                // Authenticated endpoints
                .requestMatchers(mvc.pattern(HttpMethod.PUT, "/api/users/updateUser")).authenticated()
                .requestMatchers(mvc.pattern(HttpMethod.POST, "/api/get_user")).authenticated()
                
                // Role-based endpoints
                .requestMatchers(mvc.pattern("/api/admin/**")).hasAuthority(AuthoritiesConstants.ADMIN)
                
                // Company-specific permission-based endpoints
                .requestMatchers(mvc.pattern("/api/lists/{companyName}/**"))
                    .access((authentication, context) -> {
                        String companyName = context.getVariables().get("companyName");
                        return new AuthorizationDecision(
                            webSecurity.hasPermission(companyName, "VIEW_LIST")
                        );
                    })
                
                // Default: require authentication
                .requestMatchers(mvc.pattern("/api/**")).authenticated()
        )
        .sessionManagement(session -> 
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .exceptionHandling(exceptions -> 
            exceptions
                .authenticationEntryPoint(jwtAuthEntryPoint)
                .accessDeniedHandler(jwtAuthAccessDeniedEntryPoint)
        )
        .addFilterBefore(tokenAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
    
    return http.build();
}
```

---

## React-Spring Boot Connection

### 1. API Configuration

**Constants File:**
```typescript
// src/app/Constants.tsx
const API_URL: string = import.meta.env.VITE_APP_API_URL;

// Authentication endpoints
export const POST_FETCH_BEARER_TOKEN = `${API_URL}/login`;
export const POST_FETCH_USER = `${API_URL}/get_user`;
export const POST_REGISTER_USER = `${API_URL}/create`;
export const PUT_UPDATE_USER = `${API_URL}/users/updateUser`;
export const POST_CHANGE_PASSWORD = `${API_URL}/account/change-password`;

// Company-specific endpoints
export const GET_ALL_LIST = (companyName: string | null) =>
  `${API_URL}/lists/${companyName}/list`;
```

### 2. Axios Interceptor Setup

**Request Interceptor (Add JWT Token):**
```typescript
// src/app/modules/auth/core/AuthHelpers.ts
export function setupAxios(axios: any) {
  axios.defaults.headers.Accept = 'application/json';
  
  // Request interceptor: Add Bearer token to all requests
  axios.interceptors.request.use(
    (config: { headers: { Authorization: string } }) => {
      const auth = getAuth();
      if (auth && auth.api_token) {
        config.headers.Authorization = `Bearer ${auth.api_token}`;
      }
      return config;
    },
    (err: any) => Promise.reject(err)
  );
  
  // Response interceptor: Handle errors
  axios.interceptors.response.use(
    (response: object) => response,
    (error: any) => {
      const currentPath = window.location.pathname;
      
      if (error.response.status === 401) {
        // Token expired or invalid
        if (currentPath !== LOGIN_PATH) {
          show401ErrorAlert(TOKEN_EXPIRE_ERROR);
        }
      } else if (error.response.status === 400) {
        // Bad request - show field errors or message
        const fieldErrors = error.response.data.fieldErrors;
        if (fieldErrors && fieldErrors.length > 0) {
          displayFieldErrors(fieldErrors);
        } else if (error.response.data.message) {
          showErrorAlert(error.response.data.message);
        }
      } else if (error.response.status === 500) {
        showErrorAlert(INTERNAL_SERVER_ERROR_MESSAGE);
      }
      
      return Promise.reject(error);
    }
  );
}
```

**Custom Interceptor Hook:**
```typescript
// src/app/useAxiosInterceptor.ts
export const useAxiosInterceptor = () => {
  const navigate = useNavigate();
  
  useEffect(() => {
    const interceptor = axios.interceptors.response.use(
      response => response,
      error => {
        if (error.response?.data?.statusCode === SC_FORBIDDEN) {
          // Subscription expired - redirect to dashboard
          navigate(`/${CUREENT_COMPANY_NAME}/dashboard`);
        }
        return Promise.reject(error);
      }
    );
    
    return () => axios.interceptors.response.eject(interceptor);
  }, [navigate]);
};
```

### 3. Authentication Context

```typescript
// src/app/modules/auth/core/Auth.tsx
type AuthContextProps = {
  auth: AuthModel | undefined;
  saveAuth: (auth: AuthModel | undefined) => void;
  currentUser: UserModel | undefined;
  setCurrentUser: Dispatch<SetStateAction<UserModel | undefined>>;
  logout: () => void;
};

const AuthProvider: FC<WithChildren> = ({ children }) => {
  const [auth, setAuth] = useState<AuthModel | undefined>(authHelper.getAuth());
  const [currentUser, setCurrentUser] = useState<UserModel | undefined>();
  
  const saveAuth = (auth: AuthModel | undefined) => {
    setAuth(auth);
    if (auth) {
      authHelper.setAuth(auth); // Save to localStorage
    } else {
      authHelper.removeAuth();
    }
  };
  
  const logout = () => {
    saveAuth(undefined);
    setCurrentUser(undefined);
  };
  
  return (
    <AuthContext.Provider value={{ auth, saveAuth, currentUser, setCurrentUser, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

// Initialize user on app load
const AuthInit: FC<WithChildren> = ({ children }) => {
  const { auth, currentUser, logout, setCurrentUser } = useAuth();
  const [showSplashScreen, setShowSplashScreen] = useState(true);
  
  useEffect(() => {
    const requestUser = async (apiToken: string) => {
      try {
        if (!currentUser) {
          const { data } = await getUserByToken(apiToken);
          if (data) {
            setCurrentUser(data);
          }
        }
      } catch (error) {
        if (currentUser) {
          logout();
        }
      } finally {
        setShowSplashScreen(false);
      }
    };
    
    if (auth && auth.api_token) {
      requestUser(auth.api_token);
    } else {
      logout();
      setShowSplashScreen(false);
    }
  }, []);
  
  return showSplashScreen ? <LayoutSplashScreen /> : <>{children}</>;
};
```

### 4. Local Storage Management

```typescript
// src/app/modules/auth/core/AuthHelpers.ts
const AUTH_LOCAL_STORAGE_KEY = 'kt-auth-react-v';

const getAuth = (): AuthModel | undefined => {
  if (!localStorage) return;
  
  const lsValue: string | null = localStorage.getItem(AUTH_LOCAL_STORAGE_KEY);
  if (!lsValue) return;
  
  try {
    const auth: AuthModel = JSON.parse(lsValue) as AuthModel;
    return auth;
  } catch (error) {
    console.error('AUTH LOCAL STORAGE PARSE ERROR', error);
  }
};

const setAuth = (auth: AuthModel) => {
  if (!localStorage) return;
  
  try {
    const lsValue = JSON.stringify(auth);
    localStorage.setItem(AUTH_LOCAL_STORAGE_KEY, lsValue);
  } catch (error) {
    console.error('AUTH LOCAL STORAGE SAVE ERROR', error);
  }
};

const removeAuth = () => {
  if (!localStorage) return;
  localStorage.removeItem(AUTH_LOCAL_STORAGE_KEY);
};
```

---

## Security Configuration

### Password Encoding

```java
// SecurityConfiguration.java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

### Custom User Details Service

```java
// DomainUserDetailsService.java
@Override
@Transactional(readOnly = true)
public CustomUserDetailsDTO loadUserByUsername(final String login) {
    String lowercaseLogin = login.toLowerCase(Locale.ENGLISH);
    User user = userRepository.findOneByEmailIgnoreCase(lowercaseLogin)
        .orElseThrow(() -> new UsernameNotFoundException("User not available"));
    
    if (!user.isActivated()) {
        throw new UserNotActivatedException("User was not activated");
    }
    
    // Build user details with company-wise permissions
    Map<String, List<String>> userCompanyWisePermissions = new HashMap<>();
    List<String> companies = new ArrayList<>();
    
    UserDetailsDTO userDetailsDTO = userUtil.getCompleteUserDetails(user);
    for (CompanyDetailsDTO companyDetailsDTO : userDetailsDTO.getCompanies()) {
        companies.add(companyDetailsDTO.getName());
        userCompanyWisePermissions.putIfAbsent(
            companyDetailsDTO.getName(), 
            companyDetailsDTO.getAuthorities()
        );
    }
    
    GrantedAuthority authority = new SimpleGrantedAuthority(SixSigmaConstants.ROLE_USER);
    Collection<GrantedAuthority> authorities = new HashSet<>();
    authorities.add(authority);
    
    return new CustomUserDetailsDTO(
        user.getEmail(), 
        user.getPassword(),
        authorities, 
        user.getId(), 
        companies, 
        userCompanyWisePermissions, 
        userCompanyWiseRoles
    );
}
```

---

## Exception Handling & Error Response

### Global Exception Handler

**Backend Implementation:**
```java
// ExceptionTranslator.java
@ControllerAdvice
public class ExceptionTranslator extends ResponseEntityExceptionHandler {
    
    private static final String FIELD_ERRORS_KEY = "fieldErrors";
    private static final String MESSAGE_KEY = "message";
    private static final String PATH_KEY = "path";
    
    @Value("${jhipster.clientApp.name}")
    private String applicationName;
    
    private final Environment env;
    
    @ExceptionHandler
    public ResponseEntity<Object> handleAnyException(Throwable ex, NativeWebRequest request) {
        ProblemDetailWithCause pdCause = wrapAndCustomizeProblem(ex, request);
        return handleExceptionInternal(
            (Exception) ex, 
            pdCause, 
            buildHeaders(ex), 
            HttpStatusCode.valueOf(pdCause.getStatus()), 
            request
        );
    }
    
    // Handles validation errors
    private List<FieldErrorVM> getFieldErrors(MethodArgumentNotValidException ex) {
        return ex.getBindingResult().getFieldErrors().stream()
            .map(f -> new FieldErrorVM(
                f.getObjectName().replaceFirst("DTO$", ""),
                f.getField(),
                StringUtils.isNotBlank(f.getDefaultMessage()) 
                    ? f.getDefaultMessage() 
                    : f.getCode()
            ))
            .toList();
    }
}
```

### Custom Exception Handlers

**Domain-Specific Exception Handlers:**
```java
// UserAccountExceptionHandler.java
@ControllerAdvice
public class UserAccountExceptionHandler {
    
    @ExceptionHandler(UserAlreadyExistsException.class)
    public ResponseEntity<ErrorMessage> handleUserAlreadyExists(
            UserAlreadyExistsException ex, WebRequest request) {
        ErrorMessage errorMessage = new ErrorMessage(
            HttpStatus.BAD_REQUEST.value(),
            ex.getMessage(),
            request.getDescription(false)
        );
        return new ResponseEntity<>(errorMessage, HttpStatus.BAD_REQUEST);
    }
    
    @ExceptionHandler(EntryNotFoundException.class)
    public ResponseEntity<ErrorMessage> handleEntryNotFound(
            EntryNotFoundException ex, WebRequest request) {
        ErrorMessage errorMessage = new ErrorMessage(
            HttpStatus.BAD_REQUEST.value(),
            ex.getMessage(),
            request.getDescription(false)
        );
        return new ResponseEntity<>(errorMessage, HttpStatus.BAD_REQUEST);
    }
}
```

### Error Response Structure

**Error Message DTO:**
```java
// ErrorMessage.java
public class ErrorMessage {
    private int statusCode;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, 
                pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSZ", 
                timezone = "UTC")
    private Date timestamp;
    
    private String message;
    private String description;
    
    public ErrorMessage(int statusCode, String message, String description) {
        this.statusCode = statusCode;
        this.timestamp = new Date();
        this.message = message;
        this.description = description;
    }
}
```

**RFC 7807 Problem Details Format:**
- Follows RFC 7807 standard for HTTP API error responses
- Includes `type`, `title`, `status`, `detail`, `instance`, and `fieldErrors`
- Provides consistent error structure across all endpoints

### Frontend Error Handling

**Error Response Processing:**
```typescript
// Axios interceptor handles structured error responses
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.data?.fieldErrors) {
      // Handle validation field errors
      const fieldErrors = error.response.data.fieldErrors;
      displayFieldErrors(fieldErrors);
    } else if (error.response?.data?.message) {
      // Handle general error messages
      showErrorAlert(error.response.data.message);
    }
    return Promise.reject(error);
  }
);
```

---

## Validation & Input Sanitization

### Backend Validation (Bean Validation)

**View Model Validation:**
```java
// LoginVM.java
public class LoginVM {
    @NotNull
    @Size(min = 1, max = 50)
    private String username;
    
    @NotNull
    @Size(min = 4, max = 100)
    private String password;
    
    private boolean rememberMe;
}

// ManagedUserVM.java
public class ManagedUserVM extends AdminUserDTO {
    public static final int PASSWORD_MIN_LENGTH = 4;
    public static final int PASSWORD_MAX_LENGTH = 100;
    
    @Size(min = PASSWORD_MIN_LENGTH, max = PASSWORD_MAX_LENGTH)
    private String password;
    
    // Additional fields...
}
```

**Controller Validation:**
```java
// AccountResource.java
@PostMapping("/create")
@ResponseStatus(HttpStatus.CREATED)
public void createUserAccount(@Valid @RequestBody ManagedUserVM userDTO) {
    // @Valid annotation triggers validation
    // If validation fails, MethodArgumentNotValidException is thrown
    // ExceptionTranslator handles it and returns field errors
    if (isPasswordLengthInvalid(userDTO.getPassword())) {
        throw new InvalidPasswordException();
    }
    // Process valid data...
}
```

**Common Validation Annotations:**
- `@NotNull` - Field must not be null
- `@NotBlank` - String must not be blank (not null, trimmed length > 0)
- `@Size(min=, max=)` - String/Collection size constraints
- `@Email` - Email format validation
- `@Pattern(regexp=)` - Custom regex validation
- `@Min`, `@Max` - Numeric value constraints

### Frontend Validation (Yup + Formik)

**Validation Schema:**
```typescript
// SignupForm.tsx
import * as Yup from 'yup';

const validationSchema = Yup.object({
  firstName: Yup.string()
    .min(3, "Minimum 3 characters")
    .max(15, "Maximum 15 characters")
    .test(
      "no-leading-spaces",
      "The first three characters should not be spaces.",
      (value) => {
        if (!value || value.length < 3) return true;
        return REGEX_NAME.test(value);
      })
    .required("First Name is required."),
    
  email: Yup.string()
    .matches(REGEX_EMAIL, "Invalid email format")
    .email("Email is not valid.")
    .required("Email is required."),
    
  password: Yup.string()
    .matches(
      REGEX_PASSWORD,
      "Your password needs at least 8 characters, one number, one special character, one lowercase letter, one uppercase letter—and no spaces"
    )
    .min(8, "Minimum 8 characters")
    .max(50, "Maximum 50 characters")
    .required("Password is required"),
    
  confirmPassword: Yup.string()
    .oneOf([Yup.ref("password"), ""], "Both password fields must match exactly.")
    .required("Confirm Password is required."),
});
```

**Formik Integration:**
```typescript
const formik = useFormik({
  initialValues: {
    firstName: '',
    lastName: '',
    email: '',
    password: '',
    confirmPassword: ''
  },
  validationSchema: validationSchema,
  onSubmit: async (values) => {
    // Handle form submission
  }
});
```

### Input Sanitization

**Backend Sanitization:**
```java
// Always sanitize user input before processing
public User createUser(ManagedUserVM userDTO) {
    // Normalize email (lowercase, trim)
    String email = userDTO.getEmail().toLowerCase().trim();
    
    // Normalize login (lowercase)
    String login = userDTO.getLogin().toLowerCase();
    
    // Validate and sanitize phone numbers
    String phoneNumber = sanitizePhoneNumber(userDTO.getPhoneNumber());
    
    // Use parameterized queries (JPA handles this automatically)
    // Never concatenate user input into SQL queries
}
```

**Frontend Sanitization:**
```typescript
// Sanitize input before sending to API
const sanitizeInput = (input: string): string => {
  return input.trim().replace(/[<>]/g, ''); // Remove potential HTML tags
};

// Use DOMPurify for HTML content if needed
import DOMPurify from 'dompurify';
const sanitizedHtml = DOMPurify.sanitize(userInput);
```

### SQL Injection Prevention

- **Always use JPA/Hibernate** - Parameterized queries are automatic
- **Never use string concatenation** for SQL queries
- **Use @Query with named parameters:**
```java
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
```

### XSS Prevention

- **Escape output** in React (React does this by default)
- **Use DOMPurify** for HTML content
- **Set Content Security Policy** headers
- **Validate and sanitize** all user inputs

---

## Database Migrations

### Liquibase Configuration

**Liquibase Setup:**
```java
// LiquibaseConfiguration.java
@Configuration
public class LiquibaseConfiguration {
    
    @Value("${application.liquibase.async-start:true}")
    private Boolean asyncStart;
    
    @Bean
    public SpringLiquibase liquibase(
        @Qualifier("taskExecutor") Executor executor,
        LiquibaseProperties liquibaseProperties,
        @LiquibaseDataSource ObjectProvider<DataSource> liquibaseDataSource,
        ObjectProvider<DataSource> dataSource,
        DataSourceProperties dataSourceProperties
    ) {
        SpringLiquibase liquibase;
        if (Boolean.TRUE.equals(asyncStart)) {
            liquibase = SpringLiquibaseUtil.createAsyncSpringLiquibase(
                this.env, executor, liquibaseDataSource.getIfAvailable(),
                liquibaseProperties, dataSource.getIfUnique(), dataSourceProperties
            );
        } else {
            liquibase = SpringLiquibaseUtil.createSpringLiquibase(
                liquibaseDataSource.getIfAvailable(), liquibaseProperties,
                dataSource.getIfUnique(), dataSourceProperties
            );
        }
        liquibase.setChangeLog("classpath:config/liquibase/master.xml");
        return liquibase;
    }
}
```

### Changelog Structure

**Master Changelog:**
```xml
<!-- config/liquibase/master.xml -->
<?xml version="1.0" encoding="utf-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog 
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
    
    <property name="now" value="now()" dbms="mariadb"/>
    <property name="floatType" value="float" dbms="mariadb"/>
    <property name="uuidType" value="varchar(36)" dbms="mariadb"/>
    <property name="datetimeType" value="datetime(6)" dbms="mariadb"/>
    
    <include file="config/liquibase/changelog/00000000000000_initial_schema.xml"/>
    <include file="config/liquibase/changelog/20250108123724_added_entity_AppParamGroup.xml"/>
    <!-- Additional changelogs... -->
</databaseChangeLog>
```

**Individual Changelog Example:**
```xml
<!-- config/liquibase/changelog/20250108123724_added_entity_AppParamGroup.xml -->
<databaseChangeLog>
    <changeSet id="20250108123724-1" author="jhipster">
        <createTable tableName="app_param_group">
            <column name="id" type="bigint" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="uuid" type="${uuidType}">
                <constraints unique="true" nullable="false"/>
            </column>
            <column name="name" type="varchar(75)">
                <constraints nullable="false"/>
            </column>
            <column name="active" type="boolean" defaultValueBoolean="true"/>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

### Migration Best Practices

1. **Version Control:** All changelogs should be in version control
2. **Naming Convention:** Use timestamp prefix (YYYYMMDDHHMMSS_description.xml)
3. **Idempotency:** Changes should be safe to run multiple times
4. **Rollback:** Include rollback scripts for critical changes
5. **Testing:** Test migrations on staging before production
6. **Atomic Changes:** Each changeSet should be atomic

### Maven Liquibase Plugin

**POM Configuration:**
```xml
<plugin>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-maven-plugin</artifactId>
    <version>${liquibase.version}</version>
    <configuration>
        <changeLogFile>config/liquibase/master.xml</changeLogFile>
        <driver>org.mariadb.jdbc.Driver</driver>
        <url>${liquibase-plugin.url}</url>
        <username>${liquibase-plugin.username}</username>
        <password>${liquibase-plugin.password}</password>
    </configuration>
</plugin>
```

---

## API Documentation

### SpringDoc OpenAPI Configuration

**Application Properties:**
```yaml
# application.yml
springdoc:
  api-docs:
    enabled: false  # Disabled by default, enable with 'api-docs' profile
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    enabled: true

jhipster:
  api-docs:
    default-include-pattern: /api/**
    management-include-pattern: /management/**
    title: Application API
    description: Application API documentation
    version: 0.0.1
```

### Controller Documentation

**Annotating Controllers:**
```java
// AccountResource.java
@RestController
@RequestMapping("/api")
@Tag(name = "Account", description = "Account management endpoints")
public class AccountResource {
    
    @PostMapping("/create")
    @Operation(
        summary = "Create user account",
        description = "Creates a new user account with the provided information"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "201",
            description = "User created successfully"
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Invalid input or user already exists"
        )
    })
    @ResponseStatus(HttpStatus.CREATED)
    public void createUserAccount(@Valid @RequestBody ManagedUserVM userDTO) {
        // Implementation...
    }
}
```

### DTO Documentation

**Annotating DTOs:**
```java
// AdminUserDTO.java
@Schema(description = "User DTO for admin operations")
public class AdminUserDTO {
    
    @Schema(description = "User ID", example = "1")
    private Long id;
    
    @Schema(description = "User email", example = "user@example.com", required = true)
    @Email
    private String email;
    
    @Schema(description = "First name", example = "John")
    @Size(min = 1, max = 50)
    private String firstName;
    
    // Additional fields...
}
```

### Accessing API Documentation

- **Swagger UI:** `http://localhost:8080/swagger-ui.html`
- **OpenAPI JSON:** `http://localhost:8080/v3/api-docs`
- **Enable in dev:** Add `api-docs` profile to active profiles

---

## Logging & Monitoring

### Logback Configuration

**Logback Configuration:**
```xml
<!-- logback-spring.xml -->
<configuration scan="true">
    <!-- Console Appender -->
    <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
    
    <!-- Rolling File Appender -->
    <property name="LOG_PATTERN" 
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"/>
    
    <appender name="AppFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>
    
    <!-- Application Logger -->
    <logger name="com.yourpackage" level="INFO"/>
    
    <!-- Hibernate SQL Logging (Development Only) -->
    <logger name="org.hibernate.SQL" level="DEBUG" additivity="false">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="AppFile"/>
    </logger>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="AppFile"/>
    </root>
</configuration>
```

### Logging Best Practices

**Structured Logging:**
```java
// Use SLF4J Logger
private final Logger log = LoggerFactory.getLogger(YourClass.class);

// Log levels
log.trace("Very detailed information");
log.debug("Debug information");
log.info("General information");
log.warn("Warning message");
log.error("Error message", exception);

// Best practices
log.info("User created: email={}, id={}", user.getEmail(), user.getId());
log.error("Failed to create user: email={}", email, exception);
```

**Sensitive Data Masking:**
```java
// Never log passwords, tokens, or sensitive data
log.info("User login attempt: email={}", email); // ✅ Good
log.info("User login: email={}, password={}", email, password); // ❌ Bad

// Mask sensitive data in logs
private String maskEmail(String email) {
    if (email == null || email.length() < 5) return "***";
    return email.substring(0, 2) + "***" + email.substring(email.length() - 2);
}
```

### Spring Boot Actuator

**Actuator Configuration:**
```yaml
# application.yml
management:
  endpoints:
    web:
      base-path: /management
      exposure:
        include:
          - health
          - info
          - metrics
          - prometheus
  endpoint:
    health:
      show-details: when_authorized
      roles: 'ROLE_ADMIN'
      probes:
        enabled: true
  metrics:
    enable:
      http: true
      jvm: true
      process: true
    distribution:
      percentiles-histogram:
        all: true
```

**Health Check Endpoints:**
- `/management/health` - Application health status
- `/management/health/liveness` - Liveness probe
- `/management/health/readiness` - Readiness probe
- `/management/metrics` - Application metrics
- `/management/prometheus` - Prometheus metrics

### Monitoring Integration

**Prometheus Metrics:**
```yaml
# Prometheus configuration
management:
  prometheus:
    metrics:
      export:
        enabled: true
        step: 60
```

**Custom Metrics:**
```java
// Create custom metrics
@Autowired
private MeterRegistry meterRegistry;

public void recordUserCreation() {
    meterRegistry.counter("user.creation.count").increment();
}
```

---

## Pagination & Sorting

### Backend Pagination

**Controller Implementation:**
```java
// UserResource.java
@GetMapping("/{companyName}/users")
public ResponseEntity<List<RoleUserDetailsDTO>> getAllCompanyUsers(
        @PathVariable(name = "companyName") String companyName,
        @RequestParam(required = false) String search,
        @org.springdoc.core.annotations.ParameterObject Pageable pageable) {
    
    // Validate pageable properties
    if (!onlyContainsAllowedProperties(pageable)) {
        return ResponseEntity.badRequest().build();
    }
    
    final Page<RoleUserDetailsDTO> page = userService.getAllCompanyUsers(
        search, companyName, pageable
    );
    
    HttpHeaders headers = PaginationUtil.generatePaginationHttpHeaders(
        ServletUriComponentsBuilder.fromCurrentRequest(), page
    );
    
    return new ResponseEntity<>(page.getContent(), headers, HttpStatus.OK);
}
```

**Service Implementation:**
```java
// UserService.java
public Page<RoleUserDetailsDTO> getAllCompanyUsers(
        String search, String companyName, Pageable pageable) {
    
    // Create custom pageable with sorting
    Pageable correctedPageable = convertToCorrectedPageable(pageable);
    
    // Build query with search criteria
    Specification<User> spec = buildSearchSpecification(search, companyName);
    
    Page<User> users = userRepository.findAll(spec, correctedPageable);
    
    return users.map(this::convertToRoleUserDetailsDTO);
}
```

**Pagination Utility:**
```java
// PaginationUtil.java
public static HttpHeaders generatePaginationHttpHeaders(
        UriComponentsBuilder uriBuilder, Page<?> page) {
    
    HttpHeaders headers = new HttpHeaders();
    
    headers.add("X-Total-Count", Long.toString(page.getTotalElements()));
    headers.add("X-Page-Number", Integer.toString(page.getNumber()));
    headers.add("X-Page-Size", Integer.toString(page.getSize()));
    headers.add("X-Total-Pages", Integer.toString(page.getTotalPages()));
    
    return headers;
}
```

### Frontend Pagination

**React Table with Pagination:**
```typescript
// UsersTable.tsx
const fetchUsers = async (
  globalFilter: string,
  sortingOptions: { id: string; desc: boolean }[],
  paginateOptions: { pageIndex: number; pageSize: number }
): Promise<{
  rows: RoleUserDetailsDTO[];
  pageCount: number;
  rowCount: number;
}> => {
  try {
    const response = await axios.get(
      GET_ALL_COMPANY_USERS(CUREENT_COMPANY_NAME),
      {
        params: {
          search: globalFilter,
          sort: sortingOptions[0]?.id
            ? `${sortingOptions[0].id}${sortingOptions[0].desc ? ',DESC' : ',ASC'}`
            : '',
          page: paginateOptions.pageIndex,
          size: paginateOptions.pageSize,
        },
        headers: { "Cache-Control": "no-cache" },
      }
    );
    
    // Extract pagination info from headers
    const totalRecords = Number(response.headers["x-total-count"] || 0);
    
    return {
      rows: response.data,
      pageCount: Math.ceil(totalRecords / paginateOptions.pageSize),
      rowCount: totalRecords,
    };
  } catch (error) {
    console.error("Error fetching users:", error);
    return { rows: [], pageCount: 0, rowCount: 0 };
  }
};
```

**Sorting Implementation:**
```typescript
// Handle sorting
const sortingOptions = [
  { id: 'firstName', desc: false },
  { id: 'email', desc: true }
];

// Convert to backend format: "firstName,ASC,email,DESC"
const sortParam = sortingOptions
  .map(opt => `${opt.id},${opt.desc ? 'DESC' : 'ASC'}`)
  .join(',');
```

### Custom Sorting

**Backend Custom Sort Conversion:**
```java
// CommonUtil.java
public Pageable convertToCorrectedPageable(Pageable pageable) {
    return PageRequest.of(
        pageable.getPageNumber(),
        pageable.getPageSize(),
        Sort.by(pageable.getSort().stream()
            .map(order -> {
                String property = order.getProperty();
                Sort.Direction direction = order.getDirection();
                
                // Map DTO property names to SQL column names
                switch (property) {
                    case "fullName":
                        return new Sort.Order(direction, "full_name");
                    case "email":
                        return new Sort.Order(direction, "email");
                    default:
                        return new Sort.Order(direction, property);
                }
            })
            .collect(Collectors.toList())
        )
    );
}
```

---

## CORS Configuration

### CORS Filter Setup

**WebConfigurer:**
```java
// WebConfigurer.java
@Configuration
public class WebConfigurer implements ServletContextInitializer {
    
    private final JHipsterProperties jHipsterProperties;
    
    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = jHipsterProperties.getCors();
        
        // Expose custom headers
        config.addExposedHeader("Content-Disposition");
        config.addExposedHeader("X-Total-Count");
        
        if (!CollectionUtils.isEmpty(config.getAllowedOrigins()) || 
            !CollectionUtils.isEmpty(config.getAllowedOriginPatterns())) {
            
            log.debug("Registering CORS filter");
            source.registerCorsConfiguration("/api/**", config);
            source.registerCorsConfiguration("/management/**", config);
            source.registerCorsConfiguration("/v3/api-docs", config);
            source.registerCorsConfiguration("/swagger-ui/**", config);
        }
        
        return new CorsFilter(source);
    }
}
```

### CORS Configuration (application.yml)

```yaml
# application-dev.yml
jhipster:
  cors:
    allowed-origins: "http://localhost:5173,http://localhost:3000"
    allowed-methods: "GET,POST,PUT,DELETE,OPTIONS,PATCH"
    allowed-headers: "*"
    exposed-headers: "Authorization,Link,X-Total-Count"
    allow-credentials: true
    max-age: 1800
```

### Security Considerations

1. **Specific Origins:** Never use `*` for `allowed-origins` in production
2. **Credentials:** Set `allow-credentials: true` only when needed
3. **Methods:** Limit to required HTTP methods only
4. **Headers:** Restrict exposed headers to necessary ones
5. **Environment-Specific:** Use different CORS configs for dev/prod

---

## Performance Optimization

### Database Connection Pooling

**HikariCP Configuration:**
```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

### Transaction Management

**Transaction Isolation Levels:**
```java
// Use appropriate isolation level
@Transactional(isolation = Isolation.READ_COMMITTED)
public User createUser(ManagedUserVM userDTO) {
    // Operations...
}

// Read-only transactions for queries
@Transactional(readOnly = true)
public Optional<User> getUser(Long id) {
    return userRepository.findById(id);
}
```

**Transaction Best Practices:**
- Use `@Transactional` at service layer, not controller
- Mark read-only operations with `readOnly = true`
- Keep transactions short
- Avoid long-running operations in transactions
- Use appropriate isolation levels

### Lazy vs Eager Loading

**Entity Relationships:**
```java
// Use LAZY loading by default
@Entity
public class User {
    @OneToMany(fetch = FetchType.LAZY)
    private List<Order> orders;
    
    // Use EAGER only when necessary
    @ManyToOne(fetch = FetchType.EAGER)
    private Company company;
}
```

**Fetch Joins for Performance:**
```java
// Use fetch joins to avoid N+1 queries
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);
```

### Caching Strategies

**Enable Caching:**
```java
@Configuration
@EnableCaching
public class CacheConfiguration {
    
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "companies");
    }
}

// Use caching in services
@Cacheable(value = "users", key = "#id")
public Optional<User> findById(Long id) {
    return userRepository.findById(id);
}

@CacheEvict(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

### Async Processing

**Async Configuration:**
```java
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(10000);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

// Use async methods
@Async
public CompletableFuture<Void> sendEmailAsync(User user) {
    mailService.sendActivationEmail(user);
    return CompletableFuture.completedFuture(null);
}
```

---

## Security Enhancements

### Rate Limiting (Recommended)

**Implementation Pattern:**
```java
// Consider using Bucket4j or Spring Cloud Gateway rate limiting
// Example with custom filter
@Component
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) {
        String key = getClientIp(request);
        RateLimiter limiter = limiters.computeIfAbsent(key, 
            k -> RateLimiter.create(10.0)); // 10 requests per second
        
        if (!limiter.tryAcquire()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            return;
        }
        
        filterChain.doFilter(request, response);
    }
}
```

### Token Refresh Mechanism

**Refresh Token Endpoint:**
```java
// AuthenticateController.java
@PostMapping("/refresh-token")
public ResponseEntity<AuthModel> refreshToken(@RequestParam String refreshToken) {
    if (!tokenProvider.validateToken(refreshToken)) {
        throw new RuntimeException("Invalid Refresh Token. Please login again.");
    }
    
    String username = tokenProvider.getUsernameFromToken(refreshToken);
    User user = userRepository.findOneByEmailIgnoreCase(username)
        .orElseThrow(() -> new RuntimeException("User not found"));
    
    CustomUserDetailsDTO userDetails = userUtil.loadUserByUsername(user.getLogin());
    Authentication authentication = new UsernamePasswordAuthenticationToken(
        userDetails, null, userDetails.getAuthorities()
    );
    
    String newAccessToken = tokenProvider.createToken(authentication, false);
    
    return ResponseEntity.ok(new AuthModel(newAccessToken, refreshToken, false));
}
```

### Password Strength Validation

**Password Validation:**
```java
// Password validation pattern
public static final String PASSWORD_PATTERN = 
    "^(?=.*?[A-Z])(?=.*?[a-z])(?=.*?[0-9])(?=.*?[#?!@$%^&*~_\\[\\]$£(){}.,/\\\\|'\":;<>`=+-])(?!.* ).{8,}$";

@Pattern(regexp = PASSWORD_PATTERN, 
         message = "Password must contain at least 8 characters, one uppercase, one lowercase, one number, and one special character")
private String password;
```

### Account Lockout (Recommended)

**Implementation Pattern:**
```java
// Track failed login attempts
@Entity
public class User {
    private int failedLoginAttempts;
    private LocalDateTime lockoutTime;
    
    public void incrementFailedAttempts() {
        this.failedLoginAttempts++;
        if (this.failedLoginAttempts >= 5) {
            this.lockoutTime = LocalDateTime.now().plusMinutes(30);
        }
    }
    
    public boolean isLocked() {
        return lockoutTime != null && LocalDateTime.now().isBefore(lockoutTime);
    }
}
```

### Audit Logging

**Audit Entity:**
```java
// AbstractAuditingEntity.java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AbstractAuditingEntity {
    
    @CreatedBy
    @Column(name = "created_by", length = 50, updatable = false)
    private String createdBy;
    
    @CreatedDate
    @Column(name = "created_date", updatable = false)
    private Instant createdDate = Instant.now();
    
    @LastModifiedBy
    @Column(name = "last_modified_by", length = 50)
    private String lastModifiedBy;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private Instant lastModifiedDate = Instant.now();
}
```

**Enable Auditing:**
```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "springSecurityAuditorAware")
public class DatabaseConfiguration {
    // Configuration...
}
```

---

## Testing Strategy

### Unit Testing

**Service Layer Testing:**
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void testCreateUser() {
        // Given
        ManagedUserVM userDTO = new ManagedUserVM();
        userDTO.setEmail("test@example.com");
        userDTO.setPassword("password123");
        
        when(userRepository.findOneByLogin(anyString())).thenReturn(Optional.empty());
        when(passwordEncoder.encode(anyString())).thenReturn("encodedPassword");
        when(userRepository.save(any(User.class))).thenAnswer(invocation -> {
            User user = invocation.getArgument(0);
            user.setId(1L);
            return user;
        });
        
        // When
        User result = userService.createUser(userDTO);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getEmail()).isEqualTo("test@example.com");
        verify(userRepository).save(any(User.class));
    }
}
```

### Integration Testing

**REST Controller Testing:**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class AccountResourceIT {
    
    @Autowired
    private MockMvc restAccountMockMvc;
    
    @Test
    void testCreateUser() throws Exception {
        ManagedUserVM userDTO = new ManagedUserVM();
        userDTO.setEmail("test@example.com");
        userDTO.setPassword("password123");
        userDTO.setFirstName("Test");
        userDTO.setLastName("User");
        
        restAccountMockMvc
            .perform(post("/api/create")
                .contentType(MediaType.APPLICATION_JSON)
                .content(TestUtil.convertObjectToJsonBytes(userDTO)))
            .andExpect(status().isCreated());
    }
}
```

### Test Data Management

**Test Containers (Recommended):**
```java
@Testcontainers
@SpringBootTest
class UserServiceIntegrationTest {
    
    @Container
    static MariaDBContainer<?> mariaDB = new MariaDBContainer<>("mariadb:10.11")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mariaDB::getJdbcUrl);
        registry.add("spring.datasource.username", mariaDB::getUsername);
        registry.add("spring.datasource.password", mariaDB::getPassword);
    }
}
```

---

## Code Quality & Standards

### Checkstyle Configuration

**Checkstyle Setup:**
```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>${checkstyle.version}</version>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
    </configuration>
    <executions>
        <execution>
            <phase>validate</phase>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Code Review Checklist

1. **Security:**
   - No hardcoded credentials
   - Input validation present
   - SQL injection prevention
   - XSS prevention

2. **Performance:**
   - No N+1 queries
   - Proper use of transactions
   - Efficient database queries
   - Caching where appropriate

3. **Code Quality:**
   - Follows naming conventions
   - Proper error handling
   - Logging for important operations
   - Unit tests present

4. **Architecture:**
   - Follows layered architecture
   - Proper separation of concerns
   - DTOs used correctly
   - No business logic in controllers

---

## Additional Patterns

### Soft Delete Pattern

**Entity with Soft Delete:**
```java
@Entity
public class User extends AbstractAuditingEntity {
    
    @Column(name = "deleted")
    private Boolean deleted = false;
    
    @Column(name = "deleted_date")
    private Instant deletedDate;
    
    public void softDelete() {
        this.deleted = true;
        this.deletedDate = Instant.now();
    }
    
    public void restore() {
        this.deleted = false;
        this.deletedDate = null;
    }
}
```

**Repository with Soft Delete:**
```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.deleted = false")
    List<User> findAllActive();
    
    @Modifying
    @Query("UPDATE User u SET u.deleted = true, u.deletedDate = :now WHERE u.id = :id")
    void softDelete(@Param("id") Long id, @Param("now") Instant now);
}
```

### Scheduled Tasks

**Quartz Scheduler Configuration:**
```java
// SchedulerConfig.java
@Configuration
public class SchedulerConfig {
    
    @Bean
    public SchedulerFactoryBean schedulerFactoryBean(DataSource dataSource) {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setJobFactory(jobFactory());
        factory.setQuartzProperties(quartzProperties());
        return factory;
    }
}

// Scheduled Job
@Component
public class FacebookAuthorizationCronJob extends QuartzJobBean {
    
    @Override
    protected void executeInternal(JobExecutionContext context) {
        // Scheduled task logic
    }
}
```

### File Upload Handling

**File Upload Endpoint:**
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(
        @RequestParam("file") MultipartFile file,
        @RequestParam("email") String email) {
    
    if (file.isEmpty()) {
        throw new BadRequestAlertException("File is empty", "File", "empty");
    }
    
    // Validate file type and size
    validateFile(file);
    
    // Save file
    String filePath = fileService.saveFile(file, email);
    
    return ResponseEntity.ok(filePath);
}

private void validateFile(MultipartFile file) {
    // Check file size (e.g., max 10MB)
    if (file.getSize() > 10 * 1024 * 1024) {
        throw new BadRequestAlertException("File size exceeds limit", "File", "size");
    }
    
    // Check file type
    String contentType = file.getContentType();
    if (!isAllowedContentType(contentType)) {
        throw new BadRequestAlertException("File type not allowed", "File", "type");
    }
}
```

### Internationalization (i18n)

**Message Properties:**
```properties
# messages.properties
error.user.notfound=User not found
error.user.alreadyexists=User already exists
success.user.created=User created successfully
```

**Using Messages:**
```java
@Autowired
private MessageSource messageSource;

public String getMessage(String key, Object... args) {
    return messageSource.getMessage(key, args, LocaleContextHolder.getLocale());
}
```

---

## Key Patterns & Best Practices

### 1. Layered Architecture

```
Controller (REST) → Service → Repository → Database
     ↓              ↓
   DTOs          Entities
```

### 2. DTO Pattern

- **View Models (VM):** For incoming requests (e.g., `ManagedUserVM`, `LoginVM`)
- **DTOs:** For outgoing responses (e.g., `AdminUserDTO`, `UserDetailsDTO`)
- **Entities:** JPA domain models (e.g., `User`, `Company`)

### 3. Transaction Management

```java
@Transactional
public User createUser(ManagedUserVM managedUserVM) {
    // All operations in this method are transactional
}
```

### 4. Error Handling

See [Exception Handling & Error Response](#exception-handling--error-response) section for detailed implementation.

### 5. API Endpoint Naming Convention

- **Public endpoints:** `/api/login`, `/api/create`, `/api/register`
- **Authenticated endpoints:** `/api/users/updateUser`, `/api/get_user`
- **Company-specific endpoints:** `/api/{companyName}/list`, `/api/{companyName}/users`
- **Admin endpoints:** `/api/admin/**`

### 6. Security Best Practices

1. **JWT Token Storage:** Store in localStorage (consider httpOnly cookies for production)
2. **Token Expiry:** Short-lived access tokens (e.g., 1 hour) + refresh tokens (e.g., 30 days)
3. **Password Encoding:** Always use BCrypt
4. **CORS Configuration:** Configure allowed origins
5. **CSRF:** Disabled for stateless JWT (consider enabling for session-based auth)

### 7. State Management

- **React Context:** For authentication state
- **Local Storage:** For persisting auth tokens
- **Session Storage:** For company-specific data

### 8. Type Safety

**TypeScript Models:**
```typescript
// _models.ts
export interface AuthModel {
  api_token: string;
  refresh_token?: string;
  trialExpired?: boolean;
}

export interface UserModel {
  id: number;
  email: string;
  firstName: string;
  lastName: string;
  companies: CompanyModel[];
  authorities: string[];
}
```

---

## Environment Configuration

### Backend (application-dev.yml)
```yaml
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/dbname
    username: user
    password: password
  
jhipster:
  security:
    authentication:
      jwt:
        base64-secret: your-secret-key
        token-validity-in-seconds: 3600
        token-validity-in-seconds-for-remember-me: 2592000
```

### Frontend (.env)
```env
VITE_APP_API_URL=http://localhost:8080/api
VITE_APP_URL=http://localhost:5173
VITE_FACE_BOOK_APP_ID=your-app-id
```

---

## Common Workflows

### Complete Login Flow

1. User enters email/password in React form
2. React calls `POST /api/login` with credentials
3. Spring Boot authenticates via `AuthenticationManager`
4. JWT token generated and returned in response
5. React saves token to localStorage
6. React calls `POST /api/get_user` with Bearer token
7. Spring Boot validates token via `JWTFilter`
8. User details returned to React
9. React updates AuthContext with user data
10. User redirected to dashboard

### Complete User Creation Flow

1. User fills registration form in React
2. React calls `POST /api/create` with user data
3. Spring Boot validates input via `@Valid`
4. Service checks for existing user
5. Password encoded with BCrypt
6. User entity created and saved
7. Company-User relationship created
8. Role assigned to user
9. Success response returned to React
10. User redirected to login or auto-login

---

## Notes

- This architecture follows a **stateless** authentication pattern using JWT
- **Multi-tenancy** is supported via company-based routing and permissions
- **Subscription management** is integrated into the authentication flow
- **OAuth2** support is available for social login (Facebook)
- All API endpoints follow RESTful conventions
- Error handling is centralized in both frontend and backend

---

## Usage for New Projects

When starting a new project:

1. **Copy the structure** from this guide
2. **Update package names** and constants
3. **Configure database** connection
4. **Set up environment variables**
5. **Customize security rules** based on requirements
6. **Add domain-specific entities** and services
7. **Implement feature-specific endpoints** following the same patterns

This architecture provides a solid foundation for scalable Spring Boot + React applications with proper authentication, authorization, and separation of concerns.


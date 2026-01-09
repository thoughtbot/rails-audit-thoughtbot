# Security Audit Checklist

## Critical Security Issues

### 1. SQL Injection

**Detection Patterns**:
```ruby
# DANGEROUS - String interpolation in queries
where("name = '#{params[:name]}'")
where("name LIKE '%#{term}%'")
find_by_sql("SELECT * FROM users WHERE id = #{id}")
order("#{params[:sort]} #{params[:direction]}")
```

**Safe Alternatives**:
```ruby
# Use parameterized queries
where("name = ?", params[:name])
where("name LIKE ?", "%#{term}%")
where(name: params[:name])
order(Arel.sql(safe_order_string))
```

**Severity**: Critical

---

### 2. Mass Assignment

**Detection Patterns**:
```ruby
# DANGEROUS
params.permit!
User.new(params[:user])  # Without strong params
update_attributes(params)
```

**Safe Alternatives**:
```ruby
# Use strong parameters
def user_params
  params.require(:user).permit(:name, :email)
end

User.new(user_params)
```

**Severity**: Critical

---

### 3. Cross-Site Scripting (XSS)

**Detection Patterns**:
```erb
<%# DANGEROUS - raw output %>
<%= raw user_input %>
<%= user_input.html_safe %>
<%== user_input %>
```

**Safe Alternatives**:
```erb
<%# Safe - auto-escaped %>
<%= user_input %>
<%= sanitize(user_input) %>
```

**Check in JavaScript**:
```javascript
// DANGEROUS
element.innerHTML = userInput;

// Safe
element.textContent = userInput;
```

**Severity**: High

---

### 4. Command Injection

**Detection Patterns**:
```ruby
# DANGEROUS
system("convert #{params[:file]}")
`ls #{user_input}`
exec("command #{args}")
%x(command #{args})
```

**Safe Alternatives**:
```ruby
# Use array form
system("convert", params[:file])
```

**Severity**: Critical

---

### 5. Path Traversal

**Detection Patterns**:
```ruby
# DANGEROUS
send_file(params[:filename])
File.read(params[:path])
render file: params[:template]
```

**Safe Alternatives**:
```ruby
# Validate and sanitize paths
basename = File.basename(params[:filename])
safe_path = Rails.root.join("uploads", basename)
send_file(safe_path) if File.exist?(safe_path)
```

**Severity**: Critical

---

### 6. Insecure Direct Object References (IDOR)

**Detection Patterns**:
```ruby
# DANGEROUS - No authorization check
def show
  @document = Document.find(params[:id])
end
```

**Safe Alternatives**:
```ruby
# Scope to current user
def show
  @document = current_user.documents.find(params[:id])
end

# Or use authorization
def show
  @document = Document.find(params[:id])
  authorize @document
end
```

**Severity**: High

---

### 7. Missing Authentication

**Detection Patterns**:
```ruby
# Check for missing before_action
class AdminController < ApplicationController
  # Missing: before_action :authenticate_admin!
  
  def destroy
    User.find(params[:id]).destroy
  end
end
```

**Audit Check**: Every controller should have authentication unless explicitly public.

**Severity**: Critical

---

### 8. Sensitive Data Exposure

**Detection Patterns**:
```ruby
# DANGEROUS - Logging sensitive data
Rails.logger.info("Password: #{params[:password]}")
Rails.logger.info(params.inspect)

# DANGEROUS - Exposing in JSON
render json: user  # May include password_digest, tokens, etc.
```

**Safe Alternatives**:
```ruby
# Filter sensitive params
config.filter_parameters += [:password, :token, :secret]

# Whitelist JSON attributes
render json: user.as_json(only: [:id, :name, :email])
```

**Severity**: High

---

### 9. Weak Cryptography

**Detection Patterns**:
```ruby
# DANGEROUS
Digest::MD5.hexdigest(password)
Digest::SHA1.hexdigest(password)
Base64.encode64(secret)  # Not encryption!
```

**Safe Alternatives**:
```ruby
# Use bcrypt (via has_secure_password)
BCrypt::Password.create(password)

# For encryption
ActiveSupport::MessageEncryptor
```

**Severity**: High

---

### 10. Session Security

**Detection Patterns**:
```ruby
# Check session configuration
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, 
  key: '_app_session'
  # Missing: secure: true, httponly: true, same_site: :lax
```

**Safe Configuration**:
```ruby
Rails.application.config.session_store :cookie_store,
  key: '_app_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 30.minutes
```

**Severity**: Medium

---

### 11. Cross-Site Request Forgery (CSRF)

**Detection Patterns**:
```ruby
# DANGEROUS - Skipping CSRF
class ApiController < ApplicationController
  skip_before_action :verify_authenticity_token
end

# DANGEROUS - Not using form helpers
<form action="/posts" method="post">
  # Missing CSRF token
</form>
```

**Safe Alternatives**:
```ruby
# Use proper token handling for APIs
class ApiController < ApplicationController
  protect_from_forgery with: :null_session
end

# Use Rails form helpers
<%= form_with url: posts_path do |f| %>
```

**Severity**: Medium

---

### 12. Redirect Security

**Detection Patterns**:
```ruby
# DANGEROUS - Open redirect
redirect_to params[:return_to]
redirect_to request.referer
```

**Safe Alternatives**:
```ruby
# Validate redirect URLs
redirect_to url_for(params[:return_to]) rescue redirect_to root_path

# Or use allowlist
ALLOWED_REDIRECTS = %w[/dashboard /profile /settings]
redirect_to params[:return_to] if ALLOWED_REDIRECTS.include?(params[:return_to])
```

**Severity**: Medium

---

## Security Audit Checklist

### Authentication
- [ ] All sensitive actions require authentication
- [ ] Password requirements enforced (length, complexity)
- [ ] Account lockout after failed attempts
- [ ] Secure password reset flow
- [ ] Session timeout configured

### Authorization
- [ ] Resources scoped to authorized users
- [ ] Admin actions protected
- [ ] Role-based access control where needed

### Input Validation
- [ ] All user input validated
- [ ] Strong parameters used
- [ ] File upload restrictions (type, size)
- [ ] No SQL interpolation

### Output Encoding
- [ ] No `raw` or `html_safe` with user input
- [ ] JSON responses don't expose sensitive data
- [ ] Logs filtered for sensitive data

### Configuration
- [ ] HTTPS enforced in production
- [ ] Secure session configuration
- [ ] CSRF protection enabled
- [ ] Security headers configured (CSP, X-Frame-Options, etc.)

### Dependencies
- [ ] Gemfile.lock reviewed for vulnerabilities
- [ ] Using `bundler-audit` or similar

---

## Severity Reference

| Severity | Impact | Examples |
|----------|--------|----------|
| Critical | Data breach, full system compromise | SQL injection, RCE, authentication bypass |
| High | Significant data exposure or modification | XSS, IDOR, mass assignment |
| Medium | Limited impact, requires interaction | CSRF, open redirect, session issues |
| Low | Minor issues, defense in depth | Missing headers, verbose errors |

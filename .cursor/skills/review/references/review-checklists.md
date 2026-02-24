# Code Review Checklists

Standard checklists for different review scenarios.

## Quick Reference

Use these checklists based on review context:
- **New Feature**: Use "Feature Review Checklist"
- **Bug Fix**: Use "Bug Fix Review Checklist"
- **Refactoring**: Use "Refactoring Review Checklist"
- **Pre-Deployment**: Use "Pre-Deployment Checklist"
- **Security Review**: Use "Security Review Checklist"

---

## Feature Review Checklist

### Compliance (MUST HAVE)
- [ ] All MessageResponseDict enum entries defined (no hardcoded messages)
- [ ] Success responses use code=0
- [ ] Error responses use code 100000-400000
- [ ] HttpStatus is OK by default (unless specific requirement)
- [ ] No database migration files (DDL) added or modified
- [ ] Exceptions thrown with MessageResponseDict enum

### Architecture
- [ ] Follows Controller → Service → Repository pattern
- [ ] No business logic in controllers
- [ ] Proper dependency injection (constructor preferred)
- [ ] No circular dependencies
- [ ] Appropriate use of interfaces (only when needed)

### Data Access
- [ ] @Transactional on methods with multiple DB operations
- [ ] No N+1 queries (check for loops with lazy loading)
- [ ] EntityGraph or JOIN FETCH used appropriately
- [ ] DTOs used instead of entities in API responses
- [ ] Proper pagination for list endpoints

### Error Handling
- [ ] All exceptions handled in @ControllerAdvice
- [ ] Specific exception types (not generic Exception)
- [ ] Proper logging with context (userId, businessId)
- [ ] No sensitive data in logs
- [ ] Business errors use log.warn(), system errors use log.error()

### API Design
- [ ] Correct HTTP methods (GET/POST/PUT/DELETE)
- [ ] Proper status codes (200, 201, 204, 400, 404, etc.)
- [ ] Input validation with @Valid
- [ ] Consistent response structure (ResponseCommon)
- [ ] API versioning considered if needed

### Security
- [ ] Input sanitization/validation
- [ ] Authorization checks on endpoints
- [ ] No SQL injection (parameterized queries)
- [ ] Sensitive data not logged or exposed

### Code Quality
- [ ] Methods under 100 lines
- [ ] Clear naming (Vietnamese OK per standards)
- [ ] No code duplication
- [ ] Appropriate comments for complex logic
- [ ] Logging at appropriate levels

### Testing
- [ ] Unit tests for business logic
- [ ] Integration tests for API endpoints
- [ ] Edge cases covered
- [ ] Happy path and error scenarios tested

---

## Bug Fix Review Checklist

### Root Cause
- [ ] Root cause identified and documented
- [ ] Fix addresses root cause, not just symptoms

### Impact Analysis
- [ ] Areas affected by the bug identified
- [ ] Regression risk assessed
- [ ] Side effects considered

### Fix Quality
- [ ] Minimal code changes (focused fix)
- [ ] No new features added during bug fix
- [ ] Error handling improved if relevant
- [ ] Logging added to prevent recurrence

### Validation
- [ ] Bug reproduction steps verified
- [ ] Fix tested against reproduction steps
- [ ] Related test cases added/updated
- [ ] No new violations of coding standards

---

## Refactoring Review Checklist

### Scope
- [ ] Refactoring scope is clear and limited
- [ ] No behavior changes (pure refactoring)
- [ ] All tests still pass

### Code Improvements
- [ ] Code is more readable
- [ ] Complexity reduced
- [ ] Duplication eliminated
- [ ] Method/class responsibilities clearer

### Standards Compliance
- [ ] Follows all coding standards
- [ ] MessageResponseDict usage consistent
- [ ] Logging standards maintained
- [ ] Transaction boundaries preserved

### Risk Assessment
- [ ] Changes are low-risk
- [ ] No impact on database schema
- [ ] No breaking API changes

---

## Pre-Deployment Checklist

### Code Quality
- [ ] No TODO or FIXME comments without tickets
- [ ] No commented-out code blocks
- [ ] No debug/print statements
- [ ] No hardcoded values (use configuration)

### Configuration
- [ ] Environment-specific configs externalized
- [ ] Sensitive data in environment variables
- [ ] Database connection pool properly configured
- [ ] Logging configuration appropriate for production

### Database
- [ ] NO new migration files (verify no V*.sql added)
- [ ] All required indexes exist
- [ ] Database queries optimized
- [ ] Connection pooling configured

### Performance
- [ ] No obvious performance bottlenecks
- [ ] Appropriate caching implemented
- [ ] Async processing for long operations
- [ ] Pagination on all list endpoints

### Security
- [ ] Authentication/authorization on all endpoints
- [ ] Input validation comprehensive
- [ ] CORS configured appropriately
- [ ] No secrets in code or logs

### Monitoring
- [ ] Sufficient logging for production debugging
- [ ] Log levels appropriate (INFO for production)
- [ ] Error scenarios logged with context
- [ ] Performance-sensitive operations logged

### Documentation
- [ ] API documentation updated (Swagger/OpenAPI)
- [ ] README updated if needed
- [ ] Environment variables documented
- [ ] Deployment steps documented

---

## Security Review Checklist

### Authentication & Authorization
- [ ] All endpoints require authentication
- [ ] Role-based access control implemented
- [ ] User context properly validated
- [ ] No authorization bypass possible

### Input Validation
- [ ] All user inputs validated
- [ ] Type checking enforced
- [ ] Length/size limits enforced
- [ ] Whitelist validation where possible

### SQL Injection Prevention
- [ ] All queries use parameterized statements
- [ ] No string concatenation in queries
- [ ] ORM used correctly (JPA/Hibernate)
- [ ] Dynamic queries properly validated

### Data Protection
- [ ] Passwords never logged
- [ ] Tokens never logged
- [ ] Personal data handling compliant
- [ ] Sensitive fields encrypted at rest

### API Security
- [ ] CORS configured restrictively
- [ ] CSRF protection enabled where needed
- [ ] Rate limiting considered
- [ ] API keys/tokens properly managed

### Error Handling
- [ ] No stack traces exposed to clients
- [ ] Generic error messages for users
- [ ] Detailed errors logged server-side
- [ ] No information leakage in errors

---

## Business Logic Review Checklist

### Requirements Validation
- [ ] All requirements implemented
- [ ] Edge cases handled
- [ ] Business rules enforced
- [ ] Data validation complete

### Transaction Integrity
- [ ] @Transactional where needed
- [ ] All-or-nothing operations guaranteed
- [ ] Rollback scenarios handled
- [ ] Idempotency considered

### Data Consistency
- [ ] Concurrent access handled
- [ ] Race conditions prevented
- [ ] Database constraints enforced
- [ ] Foreign key relationships maintained

### Error Scenarios
- [ ] All failure paths tested
- [ ] Proper error messages
- [ ] Graceful degradation
- [ ] Recovery mechanisms in place

---

## Performance Review Checklist

### Database Optimization
- [ ] No N+1 queries
- [ ] Appropriate indexes used
- [ ] Query result set limited
- [ ] Lazy loading used correctly

### Caching
- [ ] Cache for frequently accessed data
- [ ] Cache invalidation strategy
- [ ] Cache key design appropriate
- [ ] TTL configured properly

### API Performance
- [ ] Pagination on all lists
- [ ] Response size reasonable
- [ ] Async for independent operations
- [ ] Bulk operations where appropriate

### Resource Management
- [ ] Connection pools configured
- [ ] Threads managed properly
- [ ] Memory usage reasonable
- [ ] File handles closed

---

## How to Use These Checklists

1. **Select appropriate checklist** based on review type
2. **Check each item systematically**
3. **Mark issues in review report** with severity:
   - Missing MUST HAVE items → 🔴 CRITICAL
   - Security/performance issues → 🟡 WARNING
   - Code quality improvements → 🔵 SUGGESTION
4. **Reference specific line numbers** for each issue
5. **Provide fix examples** for critical items

## Combining Checklists

For comprehensive reviews, combine multiple checklists:
- New Feature = Feature Review + Security Review + Performance Review
- Pre-Release = Pre-Deployment + Security Review + Performance Review
- Major Refactoring = Refactoring Review + Code Quality + Performance Review
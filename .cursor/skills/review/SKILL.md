---
name: bhxh-code-review
description: Comprehensive Spring Boot code review skill for BHXH project. Use when reviewing Java code, commits, pull requests, or packages against coding standards (bak_prompt_technical.md). Validates compliance with MessageResponseDict usage, exception handling, transaction management, logging standards, API conventions, security rules, and Spring Boot best practices. Identifies business logic errors, performance issues, and architectural problems.
---

# BHXH Code Review Skill

Review Spring Boot code for the BHXH project against established coding standards and best practices.

## Quick Start

**Basic usage:**
```
Review this code/commit/PR against the coding standards in prompt_technical.md
```

**With specific focus:**
```
Review UserService.java focusing on transaction management and exception handling
```

**With checklist:**
```
Review this PR against the deployment checklist to verify all requirements are met
```

## Review Process

Follow these steps in order:

### 1. Load Coding Standards
- Read `references/prompt_technical.md` to understand all coding rules
- Pay special attention to mandatory requirements (marked with ❌ or ✅)

### 2. Identify Review Scope
Determine what to review:
- **Single file**: Focus on that file
- **Package/directory**: Review all Java files in scope
- **Commit/PR**: Analyze changed files only
- **With checklist**: Cross-reference against checklist items

### 3. Execute Systematic Review
Run checks in this order:

#### A. Critical Compliance (MUST FIX)
Check these first as they are mandatory:
- MessageResponseDict enum usage (no hardcoded messages/codes)
- Database migration rules (NO DDL changes)
- Exception handling patterns
- Transaction management
- Response code standards (0 for success, 100000-400000 for errors)

#### B. Architecture & Design
- Layer separation (Controller → Service → Repository)
- Dependency injection
- Circular dependencies
- Interface usage appropriateness

#### C. Spring Boot Patterns
See `references/spring-boot-antipatterns.md` for detailed checks:
- Transaction boundaries and propagation
- JPA/Entity issues (N+1, lazy loading, bidirectional relationships)
- Exception handling (@ControllerAdvice, proper exception types)
- REST API design (HTTP methods, status codes, validation)

#### D. Performance & Quality
- Database query efficiency
- Memory leak potential
- Logging standards (format, levels, context)
- Code organization and maintainability

#### E. Security
- Input validation
- SQL injection prevention
- Sensitive data handling
- Authentication/authorization checks

### 4. Generate Structured Report

Use this exact format:

```
## 🔴 CRITICAL ISSUES (Must Fix Immediately)

### Issue #1: [Title]
**Location:** [File]:[Line]
**Rule Violated:** [Specific rule from prompt_technical.md]
**Issue:** [Description]
**Impact:** [Business/performance/security impact]
**Fix:**
[Code example showing before/after]

## 🟡 WARNINGS (Should Fix Soon)

### Warning #1: [Title]
[Same structure as Critical Issues]

## 🔵 SUGGESTIONS (Nice to Have)

### Suggestion #1: [Title]
**Location:** [File]:[Line]
**Suggestion:** [Improvement opportunity]
**Benefit:** [Why this helps]

## 📊 SUMMARY
- Total issues: [number]
- Critical: [number] 
- Warnings: [number]
- Suggestions: [number]
- Compliance score: [X/10]
- Top 3 priorities: [list]

## ✅ POSITIVE FINDINGS
[What was done well]
```

## Common Review Patterns

### Pattern 1: MessageResponseDict Compliance
**Check for:**
- All exceptions throw with MessageResponseDict enum
- No hardcoded messages: ~~`throw new Exception("Error")`~~
- Success code = 0, error codes 100000-400000
- HttpStatus defaults to OK unless specified

**Example violations:**
```java
// ❌ BAD
throw new BusinessException("User not found");
return new ResponseCommon(400, "Invalid input");

// ✅ GOOD  
throw new BusinessException(MessageResponseDict.USER_NOT_FOUND);
return ResponseUtils.create(MessageResponseDict.ADD_SUCCESS, data);
```

### Pattern 2: Transaction Management
**Check for:**
- @Transactional on multi-DB-operation methods
- Self-invocation issues (calling @Transactional from same class)
- Transaction boundaries (not too broad/narrow)

### Pattern 3: Logging Standards
**Check format:**
```
[{userId}/{businessId}] {action} - {details}
```

**Check levels:**
- `log.error()`: System errors with stacktrace
- `log.warn()`: Business errors without stacktrace  
- `log.info()`: State changes, important operations
- `log.debug()`: Detailed debugging (disabled in production)

### Pattern 4: N+1 Query Detection
```java
// ❌ BAD - N+1 problem
List<User> users = userRepository.findAll();
for(User user : users) {
    user.getOrders().size(); // Triggers N queries
}

// ✅ GOOD
@EntityGraph(attributePaths = "orders")
List<User> users = userRepository.findAll();
```

## Cross-Reference with Business Logic

When provided with business requirements or checklist:
1. Verify each requirement has corresponding implementation
2. Check edge cases are handled
3. Validate business rules enforcement
4. Confirm proper data validation

## Output Guidelines

- Be specific: Always include file names and line numbers
- Be actionable: Provide concrete fix examples
- Prioritize: Critical issues first
- Reference rules: Cite specific sections from prompt_technical.md
- Be concise: Focus on impact, not lengthy explanations
- Show examples: Include before/after code for fixes

## Resources

### references/
- `prompt_technical.md`: Complete coding standards (loaded from user upload)
- `spring-boot-antipatterns.md`: Common Spring Boot mistakes and fixes
- `review-checklists.md`: Standard review checklists for different scenarios
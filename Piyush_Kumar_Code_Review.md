# Code Review: Celonis Backend Challenge Solution

**Reviewed by:** Staff Software Engineer (10+ years Java experience)  
**Date:** 2024  
**Review Scope:** Design, Performance, Memory Management, Thread Safety, Best Practices

---

## Executive Summary

The solution implements the required functionality but has **critical issues** in thread safety, performance, memory management, and code quality that would prevent this from being production-ready. The code demonstrates understanding of Spring Boot basics but lacks the sophistication expected at a senior level.

**Overall Assessment:** ⚠️ **Requires Significant Improvements**

**Critical Issues Found:** 8  
**Major Issues Found:** 12  
**Minor Issues Found:** 8

---

## 1. CRITICAL ISSUES

### 1.1 Thread Safety Violation in CounterTaskService
**Severity:** 🔴 CRITICAL  
**Location:** `CounterTaskService.executeTask()`

**Problem:**
The `CounterTask` entity is shared across threads without proper synchronization. Multiple threads can access the same entity instance concurrently, leading to race conditions.

```java
// Line 60-68: Entity is accessed from background thread without locking
for (int i = counterTask.getX(); i <= counterTask.getY(); i++) {
    int progress = counterTask.getProgress() + 1;  // Race condition!
    counterTask.setProgress(progress);
    counterTaskRepository.save(counterTask);  // Not thread-safe
}
```

**Issues:**
- Entity instance is captured in lambda closure and modified from background thread
- No synchronization or locking mechanism
- Progress updates can be lost or corrupted
- Status updates can race with cancellation

**Impact:**
- Data corruption
- Lost progress updates
- Inconsistent task states
- Potential deadlocks

**Recommendation:**
- Fetch fresh entity instance from database in each iteration
- Use optimistic locking (`@Version`) on entity
- Use database-level updates (`@Modifying` queries) for atomic operations
- Consider using `AtomicInteger` for progress tracking with periodic DB sync

---

### 1.2 Incorrect Progress Calculation Logic
**Severity:** 🔴 CRITICAL  
**Location:** `CounterTaskService.executeTask()` and `buildProgressMessage()`

**Problem:**
The progress calculation is fundamentally flawed. It doesn't track the actual counter value correctly.

```java
// Line 60-71: Progress is incremented, but doesn't match the actual counter
for (int i = counterTask.getX(); i <= counterTask.getY(); i++) {
    int progress = counterTask.getProgress() + 1;  // Wrong!
    // Progress should track: (i - x + 1) / (y - x + 1) * 100
}
```

```java
// Line 103: Calculation assumes progress is percentage, but it's stored as count
int current = (int) (task.getX() + (task.getY() - task.getX()) * (task.getProgress() / 100.0));
```

**Issues:**
- Progress is stored as incrementing counter, not percentage
- Progress message calculation assumes percentage format
- Logic doesn't match requirement: "start from x, increase by one every second, reach y"

**Impact:**
- Incorrect progress reporting to clients
- Confusing user experience
- Progress can exceed 100%

**Recommendation:**
- Store actual counter value: `currentValue = i`
- Calculate percentage: `progress = ((currentValue - x) / (y - x + 1)) * 100`
- Or store percentage directly: `progress = ((i - x + 1) / (y - x + 1)) * 100`

---

### 1.3 ExecutorService Resource Leak
**Severity:** 🔴 CRITICAL  
**Location:** `CounterTaskService`

**Problem:**
`ExecutorService` is created but never shut down, causing thread and memory leaks.

```java
// Line 23: ExecutorService created but never shut down
private final ExecutorService executorService = Executors.newCachedThreadPool();
```

**Issues:**
- Application shutdown won't terminate background threads
- Threads remain active after application stops
- Memory leak from thread pool
- No graceful shutdown mechanism

**Impact:**
- Resource exhaustion
- Application won't shut down cleanly
- Production deployment issues

**Recommendation:**
- Implement `@PreDestroy` method to shutdown executor
- Or use Spring's `@Async` with `TaskExecutor` bean (managed by Spring)
- Use `CompletableFuture` with managed executor
- Configure proper thread pool size (cached thread pool is unbounded)

---

### 1.4 Performance: Database Save in Loop
**Severity:** 🔴 CRITICAL  
**Location:** `CounterTaskService.executeTask()`

**Problem:**
Database save is called inside a tight loop (once per second for potentially hours), causing severe performance issues.

```java
// Line 68: DB save in every iteration
counterTaskRepository.save(counterTask);  // Called (y-x+1) times!
Thread.sleep(1000);
```

**Issues:**
- For task counting from 0 to 86400 (1 day), 86,401 database writes occur
- No batching or periodic updates
- Unnecessary database load
- Slower execution than needed

**Impact:**
- Database performance degradation
- High latency
- Poor scalability
- Cost implications in cloud environments

**Recommendation:**
- Save periodically (every N seconds or every M progress updates)
- Use batch updates for progress
- Consider in-memory cache with periodic flush
- Use database triggers or stored procedures for progress updates

---

### 1.5 Race Condition in Task Cancellation
**Severity:** 🔴 CRITICAL  
**Location:** `CounterTaskService.cancelTask()`

**Problem:**
Race condition between cancellation check and task execution.

```java
// Line 89-99: Race condition possible
public void cancelTask(String taskId) {
    Future<?> futureTask = runningTasks.get(taskId);  // Not atomic with status check
    if (futureTask != null && !futureTask.isDone()) {
        futureTask.cancel(Boolean.TRUE);
    }
    CounterTask counterTask = counterTaskRepository.findById(taskId).orElseThrow(NotFoundException::new);
    if (counterTask.getStatus() == TaskStatus.RUNNING) {  // Status might have changed
        counterTask.setStatus(TaskStatus.CANCELLED);
        counterTaskRepository.save(counterTask);
    }
}
```

**Issues:**
- Task might complete between `runningTasks.get()` and status check
- Status might be updated by execution thread simultaneously
- No synchronization between cancellation and execution
- `runningTasks.remove()` called in multiple places

**Impact:**
- Tasks might not cancel properly
- Inconsistent state
- Memory leak in `runningTasks` map

**Recommendation:**
- Use atomic operations or synchronized blocks
- Check status atomically with cancellation
- Remove from map in single location
- Use `compareAndSet` pattern with status field

---

### 1.6 Missing Input Validation
**Severity:** 🔴 CRITICAL  
**Location:** `CounterTaskController.createAndStartTask()`

**Problem:**
No validation for input parameters, allowing invalid states.

```java
// Line 25: No validation
public ResponseEntity<?> createAndStartTask(@RequestParam int x, @RequestParam int y)
```

**Issues:**
- No check if `y < x`
- No check for negative values
- No check for extremely large values (could cause OOM or infinite loops)
- No maximum range validation

**Impact:**
- Application errors or hangs
- Resource exhaustion
- Poor user experience
- Security vulnerabilities

**Recommendation:**
- Add `@Min` and `@Max` annotations
- Add validation: `y >= x`
- Add reasonable bounds: `y - x <= MAX_COUNT`
- Return proper error messages

---

### 1.7 Memory Leak: Temporary Files Not Cleaned
**Severity:** 🔴 CRITICAL  
**Location:** `FileService.storeResult()`

**Problem:**
Temporary files created with `deleteOnExit()` are never explicitly cleaned up.

```java
// Line 46-47: Files marked for deletion on exit only
File outputFile = File.createTempFile(taskId, ".zip");
outputFile.deleteOnExit();  // Only deleted on JVM shutdown!
```

**Issues:**
- Files accumulate until JVM shutdown
- No cleanup after task completion/failure
- Disk space exhaustion
- `deleteOnExit()` has known issues with large number of files

**Impact:**
- Disk space exhaustion
- File system issues
- Production outages

**Recommendation:**
- Delete files after download or after expiration period
- Implement proper cleanup service
- Use file system monitoring
- Consider using temporary file service with TTL

---

### 1.8 Hardcoded Security Secret
**Severity:** 🔴 CRITICAL  
**Location:** `SimpleHeaderFilter`

**Problem:**
Security token is hardcoded in source code.

```java
// Line 15-16: Hardcoded secret
private final String HEADER_NAME = "Celonis-Auth";
private final String HEADER_VALUE = "totally_secret";  // Hardcoded!
```

**Issues:**
- Secret exposed in source code
- Cannot be changed without code deployment
- Security vulnerability

**Impact:**
- Security breach
- Compliance violations
- Cannot rotate credentials

**Recommendation:**
- Use Spring properties or environment variables
- Use Spring Cloud Config or secrets management
- Never commit secrets to version control

---

## 2. MAJOR ISSUES

### 2.1 Missing Transaction Management
**Severity:** 🟠 MAJOR  
**Location:** Multiple service methods

**Problem:**
No `@Transactional` annotations on methods that perform multiple database operations.

```java
// CounterTaskService.createTask() - no @Transactional
// CounterTaskService.executeTask() - no @Transactional
// CounterTaskService.cancelTask() - no @Transactional
```

**Issues:**
- Inconsistent data states on failures
- Partial updates possible
- No rollback capability

**Recommendation:**
- Add `@Transactional` to service methods
- Use appropriate isolation levels
- Handle rollback scenarios

---

### 2.2 Inconsistent Dependency Injection
**Severity:** 🟠 MAJOR  
**Location:** `CounterTaskController` vs `TaskController`

**Problem:**
Mixed injection styles - field injection vs constructor injection.

```java
// CounterTaskController uses @Autowired field injection
@Autowired
private CounterTaskService counterTaskService;

// TaskController uses constructor injection (preferred)
public TaskController(TaskService taskService, FileService fileService)
```

**Issues:**
- Field injection is harder to test
- Field injection hides dependencies
- Inconsistent with best practices

**Recommendation:**
- Use constructor injection consistently
- Remove `@Autowired` annotations (not needed in Spring 4.3+)

---

### 2.3 Deprecated API Usage
**Severity:** 🟠 MAJOR  
**Location:** `CorsConfiguration`

**Problem:**
Using deprecated `WebMvcConfigurerAdapter`.

```java
// Line 10: Deprecated class
public class CorsConfiguration extends WebMvcConfigurerAdapter
```

**Issues:**
- Deprecated API may be removed in future versions
- Not following modern Spring practices

**Recommendation:**
- Implement `WebMvcConfigurer` interface directly (Spring 5+)

---

### 2.4 Using Legacy Date API
**Severity:** 🟠 MAJOR  
**Location:** Entity classes

**Problem:**
Using `java.util.Date` instead of `java.time.*` APIs.

```java
// CounterTask.java, ProjectGenerationTask.java
private Date creationDate;  // Legacy API
```

**Issues:**
- `Date` is mutable and thread-unsafe
- Poor API design
- Timezone handling issues
- Deprecated in favor of `java.time.*`

**Recommendation:**
- Use `LocalDateTime` or `Instant`
- Use JPA `@CreationTimestamp` for automatic date setting

---

### 2.5 Generic Exception Handling
**Severity:** 🟠 MAJOR  
**Location:** `ErrorController.handleInternalError()`

**Problem:**
Catching all exceptions generically hides important errors.

```java
// Line 32-36: Catches all exceptions
@ExceptionHandler(Exception.class)
public String handleInternalError(Exception e) {
    logger.error("Unhandled Exception in Controller", e);
    return "Internal error";  // Generic message
}
```

**Issues:**
- Hides root causes
- Difficult to debug
- Poor user experience
- Security risk (information leakage)

**Recommendation:**
- Handle specific exceptions
- Provide detailed error messages for known exceptions
- Log stack traces appropriately
- Return proper error DTOs

---

### 2.6 Poor Exception Design
**Severity:** 🟠 MAJOR  
**Location:** Custom exception classes

**Problem:**
Exceptions don't accept or store messages properly.

```java
// NotAuthorizedException.java - no constructors
public class NotAuthorizedException extends RuntimeException {
}

// NotFoundException.java - no constructors
public class NotFoundException extends RuntimeException {
}
```

**Issues:**
- Cannot provide context-specific error messages
- Poor debugging experience
- Inconsistent with `InternalException` which has constructors

**Recommendation:**
- Add constructors accepting messages
- Add constructors accepting cause
- Follow exception best practices

---

### 2.7 No Cleanup for ProjectGenerationTask
**Severity:** 🟠 MAJOR  
**Location:** `CounterTaskCleanUpService`

**Problem:**
Cleanup service only handles `CounterTask`, not `ProjectGenerationTask`.

```java
// Only cleans up CounterTask
List<CounterTask> oldTasks = counterTaskRepository.findByStatusAndCreationDateBefore(...);
```

**Issues:**
- Requirement states cleanup for all tasks
- `ProjectGenerationTask` files will accumulate
- Incomplete implementation

**Recommendation:**
- Add cleanup for `ProjectGenerationTask`
- Delete associated files when cleaning up
- Make cleanup generic for all task types

---

### 2.8 Incomplete Cleanup Logic
**Severity:** 🟠 MAJOR  
**Location:** `CounterTaskCleanUpService.cleanupOldTasks()`

**Problem:**
Only cleans up tasks with `CREATED` status, but requirement says "tasks that are not executed".

```java
// Line 33: Only CREATED status
List<CounterTask> oldTasks = counterTaskRepository.findByStatusAndCreationDateBefore(
    TaskStatus.CREATED, startDate);
```

**Issues:**
- Should clean up any task that hasn't been executed (not just CREATED)
- Missing executed tasks that are old
- Incomplete requirement implementation

**Recommendation:**
- Clean up tasks based on execution status, not just creation status
- Or clarify: only CREATED tasks need cleanup
- Consider archiving completed tasks before deletion

---

### 2.9 Missing Return Type in Controller
**Severity:** 🟠 MAJOR  
**Location:** `CounterTaskController.cancelTask()`

**Problem:**
Endpoint returns `void` but should return proper response.

```java
// Line 39-43: Returns void
@PostMapping("/{taskId}/cancel")
public void cancelTask(@PathVariable String taskId)
```

**Issues:**
- No indication of success/failure
- HTTP status unclear
- Poor API design

**Recommendation:**
- Return `ResponseEntity` with appropriate status
- Return success message or task status
- Handle case when task doesn't exist

---

### 2.10 Duplicate Log Statement
**Severity:** 🟠 MAJOR  
**Location:** `CounterTaskController.getProgress()`

**Problem:**
Duplicate log statement.

```java
// Line 34-35: Duplicate log
log.info("CounterTaskController::getProgress() for taskId={}", taskId);
log.info("CounterTaskController::getProgress() for taskId={}", taskId);
```

**Issues:**
- Code quality issue
- Unnecessary logging overhead

**Recommendation:**
- Remove duplicate line

---

### 2.11 Inconsistent Entity Design
**Severity:** 🟠 MAJOR  
**Location:** Entity classes

**Problem:**
`CounterTask` uses manual getters/setters while `TaskProgressResponse` uses Lombok.

```java
// CounterTask - manual getters/setters
public String getId() { return id; }
public void setId(String id) { this.id = id; }

// TaskProgressResponse - Lombok
@Data
@Builder
public class TaskProgressResponse {
```

**Issues:**
- Inconsistent code style
- More boilerplate code
- Harder to maintain

**Recommendation:**
- Use Lombok consistently (`@Data`, `@Entity`)
- Or avoid Lombok entirely for consistency

---

### 2.12 Missing Entity Validation
**Severity:** 🟠 MAJOR  
**Location:** Entity classes and controllers

**Problem:**
No JPA validation annotations on entities.

```java
// CounterTask - no validation
private int x;
private int y;  // Should validate y >= x
```

**Issues:**
- Invalid data can be persisted
- Business rule violations
- Data integrity issues

**Recommendation:**
- Add `@NotNull`, `@Min`, `@Max` annotations
- Add custom validators for complex rules
- Validate at entity and DTO levels

---

## 3. MINOR ISSUES

### 3.1 Code Style: Unnecessary Modifier
**Severity:** 🟡 MINOR  
**Location:** `SimpleHeaderFilter`

```java
// Line 15-16: Should be final and static final respectively
private final String HEADER_NAME = "Celonis-Auth";
private final String HEADER_VALUE = "totally_secret";
```

**Recommendation:**
- Make `HEADER_NAME` and `HEADER_VALUE` `static final` constants

---

### 3.2 Missing Javadoc
**Severity:** 🟡 MINOR  
**Location:** All classes and methods

**Problem:**
No documentation for public APIs.

**Recommendation:**
- Add JavaDoc for public methods
- Document parameters and return values
- Add class-level documentation

---

### 3.3 Magic Numbers
**Severity:** 🟡 MINOR  
**Location:** `CounterTaskService`

```java
// Line 31: Magic number
task.setName("CT-" + (System.currentTimeMillis() % 10000));
```

**Recommendation:**
- Extract constants
- Use meaningful names

---

### 3.4 CORS Configuration Too Permissive
**Severity:** 🟡 MINOR  
**Location:** `CorsConfiguration`

```java
// Line 14: Allows all origins
registry.addMapping("/**");
```

**Recommendation:**
- Configure specific origins
- Add credentials configuration
- Restrict allowed methods

---

### 3.5 Inefficient Name Generation
**Severity:** 🟡 MINOR  
**Location:** `CounterTaskService.createTask()`

```java
// Line 31: Modulo operation might cause collisions
task.setName("CT-" + (System.currentTimeMillis() % 10000));
```

**Recommendation:**
- Use UUID or sequential ID
- Or use full timestamp

---

### 3.6 Missing Error Messages in Exceptions
**Severity:** 🟡 MINOR  
**Location:** Exception throwing

```java
// Line 42: No error message
.orElseThrow(NotFoundException::new);
```

**Recommendation:**
- Include task ID in exception message
- Provide context for debugging

---

### 3.7 Unused Import
**Severity:** 🟡 MINOR  
**Location:** `FileService`

```java
// Line 7: Unused import
import org.apache.commons.io.IOUtils;
```

Actually used, but verify all imports are necessary.

---

### 3.8 Missing Application Properties
**Severity:** 🟡 MINOR  
**Location:** Configuration

**Problem:**
No `application.properties` or `application.yml` file visible.

**Recommendation:**
- Add configuration file
- Externalize configuration values
- Document configuration options

---

## 4. DESIGN ISSUES

### 4.1 Lack of Abstraction
**Problem:**
Two separate task types (`CounterTask`, `ProjectGenerationTask`) with similar functionality but no common interface or base class.

**Recommendation:**
- Create `Task` interface or base class
- Use strategy pattern for task execution
- Generic task service

### 4.2 Tight Coupling
**Problem:**
Services are tightly coupled to specific repository implementations.

**Recommendation:**
- Use interfaces for repositories
- Consider repository pattern with interfaces
- Enable easier testing and mocking

### 4.3 No Retry Mechanism
**Problem:**
No retry logic for transient failures.

**Recommendation:**
- Implement retry mechanism for database operations
- Use Spring Retry or resilience patterns

---

## 5. PERFORMANCE ISSUES

### 5.1 Database Queries in Loop
**Location:** `CounterTaskService.executeTask()`

**Problem:**
Database save in every iteration (mentioned in Critical Issues).

### 5.2 No Connection Pooling Configuration
**Problem:**
No explicit HikariCP or connection pool configuration.

**Recommendation:**
- Configure connection pool
- Set appropriate pool size
- Monitor connection usage

### 5.3 No Caching
**Problem:**
No caching for frequently accessed data.

**Recommendation:**
- Consider caching task metadata
- Use Spring Cache abstraction
- Implement appropriate cache eviction

---

## 6. MEMORY MANAGEMENT ISSUES

### 6.1 Unbounded Thread Pool
**Location:** `CounterTaskService`

**Problem:**
`Executors.newCachedThreadPool()` creates unbounded thread pool.

**Impact:**
- Memory exhaustion with many concurrent tasks
- Thread starvation

**Recommendation:**
- Use bounded thread pool: `Executors.newFixedThreadPool(n)`
- Configure based on system resources
- Use `ThreadPoolTaskExecutor` managed by Spring

### 6.2 Unbounded Map Growth
**Location:** `CounterTaskService.runningTasks`

**Problem:**
`ConcurrentHashMap` grows unbounded, completed tasks not always removed.

**Recommendation:**
- Implement cleanup mechanism
- Use TTL-based cache (Caffeine, Guava)
- Periodic cleanup of completed tasks

### 6.3 Entity Loading
**Problem:**
Entities loaded and held in memory during long-running tasks.

**Recommendation:**
- Use DTOs instead of entities for long-lived objects
- Detach entities when not needed
- Use projections for read operations

---

## 7. TESTING CONSIDERATIONS

### 7.1 No Test Code
**Problem:**
No unit tests or integration tests visible.

**Recommendation:**
- Add comprehensive test suite
- Test concurrent execution scenarios
- Test cancellation and cleanup
- Integration tests for API endpoints

### 7.2 Hard to Test Code
**Problem:**
Tight coupling and static dependencies make testing difficult.

**Recommendation:**
- Improve dependency injection
- Mock external dependencies
- Use test profiles

---

## 8. SUMMARY OF RECOMMENDATIONS

### Priority 1 (Must Fix - Critical)
1. ✅ Fix thread safety issues in `CounterTaskService`
2. ✅ Fix progress calculation logic
3. ✅ Add ExecutorService shutdown mechanism
4. ✅ Optimize database saves in loop
5. ✅ Fix race conditions in cancellation
6. ✅ Add input validation
7. ✅ Fix file cleanup
8. ✅ Move secrets to configuration

### Priority 2 (Should Fix - Major)
1. ✅ Add transaction management
2. ✅ Standardize dependency injection
3. ✅ Replace deprecated APIs
4. ✅ Use modern date/time APIs
5. ✅ Improve exception handling
6. ✅ Complete cleanup implementation
7. ✅ Fix API response types

### Priority 3 (Nice to Have - Minor)
1. ✅ Improve code documentation
2. ✅ Extract constants
3. ✅ Improve error messages
4. ✅ Add configuration file

---

## 9. CONCLUSION

The solution demonstrates basic Spring Boot knowledge and implements the required features. However, **critical thread safety, performance, and memory management issues** prevent this from being production-ready.

**Strengths:**
- Functional implementation
- Uses Spring Boot appropriately
- Basic structure is sound

**Weaknesses:**
- Thread safety violations
- Performance issues (DB saves in loop)
- Resource leaks
- Missing validation
- Security concerns

**Overall Grade:** C+ (Functional but needs significant improvement)

**Recommendation:** The code requires substantial refactoring before it can be considered production-ready. Focus on thread safety, performance optimization, and proper resource management.

---

## 10. CODE QUALITY METRICS

- **Lines of Code:** ~600
- **Cyclomatic Complexity:** Medium-High
- **Code Duplication:** Low
- **Test Coverage:** 0% (no tests visible)
- **Documentation:** Poor
- **Thread Safety:** Poor
- **Resource Management:** Poor

---

*Review completed with focus on design, performance, and memory management as requested.*


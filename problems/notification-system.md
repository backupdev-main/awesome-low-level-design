# Notification System

##  Functional Requirements

- The system should support sending notifications via EMAIL, SMS, and PUSH.
- Each notification targets a single recipient and a specific channel.
- The system should send notifications asynchronously.
- If sending fails, the system should retry the operation a few times before giving up.
- Notifications may contain a subject (optional) and a message body (mandatory).

## Non-Functional Requirements

- The system should follow object-oriented design with clear separation of concerns.
- It should be extensible, allowing future support for new notification types (e.g., WhatsApp, Slack).
- Delivery should be non-blocking, using a thread pool to manage parallel sending.

## Core Entities

1. **Notification**
   - Represents a message to be sent to a recipient
   - Contains: recipient, channel type, subject (optional), message body
   - Tracks delivery status and retry attempts

2. **NotificationChannel**
   - Abstraction for different notification delivery mechanisms
   - Implementations: EmailChannel, SMSChannel, PushChannel
   - Responsible for sending notifications through specific channels

3. **Recipient**
   - Represents the target user for the notification
   - Contains: recipient ID, contact details (email, phone, device ID)
   - Different channel types require different contact information

4. **NotificationService**
   - Main service orchestrating notification delivery
   - Manages notification queuing and dispatching
   - Handles retries and failure scenarios

5. **NotificationQueue**
   - Manages pending notifications awaiting delivery
   - Supports asynchronous processing
   - Ensures FIFO or priority-based delivery

6. **RetryPolicy**
   - Defines retry strategy for failed notifications
   - Contains: max retries, retry interval, backoff strategy

## Class Definitions

```java
// Enum for notification channels
public enum NotificationChannelType {
    EMAIL,
    SMS,
    PUSH
}

// Enum for notification delivery status
public enum NotificationStatus {
    PENDING,
    SENT,
    FAILED,
    RETRY
}

// Recipient class
public class Recipient {
    private String recipientId;
    private String email;
    private String phoneNumber;
    private String deviceId;
    
    public Recipient(String recipientId, String email, String phoneNumber, String deviceId) {
        this.recipientId = recipientId;
        this.email = email;
        this.phoneNumber = phoneNumber;
        this.deviceId = deviceId;
    }
    
    // Getters
    public String getRecipientId() { return recipientId; }
    public String getEmail() { return email; }
    public String getPhoneNumber() { return phoneNumber; }
    public String getDeviceId() { return deviceId; }
}

// Notification class
public class Notification {
    private String notificationId;
    private Recipient recipient;
    private NotificationChannelType channel;
    private String subject;
    private String message;
    private NotificationStatus status;
    private int retryCount;
    private LocalDateTime createdAt;
    private LocalDateTime sentAt;
    
    public Notification(String notificationId, Recipient recipient, 
                       NotificationChannelType channel, String subject, String message) {
        this.notificationId = notificationId;
        this.recipient = recipient;
        this.channel = channel;
        this.subject = subject;
        this.message = message;
        this.status = NotificationStatus.PENDING;
        this.retryCount = 0;
        this.createdAt = LocalDateTime.now();
    }
    
    // Getters and setters
    public String getNotificationId() { return notificationId; }
    public Recipient getRecipient() { return recipient; }
    public NotificationChannelType getChannel() { return channel; }
    public String getSubject() { return subject; }
    public String getMessage() { return message; }
    public NotificationStatus getStatus() { return status; }
    public void setStatus(NotificationStatus status) { this.status = status; }
    public int getRetryCount() { return retryCount; }
    public void incrementRetryCount() { this.retryCount++; }
}

// Abstract base class for notification channels
public abstract class NotificationChannel {
    public abstract boolean send(Notification notification);
}

// Email channel implementation
public class EmailChannel extends NotificationChannel {
    @Override
    public boolean send(Notification notification) {
        try {
            String email = notification.getRecipient().getEmail();
            String subject = notification.getSubject();
            String message = notification.getMessage();
            
            // Simulate email sending
            System.out.println("Sending email to: " + email);
            System.out.println("Subject: " + subject);
            System.out.println("Message: " + message);
            
            return true;
        } catch (Exception e) {
            System.err.println("Failed to send email: " + e.getMessage());
            return false;
        }
    }
}

// SMS channel implementation
public class SMSChannel extends NotificationChannel {
    @Override
    public boolean send(Notification notification) {
        try {
            String phoneNumber = notification.getRecipient().getPhoneNumber();
            String message = notification.getMessage();
            
            // Simulate SMS sending
            System.out.println("Sending SMS to: " + phoneNumber);
            System.out.println("Message: " + message);
            
            return true;
        } catch (Exception e) {
            System.err.println("Failed to send SMS: " + e.getMessage());
            return false;
        }
    }
}

// Push notification channel implementation
public class PushChannel extends NotificationChannel {
    @Override
    public boolean send(Notification notification) {
        try {
            String deviceId = notification.getRecipient().getDeviceId();
            String message = notification.getMessage();
            
            // Simulate push notification sending
            System.out.println("Sending push notification to device: " + deviceId);
            System.out.println("Message: " + message);
            
            return true;
        } catch (Exception e) {
            System.err.println("Failed to send push notification: " + e.getMessage());
            return false;
        }
    }
}

// Retry policy class
public class RetryPolicy {
    private int maxRetries;
    private long retryIntervalMs;
    private double backoffMultiplier;
    
    public RetryPolicy(int maxRetries, long retryIntervalMs, double backoffMultiplier) {
        this.maxRetries = maxRetries;
        this.retryIntervalMs = retryIntervalMs;
        this.backoffMultiplier = backoffMultiplier;
    }
    
    public int getMaxRetries() { return maxRetries; }
    public long getRetryIntervalMs() { return retryIntervalMs; }
    public double getBackoffMultiplier() { return backoffMultiplier; }
    
    public long getRetryIntervalForAttempt(int attemptNumber) {
        return (long) (retryIntervalMs * Math.pow(backoffMultiplier, attemptNumber - 1));
    }
}

// Notification service class
public class NotificationService {
    private Map<NotificationChannelType, NotificationChannel> channels;
    private Queue<Notification> notificationQueue;
    private ExecutorService executorService;
    private RetryPolicy retryPolicy;
    
    public NotificationService(int threadPoolSize, RetryPolicy retryPolicy) {
        this.channels = new HashMap<>();
        this.notificationQueue = new ConcurrentLinkedQueue<>();
        this.executorService = Executors.newFixedThreadPool(threadPoolSize);
        this.retryPolicy = retryPolicy;
        
        // Initialize channels
        initializeChannels();
    }
    
    private void initializeChannels() {
        channels.put(NotificationChannelType.EMAIL, new EmailChannel());
        channels.put(NotificationChannelType.SMS, new SMSChannel());
        channels.put(NotificationChannelType.PUSH, new PushChannel());
    }
    
    public void sendNotification(Recipient recipient, NotificationChannelType channel, 
                                 String subject, String message) {
        String notificationId = UUID.randomUUID().toString();
        Notification notification = new Notification(notificationId, recipient, channel, subject, message);
        notificationQueue.add(notification);
        
        // Submit async task to executor service
        executorService.submit(() -> processNotification(notification));
    }
    
    private void processNotification(Notification notification) {
        NotificationChannel channel = channels.get(notification.getChannel());
        
        if (channel == null) {
            System.err.println("Channel not supported: " + notification.getChannel());
            notification.setStatus(NotificationStatus.FAILED);
            return;
        }
        
        boolean sent = channel.send(notification);
        
        if (sent) {
            notification.setStatus(NotificationStatus.SENT);
            System.out.println("Notification " + notification.getNotificationId() + " sent successfully");
        } else {
            handleFailedNotification(notification);
        }
    }
    
    private void handleFailedNotification(Notification notification) {
        if (notification.getRetryCount() < retryPolicy.getMaxRetries()) {
            notification.setStatus(NotificationStatus.RETRY);
            notification.incrementRetryCount();
            
            long retryDelay = retryPolicy.getRetryIntervalForAttempt(notification.getRetryCount());
            
            executorService.schedule(() -> processNotification(notification), retryDelay, TimeUnit.MILLISECONDS);
        } else {
            notification.setStatus(NotificationStatus.FAILED);
            System.err.println("Notification " + notification.getNotificationId() + " failed after " + 
                             retryPolicy.getMaxRetries() + " retries");
        }
    }
    
    public void shutdown() {
        executorService.shutdown();
    }
}
```

## Class Relationships

```
┌─────────────────────────────────────────────────────────┐
│                  NotificationService                     │
│  - channels: Map<ChannelType, NotificationChannel>      │
│  - notificationQueue: Queue<Notification>               │
│  - executorService: ExecutorService                     │
│  - retryPolicy: RetryPolicy                             │
└────────────────────┬────────────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
    ┌─────────┐ ┌────────┐ ┌──────────┐
    │ Notif.  │ │RetryPol│ │Channels  │
    │         │ │        │ │(Abstract)│
    └─────────┘ └────────┘ └──────────┘
          │                     │
          │          ┌──────────┼──────────┐
          │          │          │          │
          ▼          ▼          ▼          ▼
    ┌──────────┐ ┌─────────┐ ┌──────┐ ┌────────┐
    │Recipient │ │Email    │ │ SMS  │ │ Push   │
    │          │ │Channel  │ │Channel│ │Channel │
    └──────────┘ └─────────┘ └──────┘ └────────┘

Relationships:
- NotificationService uses NotificationChannel (composition)
- NotificationService uses RetryPolicy (composition)
- Notification contains Recipient (composition)
- EmailChannel, SMSChannel, PushChannel extend NotificationChannel (inheritance)
```

## Key Design Patterns

### 1. **Strategy Pattern**
   - **Purpose**: Encapsulate different notification strategies (Email, SMS, Push)
   - **Implementation**: `NotificationChannel` abstract class with concrete implementations
   - **Benefit**: Easy to add new channels without modifying existing code
   - **Example**: Different channel types implement `send()` method differently

### 2. **Factory Pattern**
   - **Purpose**: Create appropriate channel instances based on type
   - **Implementation**: `initializeChannels()` method in `NotificationService`
   - **Benefit**: Centralizes object creation and simplifies channel management
   - **Example**: Map of channels keyed by `NotificationChannelType`

### 3. **Queue/Producer-Consumer Pattern**
   - **Purpose**: Decouple notification creation from processing
   - **Implementation**: `notificationQueue` for storing pending notifications
   - **Benefit**: Enables asynchronous processing and load balancing
   - **Example**: Notifications added to queue and processed by thread pool

### 4. **Retry Pattern with Exponential Backoff**
   - **Purpose**: Handle transient failures gracefully
   - **Implementation**: `RetryPolicy` class with configurable retry logic
   - **Benefit**: Improves reliability without overwhelming the system
   - **Example**: `getRetryIntervalForAttempt()` calculates increasing delays

### 5. **Thread Pool Pattern**
   - **Purpose**: Manage parallel notification delivery efficiently
   - **Implementation**: `ExecutorService` with fixed thread pool
   - **Benefit**: Controls resource consumption and enables non-blocking delivery
   - **Example**: `executorService.submit()` for async task submission

### 6. **Observer Pattern (Implicit)**
   - **Purpose**: Track notification status changes
   - **Implementation**: Notification status updates during lifecycle
   - **Benefit**: Allows monitoring and logging of delivery status
   - **Example**: Status transitions from PENDING → SENT/RETRY → FAILED

### 7. **Decorator Pattern (Extensible)**
   - **Purpose**: Add behavior to notifications (e.g., logging, encryption)
   - **Implementation**: Can wrap notification processing with additional handlers
   - **Benefit**: Supports cross-cutting concerns without modifying core logic
   - **Example**: Could add logging or encryption decorators   - **Example**: Could add logging or encryption decorators

## Demo: NotificationServiceDemo

```java
import java.util.concurrent.TimeUnit;

public class NotificationServiceDemo {
    public static void main(String[] args) {
        System.out.println("========== Notification System Demo ==========\n");
        
        // Step 1: Create retry policy
        RetryPolicy retryPolicy = new RetryPolicy(3, 1000, 2.0);
        
        // Step 2: Initialize notification service with thread pool
        NotificationService service = new NotificationService(5, retryPolicy);
        
        // Step 3: Create recipients with different contact information
        Recipient recipient1 = new Recipient(
            "user_001",
            "john.doe@example.com",
            "+1-555-0101",
            "device_uuid_001"
        );
        
        Recipient recipient2 = new Recipient(
            "user_002",
            "jane.smith@example.com",
            "+1-555-0102",
            "device_uuid_002"
        );
        
        Recipient recipient3 = new Recipient(
            "user_003",
            "bob.johnson@example.com",
            "+1-555-0103",
            "device_uuid_003"
        );
        
        // Step 4: Send notifications via different channels
        System.out.println("--- Sending Notifications ---\n");
        
        // Email notification
        service.sendNotification(
            recipient1,
            NotificationChannelType.EMAIL,
            "Welcome to Our Service",
            "Hello John! Thank you for signing up."
        );
        
        // SMS notification
        service.sendNotification(
            recipient2,
            NotificationChannelType.SMS,
            null,
            "Hi Jane! Your order has been confirmed."
        );
        
        // Push notification
        service.sendNotification(
            recipient3,
            NotificationChannelType.PUSH,
            null,
            "Bob, you have a new message from your friend."
        );
        
        System.out.println("\n--- Notifications Queued for Processing ---\n");
        
        // Step 5: Allow time for async processing
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("\n--- Processing Complete ---\n");
        
        // Step 6: Shutdown the service
        service.shutdown();
        
        System.out.println("Notification Service shut down successfully.");
    }
}
```

### Demo Execution Flow

**Input:**
- 3 Recipients with different contact details
- 3 Notifications through different channels (Email, SMS, Push)
- Retry policy: 3 max retries, 1000ms initial delay, 2x backoff multiplier

**Expected Output:**
```
========== Notification System Demo ==========

--- Sending Notifications ---

Sending email to: john.doe@example.com
Subject: Welcome to Our Service
Message: Hello John! Thank you for signing up.
Notification [notification_id_1] sent successfully

Sending SMS to: +1-555-0102
Message: Hi Jane! Your order has been confirmed.
Notification [notification_id_2] sent successfully

Sending push notification to device: device_uuid_003
Message: Bob, you have a new message from your friend.
Notification [notification_id_3] sent successfully

--- Notifications Queued for Processing ---

--- Processing Complete ---

Notification Service shut down successfully.
```

### Use Case Covered

**Success Scenario:**
- All notifications are successfully sent on the first attempt
- Email, SMS, and Push channels all function correctly
- Thread pool efficiently processes all notifications asynchronously
- No retries are needed as all operations succeed
- Service gracefully shuts down after all notifications are processed

### Key Observations

1. **Asynchronous Processing**: Notifications are submitted to the thread pool and processed concurrently
2. **Channel Abstraction**: Different delivery mechanisms are handled uniformly through the `NotificationChannel` interface
3. **Thread Safety**: `ConcurrentLinkedQueue` ensures safe concurrent access to the notification queue
4. **Resource Management**: Thread pool with fixed size (5 threads) controls resource consumption
5. **Extensibility**: New recipients and channels can be added without modifying existing code

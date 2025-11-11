---
layout: post
title: "Building Resilient Systems with Circuit Breakers and Exponential Backoff"
date: 2025-11-10 10:00:00 -0500
categories: engineering
author: Jose Rodriguez
---

When building systems that depend on external services, failures are not a matter of "if" but "when". A third-party API might become slow, timeout, or fail completely. Without proper protection, these failures can bring down your entire application. In this article, we will explore two powerful patterns that help build resilient systems: Circuit Breakers and Exponential Backoff.

## The Problem: Cascading Failures

Imagine you have a backend service that needs to fetch location coordinates from a third-party mapping API. When this API becomes slow or unresponsive, your application might:

- Keep sending requests to an already failing service
- Waste resources waiting for timeouts
- Create a pile-up of pending requests
- Eventually crash due to resource exhaustion

This is a cascading failure: one component's failure causes the entire system to fail.

## Solution 1: Exponential Backoff

Exponential backoff is a strategy where you wait longer between each retry attempt. Instead of hammering a failing service with requests, you give it time to recover.

### How Exponential Backoff Works

The wait time increases exponentially with each failed attempt:
- First retry: wait 1 second
- Second retry: wait 2 seconds
- Third retry: wait 4 seconds
- Fourth retry: wait 8 seconds

This approach reduces load on the failing service and increases the chance of success.

### Implementing Exponential Backoff in TypeScript

Let's implement a coordinate fetching service with exponential backoff:

```typescript
interface Coordinates {
  latitude: number;
  longitude: number;
}

interface BackoffConfig {
  initialDelayMs: number;
  maxDelayMs: number;
  maxRetries: number;
  multiplier: number;
}

class CoordinateService {
  private readonly config: BackoffConfig = {
    initialDelayMs: 1000,
    maxDelayMs: 32000,
    maxRetries: 5,
    multiplier: 2,
  };

  async fetchCoordinates(address: string): Promise<Coordinates> {
    let lastError: Error;
    let delay = this.config.initialDelayMs;

    for (let attempt = 0; attempt <= this.config.maxRetries; attempt++) {
      try {
        const response = await fetch(
          `https://api.mapping-service.com/geocode?address=${encodeURIComponent(address)}`
        );

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        return {
          latitude: data.lat,
          longitude: data.lng,
        };
      } catch (error) {
        lastError = error as Error;

        if (attempt < this.config.maxRetries) {
          console.log(`Attempt ${attempt + 1} failed. Retrying in ${delay}ms...`);
          await this.sleep(delay);

          // Calculate next delay with exponential backoff
          delay = Math.min(delay * this.config.multiplier, this.config.maxDelayMs);

          // Add jitter to prevent thundering herd
          delay = delay + Math.random() * 1000;
        }
      }
    }

    throw new Error(`Failed to fetch coordinates after ${this.config.maxRetries} attempts: ${lastError!.message}`);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

The code above implements exponential backoff with several important features:

1. **Increasing delays**: Each retry waits longer than the previous one
2. **Maximum delay cap**: Prevents waiting too long between retries
3. **Jitter**: Adds randomness to prevent many clients from retrying at the same time
4. **Maximum retries**: Eventually gives up if the service doesn't recover

## Solution 2: Circuit Breaker

While exponential backoff helps with temporary failures, what happens when a service is down for an extended period? You don't want to keep retrying indefinitely. This is where the Circuit Breaker pattern comes in.

### How Circuit Breakers Work

A circuit breaker has three states:

1. **Closed**: Requests pass through normally. The service is healthy.
2. **Open**: Requests are immediately rejected without calling the service. The service is failing.
3. **Half-Open**: A limited number of test requests are allowed through to check if the service has recovered.

The circuit breaker monitors failures and opens when a threshold is reached. After a timeout period, it moves to half-open to test if the service has recovered.

### Implementing a Circuit Breaker in TypeScript

Let's create a circuit breaker for our coordinate service:

```typescript
enum CircuitState {
  CLOSED = 'CLOSED',
  OPEN = 'OPEN',
  HALF_OPEN = 'HALF_OPEN',
}

interface CircuitBreakerConfig {
  failureThreshold: number;
  successThreshold: number;
  timeout: number;
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount: number = 0;
  private successCount: number = 0;
  private nextAttemptTime: number = Date.now();

  private readonly config: CircuitBreakerConfig = {
    failureThreshold: 5,    // Open circuit after 5 failures
    successThreshold: 2,     // Close circuit after 2 successes in half-open
    timeout: 60000,          // Wait 60 seconds before trying again
  };

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() < this.nextAttemptTime) {
        throw new Error('Circuit breaker is OPEN. Service is unavailable.');
      }
      // Timeout has passed, move to half-open
      this.state = CircuitState.HALF_OPEN;
      console.log('Circuit breaker is now HALF_OPEN. Testing service...');
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      console.log(`Success in HALF_OPEN state (${this.successCount}/${this.config.successThreshold})`);

      if (this.successCount >= this.config.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
        console.log('Circuit breaker is now CLOSED. Service is healthy.');
      }
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.successCount = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.state = CircuitState.OPEN;
      this.nextAttemptTime = Date.now() + this.config.timeout;
      console.log('Circuit breaker is OPEN again. Service still failing.');
      return;
    }

    if (this.failureCount >= this.config.failureThreshold) {
      this.state = CircuitState.OPEN;
      this.nextAttemptTime = Date.now() + this.config.timeout;
      console.log(`Circuit breaker is now OPEN. Too many failures (${this.failureCount}).`);
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

### Combining Circuit Breaker with Exponential Backoff

Now let's combine both patterns to create a highly resilient coordinate service:

```typescript
class ResilientCoordinateService {
  private readonly backoffService: CoordinateService;
  private readonly circuitBreaker: CircuitBreaker;

  constructor() {
    this.backoffService = new CoordinateService();
    this.circuitBreaker = new CircuitBreaker();
  }

  async getCoordinates(address: string): Promise<Coordinates> {
    try {
      return await this.circuitBreaker.execute(() =>
        this.backoffService.fetchCoordinates(address)
      );
    } catch (error) {
      console.error(`Failed to fetch coordinates: ${(error as Error).message}`);

      // Return a fallback response or rethrow
      throw new Error(`Unable to get coordinates for address: ${address}`);
    }
  }

  getCircuitStatus(): CircuitState {
    return this.circuitBreaker.getState();
  }
}
```

### Using the Resilient Service

Here's how you would use this service in your application:

```typescript
const coordinateService = new ResilientCoordinateService();

async function processLocation(address: string) {
  try {
    const coordinates = await coordinateService.getCoordinates(address);
    console.log(`Location: ${address}`);
    console.log(`Coordinates: ${coordinates.latitude}, ${coordinates.longitude}`);
    return coordinates;
  } catch (error) {
    console.error(`Could not process location: ${error}`);
    // Handle the error appropriately
    // Maybe use cached data or a default location
    return null;
  }
}

// Example usage
await processLocation('1600 Amphitheatre Parkway, Mountain View, CA');

// Check circuit breaker status
console.log(`Circuit status: ${coordinateService.getCircuitStatus()}`);
```

## When to Use Each Pattern

### Use Exponential Backoff when:
- You expect temporary failures that resolve quickly
- The service has occasional hiccups but is generally reliable
- You want to automatically retry operations

### Use Circuit Breaker when:
- You want to fail fast instead of waiting for timeouts
- You need to protect your system from a completely down service
- You want to give failing services time to recover

### Use Both when:
- You're building critical systems that must be highly available
- You depend on external services that might fail
- You want the best resilience strategy

## Benefits of These Patterns

1. **Improved Reliability**: Your application continues working even when dependencies fail
2. **Better User Experience**: Faster failure detection means quicker error messages
3. **Resource Protection**: Prevents wasting resources on operations that will fail
4. **Service Recovery**: Gives failing services time to recover without constant requests

## Conclusion

Building resilient systems requires planning for failure. Circuit breakers and exponential backoff are two essential patterns that work together to create robust applications. By implementing these patterns, you protect your system from cascading failures and provide a better experience for your users.

Remember: failures will happen. The question is not how to prevent them, but how to handle them gracefully.

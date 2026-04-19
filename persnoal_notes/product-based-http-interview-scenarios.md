# Product-Based Company Angular Interviews: Core HTTP & Endpoint Scenarios

Product-based companies rarely ask "How do you inject HttpClient?". They ask *how you handle real-world network instability, race conditions, and optimization.*

Here are the top 5 simplest yet most frequently asked HTTP/RxJS scenarios in product-based interviews.

---

## Scenario 1: Parallel API Calls (The Dashboard Problem)
**Question:** "Your dashboard needs to show User Details, Recent Orders, and Notifications. All three APIs are independent. How do you load them efficiently?"

**The Trick:** Do not subscribe to them one by one. That takes `Time(A) + Time(B) + Time(C)`. You need them in parallel using `forkJoin` which takes `Max(Time(A, B, C))`.

```typescript
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { forkJoin, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DashboardService {
  private http = inject(HttpClient);

  getDashboardData(): Observable<any> {
    const user$ = this.http.get('/api/user/1');
    const orders$ = this.http.get('/api/orders?userId=1');
    const notifications$ = this.http.get('/api/notifications');

    // forkJoin waits for ALL observables to complete, then emits an array or object containing all results.
    return forkJoin({
      user: user$,
      orders: orders$,
      notifications: notifications$
    });
  }
}
```

---

## Scenario 2: Sequential API Calls (Dependent Endpoints)
**Question:** "You need to fetch briefly a user's profile. You must extract the `companyId` from that profile, and then immediately fetch the Company Details. How do you do this without nested subscriptions?"

**The Trick:** Never nest `subscribe()` calls (Callback Hell). Use `switchMap` or `mergeMap` to chain observables together cleanly.

```typescript
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { switchMap, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class UserProfileService {
  private http = inject(HttpClient);

  getUserAndCompany(userId: string): Observable<any> {
    // 1. First HTTP Call
    return this.http.get(`/api/users/${userId}`).pipe(
      // 2. Extract companyId and switch to a NEW observable (Second HTTP Call)
      switchMap((user: any) => {
        const companyId = user.companyId;
        return this.http.get(`/api/companies/${companyId}`);
      })
    );
  }
}
```

---

## Scenario 3: Global Error Handling (The Interceptor)
**Question:** "If our backend throws a 401 Unauthorized or a 500 Server Error across 50 different endpoints, how do you handle it in one single place without rewriting code?"

**The Trick:** Write an `HttpInterceptor`. It catches every outgoing request and incoming response.

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { Router } from '@angular/router';

// Modern Functional Interceptor (Angular 15+)
export const globalErrorHandlerInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        // Token expired! Log them out dynamically.
        console.error('Unauthorized! Redirecting to login.');
        router.navigate(['/login']);
      } else if (error.status === 500) {
        console.error('Server is down. Show toast notification.');
      }
      
      // Always rethrow so specific components can still handle their own UI logic if needed
      return throwError(() => error);
    })
  );
};
```
*(Remember to register this interceptor in `provideHttpClient(withInterceptors([globalErrorHandlerInterceptor]))` inside `app.config.ts`)*

---

## Scenario 4: The Transient Network Failure (Retry Mechanism)
**Question:** "The user is on a mobile network, and occasionally the `GET /data` endpoint drops. Before showing a 'Failed' screen, how do you try calling the API 3 times invisibly?"

**The Trick:** Use the `retry` operator.

```typescript
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { retry, catchError, Observable, throwError } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class MobileDataService {
  private http = inject(HttpClient);

  getFlakyData(): Observable<any> {
    return this.http.get('/api/flaky-endpoint').pipe(
      // Wait 1 second, then retry. Do this a maximum of 3 times.
      retry({ count: 3, delay: 1000 }),
      catchError(err => {
        console.error('Failed completely after 3 retries.');
        return throwError(() => err);
      })
    );
  }
}
```

---

## Scenario 5: Prevent Duplicate Network Calls (Data Caching)
**Question:** "We have a list of Countries. 5 different components request this list on initialization via the `CountryService`. Our server is getting identical requests hitting the DB. How do you fetch it once and share it?"

**The Trick:** Use `shareReplay()`. It caches the last emitted value and hands it instantly to any late subscribers without re-firing the HTTP call.

```typescript
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, shareReplay } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ConfigService {
  private http = inject(HttpClient);
  
  // Notice we store the Observable in a variable, not a function returning an observable.
  private countriesCache$: Observable<string[]> | null = null;

  getCountries(): Observable<string[]> {
    if (!this.countriesCache$) {
      this.countriesCache$ = this.http.get<string[]>('/api/countries').pipe(
        // Buffer size 1: Keeps the last 1 emission.
        // refCount false: Keeps the cache alive even if all components unsubscribe.
        shareReplay({ bufferSize: 1, refCount: false })
      );
    }
    // All 5 components get this same, cached stream. Only 1 network request fires.
    return this.countriesCache$;
  }
}
```

These 5 patterns (`forkJoin`, `switchMap`, `HttpInterceptor`, `retry`, and `shareReplay`) are arguably the most heavily tested HTTP scenarios in any senior Angular technical interview.

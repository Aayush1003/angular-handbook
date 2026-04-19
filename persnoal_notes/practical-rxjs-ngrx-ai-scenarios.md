# Practical Mastery: RxJS, NgRx & AI Integration Scenarios in Angular

This document shifts from theory to highly practical, code-heavy implementations. We will cover advanced RxJS endpoint handling and state management using NgRx (SignalStore) specifically tailored for connecting an Angular application to an AI backend (like an LLM chat endpoint).

---

## Scenario 1: The AI Chat Endpoint (Advanced RxJS)
When dealing with an AI endpoint (e.g., generating long text), you face specific challenges: huge latency, users typing rapidly triggering multiple requests (race conditions), and potential stream chunks.

### The Problem: Race Conditions & API Spam
If a user hits "Generate" 3 times fast, we don't want 3 concurrent heavy AI requests slowing down the server and causing UI state collision.

### The Solution: RxJS `switchMap` and `catchError`

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, Subject, of } from 'rxjs';
import { switchMap, catchError, map, startWith, debounceTime } from 'rxjs/operators';

export interface PromptResponse {
  result: string;
  tokensUsed: number;
}

@Injectable({ providedIn: 'root' })
export class OpenAiConnectorService {
  private http = inject(HttpClient);
  private endpoint = '/api/v1/generate-completion';

  // 1. A Subject to listen to incoming prompt triggers
  private promptSubject = new Subject<string>();

  // 2. The derived Observable that handles the actual work
  public promptResponse$: Observable<{ data?: PromptResponse, loading: boolean, error?: string }> = 
    this.promptSubject.pipe(
      // Debounce: Wait 300ms after the user stops spamming the trigger
      debounceTime(300),
      
      // SwitchMap: If a new prompt comes in while an old one is pending, 
      // CANCEL the old HTTP request and only care about the new one.
      switchMap((prompt) => {
        // Emit a 'loading' state immediately before the HTTP call
        return this.http.post<PromptResponse>(this.endpoint, { prompt }).pipe(
          map(response => ({ data: response, loading: false })),
          catchError(error => of({ error: error.message, loading: false })),
          startWith({ loading: true }) // Start loading immediately on subscription
        );
      })
    );

  // Method called by the Component to trigger a generation
  generateText(prompt: string) {
    this.promptSubject.next(prompt);
  }
}
```

**Why this is powerful:** We handle debouncing, request cancellation (`switchMap`), and loading/error states in one cohesive reactive stream. No manual `if (loading) return` checks needed.

---

## Scenario 2: Smart Polling for Long-Running AI Jobs
Sometimes an AI generation job takes 30 seconds. A backend will often return a `jobId` immediately, and you must "poll" the status endpoint until complete.

### The RxJS Polling Solution using `timer` and `takeWhile`
```typescript
import { timer } from 'rxjs';
import { switchMap, takeWhile, tap } from 'rxjs/operators';

pollAiJobStatus(jobId: string): Observable<AiJobStatus> {
   // Emit a value every 2000 milliseconds (2 seconds)
   return timer(0, 2000).pipe(
      // Make the HTTP status check call
      switchMap(() => this.http.get<AiJobStatus>(`/api/v1/jobs/${jobId}`)),
      
      // STOP polling automatically when the status is no longer 'PENDING'
      takeWhile(response => response.status === 'PENDING', true), 
      
      // Side-effect mapping if needed
      tap(res => {
         if (res.status === 'SUCCESS') {
             console.log('AI Generation complete!', res.result);
         }
      })
   );
}
```

---

## Scenario 3: Modern State Management for AI with NgRx SignalStore
NgRx has evolved. While NgRx Global Store (Redux-style with Actions/Reducers) is still used, the **NgRx SignalStore** is the modern, highly pragmatic way to manage state for specific features like an AI Chatbot.

### Building an AI Chat SignalStore
This store acts as a self-contained brain. It manages the messages array, the loading flag, and the asynchronous endpoint calls.

```typescript
// chat.store.ts
import { signalStore, withState, withMethods, patchState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { inject } from '@angular/core';
import { pipe, switchMap, tap } from 'rxjs';
import { OpenAiConnectorService } from './open-ai-connector.service'; // from scenario 1

export interface ChatMessage {
  role: 'user' | 'ai';
  content: string;
}

export interface ChatState {
  messages: ChatMessage[];
  isThinking: boolean;
  error: string | null;
}

const initialState: ChatState = {
  messages: [],
  isThinking: false,
  error: null
};

// Create the Signal Store
export const ChatStore = signalStore(
  { providedIn: 'root' },
  
  // 1. Define State
  withState(initialState),
  
  // 2. Define Methods (Reducers + Effects rolled into one)
  withMethods((store, aiService = inject(OpenAiConnectorService)) => ({
    
    // Synchronous method: Add a user message to the UI immediately
    addUserMessage: (prompt: string) => {
      patchState(store, (state) => ({ 
        messages: [...state.messages, { role: 'user', content: prompt }],
        isThinking: true,
        error: null
      }));
    },

    // Asynchronous rxMethod (replaces NgRx Effects)
    // Connects an RxJS stream directly into store state mutations
    submitPrompt: rxMethod<string>(
      pipe(
        // Tap allows side-effects. Here we call our sync method.
        tap(prompt => patchState(store, { isThinking: true })),
        
        switchMap((prompt) => 
           aiService.generateText(prompt).pipe(
             // On Success: Update state with AI response
             tapResponse({
               next: (apiResponse) => patchState(store, (state) => ({
                 messages: [...state.messages, { role: 'ai', content: apiResponse.result }],
                 isThinking: false
               })),
               error: (err: Error) => patchState(store, { 
                 error: 'AI failed to respond: ' + err.message, 
                 isThinking: false 
               })
             })
           )
        )
      )
    )
  }))
);
```

### Consuming the SignalStore in an Angular 18 Component
Notice how exceptionally clean the component becomes. No subscriptions, no manual unsubscriptions, just pure Signals.

```typescript
// chat.component.ts
import { Component, inject } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { ChatStore } from './chat.store';

@Component({
  selector: 'app-ai-chat',
  standalone: true,
  imports: [FormsModule],
  template: `
    <div class="chat-container">
      
      <!-- Built-in Control Flow (Angular 17+) over Signals -->
      @for (msg of chatStore.messages(); track $index) {
        <div [class]="msg.role">
          <b>{{ msg.role | uppercase }}:</b> {{ msg.content }}
        </div>
      }

      @if (chatStore.isThinking()) {
        <div class="loader">AI is typing...</div>
      }

      @if (chatStore.error()) {
        <div class="error">{{ chatStore.error() }}</div>
      }

    </div>

    <!-- The Input Area -->
    <div class="input-area">
      <input [(ngModel)]="currentInput" (keyup.enter)="send()" placeholder="Ask the AI...">
      <button [disabled]="chatStore.isThinking()" (click)="send()">Send</button>
    </div>
  `
})
export class AiChatComponent {
  // Inject the Signal Store directly
  chatStore = inject(ChatStore);
  currentInput = '';

  send() {
    if (!this.currentInput.trim()) return;
    
    // 1. Immediately add the user's message to UI
    this.chatStore.addUserMessage(this.currentInput);
    
    // 2. Trigger the asynchronous AI API call via the store
    this.chatStore.submitPrompt(this.currentInput);
    
    this.currentInput = ''; // clear input
  }
}
```

## Summary of Practical Realities
1. **RxJS `switchMap` is mandatory** for prompt-based inputs to prevent concurrent backend executions.
2. **Polling (`timer` + `switchMap` + `takeWhile`)** is standard practice for asynchronous AI job queues.
3. **NgRx SignalStore** drastically reduces the boilerplate of classic NgRx. You get the state predictability of a Store, but the elegant consumption and fine-grained reactivity of Signals. Your components remain completely stateless.

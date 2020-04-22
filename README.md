# Run Project

ng serve

```
w project
ng g component login
ng g component dashboard
ng g component nav
ng g service auth
ng g service timeout
ng g service timeoutdialog
ng g interface login
ng g guard auth
```

Let the user know that their session is about to expire and that they would be logged out if they didn’t take action.

Dealing with timers in Angular can be significantly different when using React components and subscriptions. The basic premises remains the same, though. We’ll have a service with a timer that will provide a Subject to which consumers of the service can subscribe. A React Subject provides an easy mechanism to trigger a “next” subscription to alert consumers that the timer has expired. The new service, which I am simply calling TimeoutService, will provide the same methods as the old AngularJS service: startTimer, resetTimer, etc.

### timeout.service.ts

```
import { Injectable } from '@angular/core';
import { Observable, Subject, Subscription, timer } from 'rxjs';
import { map, catchError } from 'rxjs/operators';


@Injectable({
  providedIn: 'root'
})
export class TimeoutService {
  private _count = 0;
  private _serviceId: string = 'idleTimeoutSvc-' + Math.floor(Math.random() * 10000);
  private _timeoutMilliseconds = 15000;
  private timerSubscription: Subscription;
  private timer: Observable<number>;
  private _timer: Observable<number>;
  private resetOnTrigger: Boolean = false;
  private lastTime: number;
  private dateTimer: Observable<number>;
  private dateTimerSubscription: Subscription;
  private dateTimerInterval: number = 1000 * 60 * 5;
  private dateTimerTolerance: number = 1000 * 10;
  public timeoutExpired: Subject<number> = new Subject<number>();


  constructor() {
      console.log('Constructed idleTimeoutService ' + this._serviceId);


      this.timeoutExpired.subscribe(n => {
          console.log('timeoutExpired subject next.. ' + n.toString());
      });


      this.startTimer();
      this.startDateCompare();
  }


  private setSubscription() {
      this._timer = timer(this._timeoutMilliseconds);
      this.timerSubscription = this._timer.subscribe(n => {
          this.timerComplete(n);
      });
  }


  private startDateCompare() {
      this.lastTime = (new Date()).getTime();
      this.dateTimer = timer(this.dateTimerInterval); // compare every five minutes
      this.dateTimerSubscription = this.dateTimer.subscribe(n => {
          const currentTime: number = (new Date()).getTime();
          if (currentTime > (this.lastTime + this.dateTimerInterval + this.dateTimerTolerance)) { // look for 10 sec diff
              console.log('Looks like the machine just woke up.. ');
          }  else {
              console.log('Machine did not sleep.. ');
          }
          this.dateTimerSubscription.unsubscribe();
          this.startDateCompare();
      });
  }


  public startTimer() {
      if (this.timerSubscription) {
          this.stopTimer();
      }


      this.setSubscription();
  }


  public stopTimer() {
      this.timerSubscription.unsubscribe();
  }


  public resetTimer() {
      this.startTimer();
  }


  private timerComplete(n: number) {
      this.timeoutExpired.next(++this._count);


      if (this.resetOnTrigger) {
          this.startTimer();
      }
  }
}
```


The timer variable is our actual Observable that we use internally. You can see below that we use React’s timer observable. We’ll have it trigger in five seconds and subscribe to it. Once it is triggered, we call our handler. The “startTimer” method sets up our subscriptions.

Our timer handler will call the “next” method on our Subject to which others may be subscribed. This provides a nice eventing mechanism. Optionally, we can reset the timer after trigger. However, for most of my use cases, I’ll be resetting this manually.

To consume the service, the TimerService is injected as needed. Then, we can subscribe to its exposed/public Subject.

The basic subscription is show in the code snippet.

```
ngOnInit() {
    this.id = localStorage.getItem('token');
    this.sessionTimeout();
  }


  sessionTimeout() {
    this.startCounter();
    this.idleTimeoutSvc.resetTimer();
    this.changeRef.markForCheck();
    this._idleTimerSubscription = this.idleTimeoutSvc.timeoutExpired.subscribe(res => {
      const modalPromise = this.dialogSvc.open('Session Expiring!',
      'Your session is about to expire. Do you need more time?', true, 'Yes', 'No');
      const newObservable = from(modalPromise);
      newObservable.subscribe(
          (resp) => {
              if (resp === true) {
                  console.log('Extending session...');
                  this.idleTimeoutSvc.resetTimer();
                  this.startCounter();
                  this.changeRef.markForCheck();
              } else {
                  console.log('Not extending session...');
                  this.logout();
                  this.startCounter();
                  this.idleTimeoutSvc.resetTimer();
                  this.changeRef.markForCheck();
              }
          },
          (reason) => {
              console.log('Dismissed ' + reason);
              this._status = 'Session was not extended.';
              this.logout();
              this.changeRef.markForCheck();
          }
      );
    });
  }
```  
  
Using an RxJS timer, to illustrate that the subject does emit to subscribers every five teen seconds. I then use a timeout dialog service to display a modal. The modal itself returns a promise result. This simulates the concept of extending a user’s session if they click “Yes” or not extending it if they click “No.”

While I didn’t illustrate it, this becomes the starting point for hooking into Http interceptors to access a heartbeat API, or some other endpoint, to extend the user’s session at the server level (or cookie refresh). All in all, timers and observables are pretty fun to play around with in Angular.

### app.module.ts

import AuthGuard and router module in app.module.ts. configure routes using constappRoutes and import appRoutes using RouterModule.forRoot(appRoutes). forRoot creates a module that contains all the directives, the given routes, and the router service itself. canActivate : [AuthGuard] is useful when we want to check something before a component gets used : checking if an user is authenticated. That's why i have declared in dashboard route section. After checking user , we will give permission to enter dashboard.

```
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';
import { FormsModule, ReactiveFormsModule } from '@angular/forms';
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';


import { AppComponent } from './app.component';
import { LoginComponent } from './login/login.component';
import { DashboardComponent } from './dashboard/dashboard.component';


import { TimeoutService } from './timeout.service';
import { JwtInterceptor } from './jwt.interceptor';
import { TimeoutdialogService, TimeoutdialogComponent } from './timeoutdialog.service';
import { NgbModule } from '@ng-bootstrap/ng-bootstrap';
import { NavComponent } from './nav/nav.component';



const appRoutes: Routes = [
  { path: 'login', component: LoginComponent },
  { path: 'dashboard', component: DashboardComponent, canActivate: [AuthGuard] },
  // otherwise redirect to home
  { path: '**', redirectTo: 'login' }
];


@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
    DashboardComponent,
    TimeoutdialogComponent,
    NavComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    ReactiveFormsModule,
    HttpClientModule,
    NgbModule.forRoot(),
    RouterModule.forRoot(appRoutes)
  ],
  providers: [
    AuthGuard,
    AuthService,
    TimeoutService,
    TimeoutdialogService,
    // {provide: HTTP_INTERCEPTORS, useClass: JwtInterceptor, multi: true}
  ],
  entryComponents: [TimeoutdialogComponent],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### timeoutdialog.service.ts

```
import { Injectable, Component, Input, OnInit, ApplicationRef, ChangeDetectorRef } from '@angular/core';
import { Observable, of } from 'rxjs';
import { delay, tap } from 'rxjs/operators';
import { NgbModal, NgbModalOptions, NgbActiveModal, ModalDismissReasons } from '@ng-bootstrap/ng-bootstrap';


@Component({
  template: `
  <div class="modal-header">
    <h4 class="modal-title">{{ title }}</h4>
    <button type="button" class="close" aria-label="Close" (click)="activeModal.dismiss('Cross click')">
      <span aria-hidden="true">&times;</span>
    </button>
  </div>
  <div class="modal-body">
    <p [innerHTML]="message"></p>
  </div>
  <div class="modal-footer">
    <button *ngIf="showCancel" type="button" class="btn btn-secondary" (click)="activeModal.close(false)">{{ cancelText }}</button>
    <button type="button" class="btn btn-secondary" (click)="activeModal.close(true)">{{ confirmText }}</button>
  </div>
`
})


export class TimeoutdialogComponent implements OnInit {
  @Input() title;
  @Input() message;
  @Input() showCancel = false;
  @Input() confirmText: String = 'Ok';
  @Input() cancelText: String = 'Cancel';


  constructor(public activeModal: NgbActiveModal, public changeRef: ChangeDetectorRef) {
      // console.log("DialogComponent construct");
  }


  ngOnInit() {
      // console.log("DialogComponent init");
  }
}


@Injectable({
  providedIn: 'root'
})
export class TimeoutdialogService {


  constructor(private modalService: NgbModal) { }


    public confirm() {
        const modalRef = this.modalService.open(TimeoutdialogComponent);


        const instance = (modalRef as any)._windowCmptRef.instance;
        instance.windowClass = '';


        // setImmediate(() => {
        //    instance.windowClass = 'custom-show'
        // })


        setTimeout(() => {
            instance.windowClass = 'custom-show';
        }, 0);


        const fx = (modalRef as any)._removeModalElements.bind(modalRef);
        (modalRef as any)._removeModalElements = () => {
            instance.windowClass = '';
            setTimeout(fx, 250);
        };


        modalRef.componentInstance.title = 'Discard Changes?';
        modalRef.componentInstance.message = 'Are you sure you want to discard your changes?';
        modalRef.componentInstance.changeRef.markForCheck();
        return modalRef.result;
    }


    public open(title: string, message: string, showCancel: boolean = false, confirmText: string = 'Ok', cancelText: string = 'Cancel',
        options: NgbModalOptions = { size: 'sm' }) {
        const modalRef = this.modalService.open(TimeoutdialogComponent, options);


        const instance = (modalRef as any)._windowCmptRef.instance;


        // setImmediate(() => {
        //    instance.windowClass = "custom-show";
        // })


        setTimeout(() => {
            instance.windowClass = 'custom-show';
        }, 0);


        const fx = (modalRef as any)._removeModalElements.bind(modalRef);
        (modalRef as any)._removeModalElements = () => {
            instance.windowClass = '';
            setTimeout(fx, 250);
        };


        modalRef.componentInstance.title = title;
        modalRef.componentInstance.message = message;
        modalRef.componentInstance.showCancel = showCancel;
        modalRef.componentInstance.confirmText = confirmText;
        modalRef.componentInstance.cancelText = cancelText;
        modalRef.componentInstance.changeRef.markForCheck();
        return modalRef.result;
    }
}
```

this is used for dynamically create modal after 15 secs session over.

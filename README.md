# ionic3-angular6-interceptor

$ ionic start interceptor blank
$ cd interceptor

$ ionic g provider interceptor

$ ionic cordova plugin add cordova-sqlite-storage

$ npm install --save @ionic/storage

app/app.module.ts to:
---------------------
import { BrowserModule } from '@angular/platform-browser';
import { ErrorHandler, NgModule } from '@angular/core';
import { IonicApp, IonicErrorHandler, IonicModule } from 'ionic-angular';
import { SplashScreen } from '@ionic-native/splash-screen';
import { StatusBar } from '@ionic-native/status-bar';
 
import { MyApp } from './app.component';
import { HomePage } from '../pages/home/home';
import { InterceptorProvider } from '../providers/interceptor/interceptor';
 
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
import { IonicStorageModule } from '@ionic/storage';
 
@NgModule({
  declarations: [
    MyApp,
    HomePage
  ],
  imports: [
    BrowserModule,
    IonicModule.forRoot(MyApp),
    IonicStorageModule.forRoot(),
    HttpClientModule
  ],
  bootstrap: [IonicApp],
  entryComponents: [
    MyApp,
    HomePage
  ],
  providers: [
    StatusBar,
    SplashScreen,
    { provide: ErrorHandler, useClass: IonicErrorHandler },
    { provide: HTTP_INTERCEPTORS, useClass: InterceptorProvider, multi: true },
  ]
})
export class AppModule {}



providers/interceptor/interceptor.ts and change it to:
-----------------------------------------------------
import { AlertController } from 'ionic-angular';
import { HttpEvent, HttpHandler, HttpInterceptor, HttpRequest } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Storage } from '@ionic/storage';
 
import { Observable } from 'rxjs';
import { _throw } from 'rxjs/observable/throw';
import { catchError, mergeMap } from 'rxjs/operators';
 
@Injectable()
export class InterceptorProvider implements HttpInterceptor {
 
    constructor(private storage: Storage, private alertCtrl: AlertController) { }
 
    // Intercepts all HTTP requests!
    intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
 
        let promise = this.storage.get('my_token');
 
        return Observable.fromPromise(promise)
            .mergeMap(token => {
                let clonedReq = this.addToken(request, token);
                return next.handle(clonedReq).pipe(
                    catchError(error => {
                        // Perhaps display an error for specific status codes here already?
                        let msg = error.message;
 
                        let alert = this.alertCtrl.create({
                            title: error.name,
                            message: msg,
                            buttons: ['OK']
                        });
                        alert.present();
 
                        // Pass the error to the caller of the function
                        return _throw(error);
                    })
                );
            });
    }
 
    // Adds the token to your headers if it exists
    private addToken(request: HttpRequest<any>, token: any) {
        if (token) {
            let clone: HttpRequest<any>;
            clone = request.clone({
                setHeaders: {
                    Accept: `application/json`,
                    'Content-Type': `application/json`,
                    Authorization: `Bearer ${token}`
                }
            });
            return clone;
        }
 
        return request;
    }
}



pages/home/home.ts and change it to:
-----------------------------------------
import { HttpClient } from '@angular/common/http';
import { Storage } from '@ionic/storage';
import { Component } from '@angular/core';
 
@Component({
  selector: 'page-home',
  templateUrl: 'home.html'
})
export class HomePage {
 
  authenticated = false;
  message = '';
 
  constructor(private http: HttpClient, private storage: Storage) { }
 
  setAuthState(authenticated) {
    if(authenticated) {
      this.storage.set('my_token', 'myspecialheadertoken').then(() => {
        this.authenticated = true;
      });
    } else {
      this.storage.remove('my_token').then(() => {
        this.authenticated = false;
      });
    }
  }
 
  getSuccessful() {
    this.http.get('https://pokeapi.co/api/v2/pokemon/').subscribe(res => {
      this.message = res['results'][0].name;
    });
  }
 
  getFail() {
    this.http.get('https://notvalid.xy').subscribe(
      res => {}
      ,err => {
        this.message = err.message;
      }
    );
  }
}



pages/home/home.html:
---------------------
<ion-header>
  <ion-navbar color="primary">
    <ion-title>
      Ionic HTTP Intercept
    </ion-title>
  </ion-navbar>
</ion-header>
 
<ion-content padding>
 
  <ion-card text-center>
    <ion-card-header>
      My Auth State: {{ authenticated }}
    </ion-card-header>
    <ion-card-content>
      {{ message }}
    </ion-card-content>
  </ion-card>
 
  <button ion-button full (click)="setAuthState(true)" *ngIf="!authenticated">Login</button>
  <button ion-button full color="danger" (click)="setAuthState(false)" *ngIf="authenticated">Logout</button>
  <button ion-button full color="secondary" (click)="getSuccessful()">Get URL successful</button>
  <button ion-button full color="light" (click)="getFail()">Get URL with error</button>
 
</ion-content>

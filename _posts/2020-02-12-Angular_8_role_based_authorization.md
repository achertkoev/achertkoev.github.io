---
layout: post
title: Angular 8 - Role-based authorization with sample
tags: Angular8 TypeScript
---

In this post I'd like to show you an example of how you can implement role based authorization / access control front end using Angular 8.

The code is available on GitHub at [https://github.com/FSou1/angular-8-role-based-authorization-sample](https://github.com/FSou1/angular-8-role-based-authorization-sample).

Demo in action:

![angular-8-rba-authorization](/images/post/angular 8 - rba authorization.gif)

Here is the result:

<iframe width="740" height="530" src="https://stackblitz.com/edit/angular-8-role-based-authorization-sample?embed=1&file=src/app/services/auth.service.ts" frameborder="0" class="center-image"></iframe>

(See on StackBlitz at [https://stackblitz.com/edit/angular-8-role-based-authorization-sample](https://stackblitz.com/edit/angular-8-role-based-authorization-sample))

## How to run the app locally

1. Install NodeJS and NPM from [https://nodejs.org](https://nodejs.org).
2. Download or clone the tutorial project source code from [https://github.com/FSou1/angular-8-role-based-authorization-sample](https://github.com/FSou1/angular-8-role-based-authorization-sample)
3. Install all required npm packages by running `npm install` from the command line in the project root folder (where the package.json is located).
4. Start the application by running `npm start` from the command line in the project root folder, this will build the application and automatically launch it in the browser on the URL http://localhost:4200.

NOTE: You can also run the app directly using the Angular CLI command ng serve --open. To do this first install the Angular CLI globally on your system with the command npm install -g @angular/cli.

> Thanks to Jason Watmore for the guide above

## Project structure

<a name="projectstructure"></a>

Bellow are the main project files that contain the application logic:

* src
   * app
      * admin
         * dashboard
            * dashboard.component.html
            * dashboard.component.ts
         * admin-routing.module.ts
         * admin.module.ts
      * app
         * app.component.html
         * app.component.ts
      * directives
         * user-role.directive.ts
         * user.directive.ts
      * error
         * not-found
            * not-found.component.html
            * not-found.component.ts
      * home
         * home.component.html
         * home.component.ts
      * login
         * login.component.html
         * login.component.ts
      * models
         * role.ts
         * user.ts
      * profile
         * profile.component.html
         * profile.component.ts
      * services
         * auth.service.ts
      * app-routing.guard.ts
      * app-routing.module.ts
      * app.module.ts
   * index.html
* package.json
* tsconfig.json

### Admin routing module

The module contains admin routes and mapped components. The module itself is [lazy loaded](https://angular.io/guide/lazy-loading-ngmodules) and managed as a part of the <a href="#app-routing-module">AppRoutingModule</a> (by passing the <a href="#auth-guard">AuthGuard</a> to the `canActivate` and the `canLoad` properties).

```
import { Routes } from '@angular/router';
import { DashboardComponent } from './dashboard/dashboard.component';

export const routes: Routes = [
    { path: 'dashboard', component: DashboardComponent }
];
```
<a href="#projectstructure">Back to top</a>

### Admin module

The admin module is the root module for admin workspace and declares the list of available components and routes.

```
import { NgModule } from '@angular/core';

import { DashboardComponent } from './dashboard/dashboard.component';
import { RouterModule } from '@angular/router';
import { routes } from './admin-routing.module';

@NgModule({
  declarations: [
    DashboardComponent
  ],
  imports: [
    RouterModule.forChild(routes)
  ],
  providers: []
})
export class AdminModule { }
```
<a href="#projectstructure">Back to top</a>

### App component template

The app component template is the root component template of the application. It contains the main nav bar which changes based on the user. 

The profile and the logout links are only visible for authorized users (by using the `isAuthorized` getter). The dashboard link is only displayed for admins (by using the `isAdmin` getter). The login link is only accesible by anonymous users.

The router-outlet directive acts as a placeholder that the app (Angular) dynamically fills based on the current url (router state). [Read more](https://angular.io/guide/router).

```
<header class="navbar navbar-expand navbar-dark bg-dark">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item">
        <a class="nav-link" [routerLink]="['/']">Home</a>
      </li>

      <!-- Authorized only -->
      <li class="nav-item" *ngIf="isAuthorized">
        <a class="nav-link" [routerLink]="['profile']">Profile</a>
      </li>

      <!-- Admin only -->
      <li class="nav-item" *ngIf="isAdmin">
        <a class="nav-link" [routerLink]="['admin', 'dashboard']">Dashboard</a>
      </li>

      <!-- Anonymous only -->
      <li class="nav-item" *ngIf="!isAuthorized">
        <a class="nav-link" [routerLink]="['login']">Login</a>
      </li>

      <!-- Authorized only -->
      <li class="nav-item" *ngIf="isAuthorized">
        <a class="nav-link" (click)="logout()">Logout</a>
      </li>
    </ul>

    <span class="navbar-text">
      <span class="badge badge-warning" *ngIf="isAuthorized">Authorized</span>
      <span class="badge badge-danger" *ngIf="isAdmin">Admin</span>
    </span>
</header>

<div class="jumbotron">
  <div class="container-fluid">
    <router-outlet></router-outlet>
  </div>
</div>

<div class="text-center">
  <p>
    <a href="https://twitter.com/maximzhukov_dev" target="_blank">@maximzhukov_dev</a>, 2020
  </p>

  <p>
    <a href="https://fsou1.github.io/Angular_8_role_based_authorization/" target="_top">Angular 8 - Role-based authorization with sample</a>
  </p>
</div>
```
<a href="#projectstructure">Back to top</a>

### App component

The app component is the root component of the application. It contains the `isAuthorized` and the `isAdmin` getters which are called inside the template. The `logout` method is used to sign out the currently authorized user and redirect him to the login page.

```
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { Role } from '../models/role';
import { AuthService } from '../services/auth.service';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit {
  Role = Role;

  constructor(private router: Router, private authService: AuthService) { }

  ngOnInit() {
  }

  get isAuthorized() {
    return this.authService.isAuthorized();
  }

  get isAdmin() {
    return this.authService.hasRole(Role.Admin);
  }

  logout() {
    this.authService.logout();
    this.router.navigate(['login']);
  }
}
```
<a href="#projectstructure">Back to top</a>

<a name="user-role-directive"></a>
### User role directive

The user role directive is a structural directive that conditionally includes a template if:
* A user is authorized
* A user has at least one of the allowed roles

Allowed roles are passed by using the `appUserRole` setter, for example: `*appUserRole="[ Role.Admin ]"`.

A good thing is that the directive encapsulates the authorization logic and can be reused within the application.

On the other hand a `hasAccess` value is evaluated within the `ngOnInit` method, so a template won't be redrawn if the value changes. That's why I wouldn't use the directive in the layout (like the navbar). At the same time it's still useful on the <a href="#profile-template">profile component template</a>.

```
import { Directive, OnInit, TemplateRef, ViewContainerRef, Input } from '@angular/core';
import { AuthService } from '../services/auth.service';
import { Role } from '../models/role';

@Directive({ selector: '[appUserRole]'})
export class UserRoleDirective implements OnInit {
    constructor(
        private templateRef: TemplateRef<any>,
        private authService: AuthService,
        private viewContainer: ViewContainerRef
    ) { }

    userRoles: Role[];

    @Input() 
    set appUserRole(roles: Role[]) {
        if (!roles || !roles.length) {
            throw new Error('Roles value is empty or missed');
        }

        this.userRoles = roles;
    }

    ngOnInit() {
        let hasAccess = false;

        if (this.authService.isAuthorized() && this.userRoles) {
            hasAccess = this.userRoles.some(r => this.authService.hasRole(r));
        }

        if (hasAccess) {
            this.viewContainer.createEmbeddedView(this.templateRef);
        } else {
            this.viewContainer.clear();
        }
    }
}
```
<a href="#projectstructure">Back to top</a>

### User directive

The user directive is similar to the <a href="#user-role-directive">UserRoleDirective</a>. It includes a template for authorized users only. You can see the `*appUser` directive in action at the <a href="#profile-template">profile component template</a>.

```
import { Directive, OnInit, TemplateRef, ViewContainerRef, Input } from '@angular/core';
import { AuthService } from '../services/auth.service';

@Directive({ selector: '[appUser]'})
export class UserDirective implements OnInit {
    constructor(
        private templateRef: TemplateRef<any>,
        private authService: AuthService,
        private viewContainer: ViewContainerRef
    ) { }

    ngOnInit() {
        const hasAccess = this.authService.isAuthorized();

        if (hasAccess) {
            this.viewContainer.createEmbeddedView(this.templateRef);
        } else {
            this.viewContainer.clear();
        }
    }
}
```
<a href="#projectstructure">Back to top</a>

### Login component template

```
<div class="alert alert-success" role="alert">
  Login component!
</div>

<button 
  class="btn btn-outline-warning" 
  (click)="login(Role.User)">Login as User</button>

<button 
  class="btn btn-outline-danger" 
  (click)="login(Role.Admin)">Login as Admin</button>
```
<a href="#projectstructure">Back to top</a>

### Login component

```
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { Role } from '../models/role';
import { AuthService } from '../services/auth.service';

@Component({
  selector: 'app-login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})
export class LoginComponent implements OnInit {
  Role = Role;

  constructor(private router: Router, private authService: AuthService) { }

  ngOnInit() {
  }

  login(role: Role) {
    this.authService.login(role);
    this.router.navigate(['/']);
  }

  logout() {
    this.authService.logout();
  }
}
```
<a href="#projectstructure">Back to top</a>

### Role model

```
export enum Role {
  User = 1,
  Admin = 2
}
```
<a href="#projectstructure">Back to top</a>

### User model

```
import { Role } from './role';

export class User {
  Role: Role;
}
```
<a href="#projectstructure">Back to top</a>

<a name="profile-template"></a>
### Profile component template

```
<div class="alert alert-success" role="alert">
  Profile component!
</div>

<div class="alert alert-warning" role="alert" *appUser>
  Content for authorized users only (e.g. display email, send message)!
</div>

<div class="alert alert-danger" role="alert" *appUserRole="[ Role.Admin ]">
  Content for admin users only (e.g. enable/disable user)!
</div>
```
<a href="#projectstructure">Back to top</a>

### Profile component

```
import { Component, OnInit } from '@angular/core';
import { Role } from '../models/role';

@Component({
  selector: 'app-profile',
  templateUrl: './profile.component.html',
  styleUrls: ['./profile.component.css']
})
export class ProfileComponent implements OnInit {
  Role = Role;

  constructor() { }

  ngOnInit() {
  }
}
```
<a href="#projectstructure">Back to top</a>

### Auth service

```
import { Injectable } from '@angular/core';
import { User } from '../models/user';
import { Role } from '../models/role';

@Injectable()
export class AuthService {
    private user: User;

    isAuthorized() {
        return !!this.user;
    }

    hasRole(role: Role) {
        return this.isAuthorized() && this.user.Role === role;
    }

    login(role: Role) {
      this.user = { Role: role };
    }

    logout() {
      this.user = null;
    }
}
```
<a href="#projectstructure">Back to top</a>

<a name="auth-guard"></a>
### Auth guard

```
import { CanActivate, Router, ActivatedRouteSnapshot, CanLoad, Route } from '@angular/router';
import { Observable } from 'rxjs';
import { Injectable } from '@angular/core';
import { AuthService } from './services/auth.service';
import { Role } from './models/role';

@Injectable()
export class AuthGuard implements CanActivate, CanLoad {
    constructor(
        private router: Router,
        private authService: AuthService
    ) { }

    canActivate(route: ActivatedRouteSnapshot): Observable<boolean> | Promise<boolean> | boolean {
        if (!this.authService.isAuthorized()) {
            this.router.navigate(['login']);
            return false;
        }

        const roles = route.data.roles as Role[];
        if (roles && !roles.some(r => this.authService.hasRole(r))) {
            this.router.navigate(['error', 'not-found']);
            return false;
        }

        return true;
    }

    canLoad(route: Route): Observable<boolean> | Promise<boolean> | boolean {
        if (!this.authService.isAuthorized()) {
            return false;
        }

        const roles = route.data && route.data.roles as Role[];
        if (roles && !roles.some(r => this.authService.hasRole(r))) {
            return false;
        }

        return true;
    }
}
```
<a href="#projectstructure">Back to top</a>

<a name="app-routing-module"></a>
### App routing module

```
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { ProfileComponent } from './profile/profile.component';
import { NotFoundComponent } from './error/not-found/not-found.component';
import { AuthGuard } from './app-routing.guard';
import { AuthService } from './services/auth.service';
import { LoginComponent } from './login/login.component';
import { Role } from './models/role';

const routes: Routes = [
  {
    path: '',
    children: [
      {
        path: '',
        component: HomeComponent
      },

      {
        path: 'profile',
        canActivate: [AuthGuard],
        component: ProfileComponent
      },

      {
        path: 'login',
        component: LoginComponent
      }
    ]
  },
  {
    path: 'admin',
    canLoad: [AuthGuard],
    canActivate: [AuthGuard],
    data: {
      roles: [
        Role.Admin,
      ]
    },
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  {
    path: '**',
    component: NotFoundComponent
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
  providers: [
    AuthGuard,
    AuthService
  ]
})
export class AppRoutingModule { }
```
<a href="#projectstructure">Back to top</a>

### App module

```
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app/app.component';
import { HomeComponent } from './home/home.component';
import { ProfileComponent } from './profile/profile.component';
import { NotFoundComponent } from './error/not-found/not-found.component';
import { LoginComponent } from './login/login.component';
import { UserRoleDirective } from './directives/user-role.directive';
import { UserDirective } from './directives/user.directive';
import { AuthService } from './services/auth.service';

@NgModule({
  declarations: [
    AppComponent,
    HomeComponent,
    ProfileComponent,
    NotFoundComponent,
    LoginComponent,
    UserDirective,
    UserRoleDirective
  ],
  imports: [
    BrowserModule,
    AppRoutingModule
  ],
  exports: [
    UserDirective,
    UserRoleDirective
  ],
  providers: [AuthService],
  bootstrap: [AppComponent]
})
export class AppModule { }
```
<a href="#projectstructure">Back to top</a>

Reference:

* [Stackblitz - sample](https://stackblitz.com/edit/angular-8-role-based-authorization-sample)
* [GitHub - source code](https://github.com/FSou1/angular-8-role-based-authorization-sample)

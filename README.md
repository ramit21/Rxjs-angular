# Rxjs-angular
Reactive Angular using Rxjs

## Project setup
Use Node v18.12.0

Use NVM to switch node versions.
```
nvm list
nvm install 18.12.0
nvm use 18.12.0
```

Starting the project:

```
npm install
cd server  
npm run server (backend server)

To start UI:
npm start  (on root directory)
http://localhost:4200
```

See proxy.json, and url call of type /api/ gets routed to our backend server running in localhost on port 9000.

## Concepts

### Async processing - traditional vs reactive way

What problem does Reactive programming solve?

1. Traditional way of async programming using promises etc can lead to callback hell. Reactive programming helps code away from callbacks.
2. Also, if the value fetched from say an API call is required elsewhere, then you need to store the result in memory, and share across components. 
This can make the code hard to maintain, whereas with Reactive, all the observers of an Observable will get the data whenever it emits a value.
Also, if you are storing and mutating component variables like this, you cannot use optimised change detection mode of 'OnPush'.

A component written in traditional way looks like this:

```
export class HomeComponent implements OnInit {

  courses: Course[];

  ngOnInit() {
      this.http.get('/api/courses')
	      .subscribe(res => {
		      this.courses = res["payload"];
  }
  
And then loop over the list of courses in the HTML using ngFor loop.

```

This code refactored in reactive way looks like this:

```
export class HomeComponent implements OnInit {

  courses$: Observable<Course[]>;

  ngOnInit() {
      this.courses$ = this.http.get<Course[]>('/api/courses')
	      .pipe(map(res => res["payload"]));
	}
  }
  

```
Things to note above:
1. the $ in the name indiciates that is an Observable that will emit values, and to get the values, you need to subscribe to it.
2. We specify the type of Observable being returned in http.get() call.
3. We are not storing any mutable data in memory. courses$ is just an Observable, not a direct reference to Courses array.
4. In the HTML if you don't have a Courses array to loop through. What you need in the html, is a subscription to courses observable.
For this, use **async** pipe which creates your subscription to the courses$ Observable:

```
<div *ngFor="let course of courses$ | async">
  <div> {{course.description}} </div>
``` 
Mostly http call code is kept in a service, and since using above way, we are not storing any state in the service, we follow 
a pattern known as **'Stateless Observable-based services'**.

If you do not make any async subscription, no HTTP call will be made, as Angular Observables are lazy by nature. 

### What is a Subject?
An RxJS Subject is a special type of Observable that allows values to be multicasted to many Observers.
While plain Observables are unicast (each subscribed Observer owns an independent execution of the Observable), subjects are multicast.

Every Subject is an Observable - an observer can subscribe to it using the suscribe() method. 

Every Subject is an observer - It is an object with the methods next(v), error(e), and complete(). To feed a new value to the Subject, 
just call next(theValue), and it will be multicasted to the Observers registered to listen to the Subject.

More on subjects, and different types of Subjects:
https://rxjs.dev/guide/subject


### Rxjs operators

**pipe()**: Take an Observable as input, and return another Observable based on the input Observable.

**map()**: convert one observable into another type of observalbe emitting different values as per the transformation specified. Mostly used within pipe().

**tap()**: a utility operator used for a sife-effect, like logging emmitted value to a console, or capture the complete/error events. Mostly used within pipe().
https://rxjs.dev/api/operators/tap

**of()**: It's a creational operator, used to create an Observable from a source.
eg. of(1, 2, 3, 4, 5).subscribe(val => ...); 

of(null) -> creates an observable which emits only 1 null value. Can be used with tap operator to trigger some operation, and concatMap to switch to another observable
once operation specified in tap has been performed (see pt. 4 below)

EMPTY vs of(null): https://lookout.dev/rules/difference-between-empty-and-of-null-from-rxjs

**shareReplay**: You generally want to use shareReplay when you have side-effects or taxing computations that you do not wish to be executed amongst
 multiple subscribers. For eg. an HTTP call - if multiple subscribers are listening the an observable being returned from the http call, shareReplay will ensure that
 only 1 HTTP call is made, and all observers get the same emitted value. Otherwise, http call will get executed as many times as the no. of subscribers.
 
**finalize()**: Execute a callback function when the observable completes.
 
**concatMap()**: Switches from emitting values of one observable to another one, but only when first observable completes.

**mergeMap()**: Switches from emitting values of one observable to another one immediately. 

**combineLatest()**: Takes an array of observables, and returns a new observable which emits a value everytime one of the input observable emits a value.

More on concatMap, mergeMap, exhaustMap, and switchMap: https://blog.angular-university.io/rxjs-higher-order-mapping/
 
### Things to note in this POC code base

1. Smart vs Presentational (dumb) component: See Home component HTML: https://github.com/ramit21/Rxjs-angular/blob/main/src/app/home/home.component.html
It is a smart component, as it is dealing with data. But delegates the display presentation responsibility to courses-card-list component, which does not have 
any information about the source data.

2. Notice how we filter to check if save call went fine, and further use tap operator to emit an event to notify about successfull save.
https://github.com/ramit21/Rxjs-angular/blob/f54355bf325f3e9690798cda3304c5c3412ab590/src/app/courses-card-list/courses-card-list.component.ts#L44

3. Sharing of data between components: If the data is to be shared between parent child components, you can use EventEmitters. But if the componentsare not parent
child, then the other option is hold the data in memory through a shared service. You can further decide on the scope of a service. For eg, if you want to hold an 
information which which stay singleton across the application, like a logged in user, you create the service at root level. This way the service is created only 
once at the root level. See this: https://github.com/ramit21/Rxjs-angular/blob/f54355bf325f3e9690798cda3304c5c3412ab590/src/app/services/auth.store.ts#L10

But if you want to hold a state which can vary across components, or in other words a service which is not a singleton at root level, instead different instances
are required at different component level, (eg. a loader service which stores boolean flag to show/hide the loader, and different components want 
to reuse it on their respective operations) you define the service like this https://github.com/ramit21/Rxjs-angular/blob/f54355bf325f3e9690798cda3304c5c3412ab590/src/app/loading/loading.service.ts#L6

Notice the use of this service in loading component, and further, how this component when specified in different htmls of other components, creates a new instance of this
service for that component: https://github.com/ramit21/Rxjs-angular/blob/f54355bf325f3e9690798cda3304c5c3412ab590/src/app/app.component.html#L56

4. Look at a very concise way of implementing a loader implementation: https://github.com/ramit21/Rxjs-angular/blob/f54355bf325f3e9690798cda3304c5c3412ab590/src/app/loading/loading.service.ts#L17
What we are trying to do here is that using tap operator, trigger the loader to be on, and then using concatMap operator, switch to the observable passed in the argument 
(ie. switching from a null observable created using of(), to the passed in observable),
so that when the passed in observable completes and stops emitting values, we can then turn the loader off using the finalize operator.

5. Error handling: https://github.com/ramit21/Rxjs-angular/blob/00c59ac4da0f7dd437b87cc690d38cd068c917ae/src/app/services/courses.store.ts#L33

thowError operator returns a new observable than the original observable, ie  switches from original observable which has errored out to a new observable,
and this new observable which completes immediately, so that the observable emit chain gets broken.

6. In this POC, we are saving course updates in an optimistic way, ie. assuming that save will not error out. That is why we emit the subject first for other parts of ui to reflect the save, and then make the backend call. Other approach is of course to emit values only on succesfull save.

https://github.com/ramit21/Rxjs-angular/blob/29560c2a8a5721e37242a9d496ee56aeebe03ed1/src/app/services/courses.store.ts#L62

7. Look at Authstore how logged-in/out observables are maintained in the constructor, and then in login/logout methods. Also look at how we then display either the login button, or the logout.

https://github.com/ramit21/Rxjs-angular/blob/29560c2a8a5721e37242a9d496ee56aeebe03ed1/src/app/services/auth.store.ts#L23

https://github.com/ramit21/Rxjs-angular/blob/29560c2a8a5721e37242a9d496ee56aeebe03ed1/src/app/app.component.html#L25

We store logged in user information in browser's localstorage to survive browser refreshes.

https://github.com/ramit21/Rxjs-angular/blob/29560c2a8a5721e37242a9d496ee56aeebe03ed1/src/app/services/auth.store.ts#L40

8. **Single data Observable Pattern:** If you have multiple subscriptions to different observables on an HTML template, having them one inside another can lead to performance issues, as outer one may take time to load, and once it loads, the inner one takes its own time sequentially as well.

```
<ng-container *ng-if="(course$|async) as course">
   <ng-container *ng-if="(lessons$|async) as lessons">
    .....
```
Also note that 2 ng-ifs cannot be clubbed in a signle ng-container, hence we do need 2 ng-containers here.

The solution to above performance issue is to use 'Single data Observable Pattern', and use a single observable (achieved using interface) emit a value when any of the input observables emits a value (using combineLatest() observable)

https://github.com/ramit21/Rxjs-angular/blob/29560c2a8a5721e37242a9d496ee56aeebe03ed1/src/app/course/course.component.ts#L58

And it's corresponding subscription (for both data.courses and data.lessons) in the html:

https://github.com/ramit21/Rxjs-angular/blob/29560c2a8a5721e37242a9d496ee56aeebe03ed1/src/app/course/course.component.html#L7

9. **OnPush Change Detection:** Optimised angular change detection mode, not on by default. By default, Angular checks all expressions across all components to decide weather re rendering is required or not. This is not very efficient. OnPush change detection, if turned on, says that if any of the observables being subscribed to (in the html) have emmitted a new value, then mark the field dirty and trigger re-rendering for the dirty fields only.

This is done for all components in this POC, check this for an example:

https://github.com/ramit21/Rxjs-angular/blob/c2d51b18ee6a8af73668a698a7365ebfa7a15952/src/app/home/home.component.ts#L11




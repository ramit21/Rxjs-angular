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

of(null) -> used to ????

EMPTY vs of(null): https://lookout.dev/rules/difference-between-empty-and-of-null-from-rxjs

**shareReplay**: You generally want to use shareReplay when you have side-effects or taxing computations that you do not wish to be executed amongst
 multiple subscribers. For eg. an HTTP call - if multiple subscribers are listening the an observable being returned from the http call, shareReplay will ensure that
 only 1 HTTP call is made, and all observers get the same emitted value. Otherwise, http call will get executed as many times as the no. of subscribers.
 
 
### Things to note in this POC code base

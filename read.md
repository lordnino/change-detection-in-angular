# Angular Change Detection

This is an article which i'm trying to explain what is change detection in angular.

## What's change detection anyway?

The basic task of change detection is to take the internal state of a program and make it somehow visible to the user interface.  
This state can be any kind of objects, array , primitive, ... just any kind of javascript data structures.  

This state might end up as paragraphs, forms, links or buttons in the user interface and specifially on the web, it's the Document 
Object Model (DOM).  

So basically we take data structure as inputs and generate DOM output to display it to the user. We call this process **process rendering**

However, it gets trickier when a change happens at runtime. Some time later when the DOM has already been rendered. How do we figure out what 
has changed in out model, and where do we need to update the DOM? Accessing the DOM tree is always expensive , so not not only we need to find out 
where updates are needed, but we also want to keep that access as tiny as possible.

This can be tackled in many different ways. One way, for instance, is simply making a http request and re-rendering the whole page.  
Another approach is the concept of diffing the DOM of the new state with the previous state and only render the difference, which is what ReactJS is doing 
with the **Virtual DOM.**

So basically the goal of change detection is always projecting data and it's change.

## What causes the change ?

Now that we know what change detection is all about, we might wonder, when excatly can such a channge happen? When does Angular know that it has to update the View? 
Well, let's take a look at the following code

```javascript
    @Component({
        template: `
            <h1>{{firstname}} {{lastname}}</h1>
            <button (click)="changeName()">Change name</button>
        `
    })
    class MyApp {

        firstname:string = 'Pascal';
        lastname:string = 'Precht';

        changeName() {
            this.firstname = 'Brad';
            this.lastname = 'Green';
        }
    }
```

The component above simply displays two properties and provides a method to change them and when the button in the template is clicked.  
The moment this particular button is clicked is the moment when applicatin state has changed, because it changes the properties of the component.  
That's the moment we want to update the view.

Here's another one:

```javascript
    @Component()
    class ContactsApp implements OnInit{

        contacts:Contact[] = [];

        constructor(private http: Http) {}

        ngOnInit() {
            this.http.get('/contacts')
            .map(res => res.json())
            .subscribe(contacts => this.contacts = contacts);
        }
    }

```

This component holds a list of contacts and when it initializes, it preforms a http request. Once the request comes back, the list get updated. 
Again, at the point, our application state has changed so we will want to update the view.

Basically application state change can be caused by three things:

* **Events** - click, submit, ...
* **XHR** - Fetching data from a remote server
* **Timers** - setTimeout, setInterval()

They are all asynchronous, which brings us to the conclusion that, basically whenever some asynchronous operation has been performed, our application 
state might have changed. This is when someone needs to tell angular to update the view.

## Who notifies Angular ?

Alright, we know what causes application state change. But what is it that tells Angular, that as this particular moment, the view has to be updated ?

Angular allows us to use native APIs directly. There are no intereceptor methods we have to call so Angular gets notified to update the DOM. is that 
pure magic?

Zones take care of this. In fact, Angular comes with its own zone called **NgZone**.

The short version is, that somewhere in Angular's source code, there's this thing called **ApplicationRef**, which listens to **NgZones** **onTurnDone** 
event. Whenever this event is fired, it executes a **tick()** function which essentially performs change detection

```javascript
    // very simplified version of actual source
    class ApplicationRef {

        changeDetectorRefs:ChangeDetectorRef[] = [];

        constructor(private zone: NgZone) {
            this.zone.onTurnDone
            .subscribe(() => this.zone.run(() => this.tick());
        }

        tick() {
            this.changeDetectorRefs
            .forEach((ref) => ref.detectChanges());
        }
    }
```

## Change Detection

Okay cool, we now know when change detection is triggered, but how is it performed? Well, the first thing we need to notice is that, 
in Angular, **each component has its own change detector.**

This is a signifact fact, since this allows us to control, for each component individually, how and when change detection is performed! More on that later.

Let's assume that somewhere in out component tree an event is fired, maybe a button has been clicked. What happens next? We just learned that zones execute 
the given handler and notify Angular when the turn is done, which eventually causes Angular to perform change detection.

Since, each component has it's own change detector, and an Angular application consists of a component tree, the logical result is that we're having a **change detector** too. this tree can also be viewed as a directed graph where data always flows from top to bottom.

The reason why data flows from top to bottom, is because change detection is also always performed from top to bottom for every single component, every single time, 
starting from the root component. This is awesome, as unidirectional data flow is more predictable than cycles. We always know where the data in our views comes from, 
because it can only result from its component.

Another interesting observation is that change detection gets stable after single pass. Meaning that, if one of our components causes any additional side effects after the 
first run during change detection, Angular will throw an error.


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

## Performance

BY default, even if we have to check every single component every single time an event happens, Angular is very fast. It can perform hundreds of thousands of checks within a couple of milliseconds. This ia mainly due to fact that **Angular generates VM friendly code.**

What does that mean? well, when we said that each component has its own change detector, it's not like there's this single generic thing in Angular that takes care of change detection for each individual component.

Thre reason for that is, that it has to be written in a dynamic way, so it can check every component no matter what its model structure looks like. VMs don't like this sort oqf dynamic code, because they can't optimize it. It's considered **polymorphic** 
as the shape of the objects is't always the same.

**Angular creates change detector classes at runtime** for each component, which are monomorphic, because they know exactly what the shape of the component's model is. VMs can perfectly optimize this code, which makes it very fast to execute. The good thing is that we don't have to care about that too much, because Angulat does it automatically.

## Smart Change Detection

Again, Angular has to check every component every single time an event happens because... well, maybe the application state has changed. But wouldn't it be great if we could tell Angular to **only** run change detection for the parts of the application that changed their state?

Yes it would, and in fact we can! It turns out there are data structure that gives is some guarantees of when something has changed ot not - **Immutables** and **Observables**. If we happen to use these structure of types, and we tell Angular about it, change detection can be much faster. Okay cool, but how so?

# Understanding Mutability

In order to understand why and how e.g. immutable data structures can help, we need to understand what mutability means.  
Assume we have the following component:

```javascript
    @Component({
        template: '<v-card [vData]="vData"></v-card>'
    })
    class VCardApp {

        constructor() {
            this.vData = {
            name: 'Christoph Burgdorf',
            email: 'christoph@thoughtram.io'
            }
        }

        changeData() {
            this.vData.name = 'Pascal Precht';
        }
    }
```

**VCardApp** uses **<v-card>** as a child component, which has an input property **vData**. We're passing data to that component with **VCardsApp**'s own **vData** property. **vData** is an object with two properties. In addition, there's method **changeData()**, whicj changes the name of **vData**. No magic going on here.

The important part is that **changeData()** mutates **vData** by changing it's **name** property. Even though that property is going to be changed, the **vData** reference itseld stays the same.

What happens when change detection is performed, assuming that some event causes **changeData()** to be executed? First, **vData.name** gets changed, and then it's passed to **<v-card>**. **<v-card>**'s change detector now checks if the given **vData** is still the same as before, and yes, it is. The reference hasn't changed. However, the **name** property has changed, so Angular will perform change detection for that object nonetheless.

Because objects are mutable by the default in javascript (except for primitive), Angular has to be conservative and run change detection every single time for every component when an event happens.

This is where immutable data structures comes into play.

## Immutable Objects

Immutable objects give us the guarantee that objects can't change. Meaning that, if we use immutable objects and e want to make a change on such an object, we'll always get a new reference with that change, as the original object is immutable.

This pseudo code demonstrates it:

```javascript
    var vData = someAPIForImmutables.create({
                name: 'Pascal Precht'
                });

    var vData2 = vData.set('name', 'Christoph Burgdorf');

    vData === vData2 // false
```

**someAPIForImmutables** can be any API we want to use for immutable data structures. However, as we can see, we can't simply change the **name** property. We'll get a new object with that particular change and this object has a new reference. Or in short: **If there's a change, we get a new reference**.

## Reducing the number of checks

Angular can skip entire change detection subtrees when input properties don't change. We just learned that a "change" means "new reference". If we use immutable objects in our Angular app, all we need to do is tell Angular that a component can skip change detection, if its input hasn't changed.

Let's see that works by taking a look at **<v-card>>**: 

```javascript
    @Component({
        template: `
            <h2>{{vData.name}}</h2>
            <span>{{vData.email}}</span>
        `
    })
    class VCardCmp {
        @Input() vData;
    }
```

As we can see, **VCardCmp** only depends on its input properties. Great. We can tell Angular to skip change detection for this component's subtree if none of its inputs changed by setting the change detection strategy to **OnPush** like this:

```javascript
    @Component({
        template: `
            <h2>{{vData.name}}</h2>
            <span>{{vData.email}}</span>
        `,
        changeDetection: ChangeDetectionStrategy.OnPush
    })
    class VCardCmp {
        @Input() vData;
    }
```

That's it! Now imagine a bigger component tree. We can skip entire subtrees when immutable objects are used and Angular is informed accordingly.

## Observables

As mentioned earlier, Observables also gives us some certain guarantees of when a change has happened. Unlike immutable objects, they don't give us new reference when a change is made. Instead, they fire events we can subscribe to in order to react to them.

So, if we use Obseravbles and we want to use **OnPush** to skip change detector subtrees, but the reference of these objects will never change, how do we deal with that? It turns out Angular has **a very smart** way to enable paths in the component tree to be checked for certain events, which is excatly what we need in that case.

To understand what that means, let's take a look at this component:

```javascript
@Component({
  template: '{{counter}}',
  changeDetection: ChangeDetectionStrategy.OnPush
})
class CartBadgeCmp {

  @Input() addItemStream:Observable<any>;
  counter = 0;

  ngOnInit() {
    this.addItemStream.subscribe(() => {
      this.counter++; // application state changed
    })
  }
}

```
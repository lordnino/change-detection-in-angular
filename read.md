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

```Angular
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
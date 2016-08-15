---
title: Data models for modern frontend applications
date: 2016-08-15 17:11:51
tags: 
- JavaScript
- TypeScript
categories:
- Development
---

There is a bunch of popular frameworks, which gives you a way to create and manipulate data model. 
For example, Backbone, Ember, and some others.
But modern JavaScript gives you the opportunity to create simple yet powerful models in minutes.

In first part of this article we will see how to do it. While the second part will show us the power 
of TypeScript in the same task. You will also learn how to create property decorators.

Let's start!  

## Part 1. JavaScript Way

So, let's create a model class named `Person`, which has 2 public properties: `name` and `surname`.

{% codeblock Person.js lang:javascript %}
var Person = function(name, surname) {
    this.name = name;
    this.surname = surname;
};
{% endcodeblock %}

Pretty simple, yeah? 

Let's add a computed property `fullname`. We will use [Object.defineProperty()](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty). 
This is a cool cross-browser feature, which has been implemented even in IE 9. 
It gives you the opportunity to set accessors without creating `.get()` and `.set()` methods.

{% codeblock Person.js continuation lang:javascript %}
Object.defineProperty(Person.prototype, 'fullname', {
    get: function() { 
        return this.name + ' ' + this.surname; 
    },
    enumerable: true,
    configurable: false
});

var john = new Person('John', 'Snow');

console.log(john.fullname); // => 'John Snow'
{% endcodeblock %}

Pay your attention to a fact that you should create property in Person.prototype instead of 
instance, because this property is actually a getter, so it is a function.

Look how this getter works. It calculates `fullname` every time you try to get it. Let's create some caching to avoid redundant computations. 
This feature requires dependencies (`name` and `surname`) to change cached value.
I suggest to create special `property()` and `computed()` functions to bind computeds and dependencies together.

{% codeblock property.js lang:javascript %}
function property(target, propName) {
    target.prototype._depsMap = target.prototype._depsMap || {};
    target.prototype._getters = target.prototype._getters || {};

    Object.defineProperty(target.prototype, propName, {
        set: function(value) { 
            this._cached = this._cached || {};

            if (value !== this._cached[propName]) {
                this._cached[propName] = value;

                var deps = this._depsMap[propName];
                
                for (var i = 0; i < deps.length; i++) {
                    this._cached[deps[i]] = this._getters[deps[i]].call(this);
                }      
            }
        },
        get: function() {
            this._cached = this._cached || {};

            return this._cached[propName]; 
        },
        enumerable: true,
        configurable: true
    });
}
{% endcodeblock %}

{% codeblock computed.js lang:javascript %}
function computed(target, propName, depsArr, getter) {
    target.prototype._depsMap = target.prototype._depsMap || {};
    target.prototype._getters = target.prototype._getters || {};

    target.prototype._getters[propName] = getter;

    for (var i = 0; i < depsArr.length; i++) {
        var dep = depsArr[i];

        target.prototype._depsMap[dep] = target.prototype._depsMap[dep] || [];

        if (target.prototype._depsMap[dep].indexOf(propName) === -1) {
            target.prototype._depsMap[dep].push(propName);
        }
    } 

    Object.defineProperty(target.prototype, propName, {
        get: function() { 
            this._cached = this._cached || {};

            return this._cached[propName];
        },
        enumerable: true,
        configurable: true
    });
}
{% endcodeblock %}

In these functions we created service prototype properties: 
- `_depsMap` to store information about relations between computeds and dependencies,
- `_getters` to store original getters of computeds.

One should mention that both of `property()` and `computed()` are actually property decorators. 

Now `Person` model should be described like this: 

{% codeblock Person.js lang:javascript %}
var Person = function(name, surname) {
    this.name = name;
    this.surname = surname;
};

property(Person, 'name');
property(Person, 'surname');
computed(Person, 'fullname', ['name', 'surname'], function () {
    return this.name + ' ' + this.surname;
});
{% endcodeblock %}

Hooray! Now we have simple but awesome model without any frameworks and libraries.

## Part 2. TypeScript Way

OK. Now Let's do the same thing using cool TypeScript features.

Simple `Person` class will look like this:

{% codeblock Person.ts lang:typescript %}
class Person {
    public name: string;
    public surname: string;
    public get fullname():string {
        return `${this.name} ${this.surname}`; 
    }

    constructor(name: string, surname: string) {
        this.name = name;
        this.surname = surname;
    }
}
{% endcodeblock %}

I have mentioned that `property()` and `computed()` from part 1 are decorators. 
Typescript has very convenient style of creating and using decorators. 
You can get more info in [this article](http://blog.wolksoftware.com/decorators-reflection-javascript-typescript).

I should say that decorators are still an experimental TS feature. 
The compiler must be run with `--experimentalDecorators` flag.

So these are new functions:

{% codeblock property.ts lang:typescript %}
function property(target: any, propName: string) {
    target._depsMap = target._depsMap || {};
    target._getters = target._getters || {};
    
    Object.defineProperty(target, propName, {
        set: function(value) { 
            this._cached = this._cached || {};
            
            if (value !== this._cached[propName]) {
                this._cached[propName] = value;

                let deps = this._depsMap[propName];
                
                for (let dep of deps) {
                    this._cached[dep] = this._getters[dep].call(this);
                }
            }
        },
        get: function() {
            this._cached = this._cached || {};
            
            return this._cached[propName]; 
        },
        enumerable: true,
        configurable: true
    });
}
{% endcodeblock %}

{% codeblock computed.ts lang:typescript %}
function computed(...deps: Array<string>) {
    return function (target, propName, descriptor: TypedPropertyDescriptor<any>) {
        target._depsMap = target._depsMap || {};
        target._getters = target._getters || {};
        
        target._getters[propName] = descriptor.get;
        
        for (let dep of deps) {
            target._depsMap[dep] = target._depsMap[dep] || [];

            if (target._depsMap[dep].indexOf(propName) === -1) {    
                target._depsMap[dep].push(propName);
            }
        }
        
        descriptor.get = function() {
            this._cached = this._cached || {};
            
            return this._cached[propName];
        }
        
        return descriptor;
    }
}
{% endcodeblock %}

Actually, these decorators are the same stuff as in JavaScript part.

And here is an updated `Person` class. Look at a very convenient '@-blabla' notation. 
This is how decorators are applied in TypeScript.

{% codeblock Person.ts lang:typescript %}
class Person { 
    @property
    public name: string;
    
    @property
    public surname: string;
    
    @computed('name', 'surname')
    public get fullname(): string {
        return `${this.name} ${this.surname}`; 
    }

    constructor(name: string, surname: string) {
        this.name = name;
        this.surname = surname;
    }
}
{% endcodeblock %} 

## Conclusion

In this post I've described a method to design data models without any frameworks. 

You can add lots of other features to your models using these approach. 
For example, you can add `observable()` decorator paired with `.onchange()` and `.off()` methods 
to implement event emitter pattern.   
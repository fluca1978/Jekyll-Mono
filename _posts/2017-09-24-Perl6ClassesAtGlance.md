---
layout: post
title:  "Perl 6: beginning with classes"
author: Luca Ferrari
tags:
- raku
- programming
permalink: /:year/:month/:day/:title.html
---
A very short introduction to class declaration and definition in Perl 6. Let's see twigils, inheritance, methods, constructors and containers.

## Perl6: beginning with classes
-----

Perl 6 does support classes within the base language, and most of the syntax has been inherited (no pun intended!) from *Moose* and alike.
One thing to keep in mind when working with classes is that almost every attribute is defined via a **twigil**, that is a double sigil:
- the first part of the sigil means the type (so the classical *$, @, %, &*);
- the second part of the sigil defines the visibility, where a ```.``` (dot) means **accessible** and a ```!``` means **private**;
- accessing a member does not mean that such member is public, rather that the compiler has defined an **accessor method**;
- *setter* accessor are not created by default, but user has to specify them explicitly.

So, assuming your class has the generic member ```a```, this is the matrix *rule-of-thumb* for including the member in a class:

| first part | second part  | twigil | meaning | accessible outside of the instance ? |
| $ | . | **$.a**  | a scalar with a *getter* method named a() | read-only |
| $ | ! | **$!a** | a scalar without any *getter* or setter |    no       |
| @ | . | **@.a** | an array with a getter method named a() | read only |
| @ | ! | **@!a** | an array without any getter or setter | no |
| % | . | **%.a** | an hash with a getter named a() | read only |
| % | ! | **%!a** | an hash without any getter or setter | no |
| & | . | **&.a** | a callable method named a() | read only |
| & | ! | **&!a** | a private method named a() | no |

Here's a simple example:

```perl
class Person {
    has $.name    ;
    has $.surname ;
    has $.age     ;
}
````

All the members will have a public **getter** named as the property itself, and therefore the object can be used as follows:

```perl
my $me = Person.new( name => 'Luca', surname => 'Ferrari', age => 39 );
say $me.surname ~ " " ~ $me.name ~ " aged " ~ $me.age;
```

but trying to assign values to the properties will result in several compilation errors:

```perl
$me.surname = 'FERRARI'; # Cannot modify an immutable Str
$me.surname( 'FERRARI' ); # Too many positionals passed; expected 1 argument but got 2
```

Therefore, what if we want to be able to mutate the members? They have to be declared has **rw** in the class definition, and this will make
the compiler to generate and accessor method named as the member, but this time modifiable:

```perl
class Person {
    has $.name    is rw;
    has $.surname is rw;
    has $.age     is rw;
}
```
tha can be now consumed as follows:

```perl
my $me = Person.new( name => 'Luca', surname => 'Ferrari', age => 39 );
$me.surname = 'FERRARI';
```

Please note that each property is like an *lvalue method*, so the following is still valid:

```perl
my $me = Person.new( name => 'Luca', surname => 'Ferrari', age => 39 );
$me.surname() = 'FERRARI';
```

while the following is wrong:

```perl
my $me = Person.new( name => 'Luca', surname => 'Ferrari', age => 39 );
$me.surname( 'FERRARI' );
```

The truth is that the member is not accessed via an *lvalue method*, more on this later, in the meantime assume it is like this.

What if our members have been declared private? No accessor will be generated in this case:

```perl
class Person {
    has $.name    is rw;
    has $.surname is rw;
    has $!age; # private!
}
```

and in this case trying to access the ```age()``` method will result in a *no such method* exception.
Please note that the ```age``` attribute **will not be set by the constructor**: a private attribute is not modifiable even at construction time.
This is of course not what we want, and a solution it to place an initialization into the construction chain, that is is
define a ```BUILD``` method (the core behind ```new```) to handle all the params. The class therefore become:

```perl
class Person {
    has $.name    is rw;
    has $.surname is rw;
    has $!age;

    submethod BUILD( :$!name, :$!surname, :$!age ){ }


    method increase_age {
        $!age++;
    }

    method say {
        return $!surname ~ " " ~ $!name ~ " aged " ~ $!age;
    }
}
```

Note that the build method does nothing, since the names of the attributes are passed directly by their name.
If you look carefully, you'll see that parameters are passed using always a ```$!``` twigil, and that is another rule of thumb:
the **!** part of a twigil does not exactly means *private*, but rather **accessible only within the class**. Since the object has not been
fully initialized when ```BUILD``` is executing, all accessor are not in place, so they cannot be accessed via ```$.``` but rather only
within the class itself.
The same consideration applies to the ```say``` method, that in fact accesses all members via the special **$!** within class twigil.

The difference between ```BUILD``` and ```say``` is that the former is a so called **submethod**, that is a special method that is not
inherited by subclasses. This makes them great for object construction and destruction and for all those special tasks that are striclty related
to the type the class is defining. Thanks to the fact that submethods are not going to be substitued by subclasses, they will always be in place
for a specific task (and note that submethods are public!).
For instance the default ```Mu::new``` calls each ```BUILD``` on the inheritance chain, allowing each type on the chain to initialize itself in the proper way without the risk that overriding a build method will lead to inconsistencies even in future (child) classes.

Now, the class can be consumed as follows:

```perl
my $me = Person.new( name => 'Luca', surname => 'Ferrari', age => 39 );
$me.increase_age;
say $me.say;
```


In order to make clearer the concept about twigils, let's see how we could write the ```say``` method:

```perl
 method say {
     return $!surname ~ " " ~ $!name ~ " aged " ~ $!age;
 }

 # or ...
 method say {
     return $.surname ~ " " ~ $.name ~ " aged " ~ $!age;
 }
```

which will convert to the following Java code:

```java
 public String say(){
   // return $!surname ~ " " ~ $!name ~ " aged " ~ $!age;
   return this.surname + " " + this.name + " aged " + this.age;
 }

 // or ...
  public String say(){
    // return $.surname ~ " " ~ $.name ~ " aged " ~ $!age;
   return this.getSurname() + " " + this.getName() + " aged " + this.age;
 }
```

Therefore, when working within the class itself, the **$.** twigil means **use the accessor to get the member value** while the **$!** means **access the member directly**. Of course, in the case of ```age``` there is no accessor at all, so the only way to access it is using the **$!** twigil (within the class).
Outside the class, the only **accessor available are always named as the member and are by default read-only (i.e., getter), but can be converted to lvalue method (i.e., methods you can assign a value) if the special *is rw* trait has been applied**.


With respect to the ```BUILD``` submethod you can also add some consistency check, for instance preventing to build a person with a negative age:

```perl
 submethod BUILD( :$!name, :$!surname, :$!age ){
     $!age = 0 if ( $!age < 0 );
 }
```

Let's continue with methods: as you can imagine you can always add methods to a class, and methods can call other methods to perform thei task.
Within a class, the special bareword **self** allows for calling a method, so for instance you can introduce the method ```happy_birthday```
as follows:

```perl
class Person {
    has $.name    is rw;
    has $.surname is rw;
    has $!age;

    # ... other methods as above

    method increase_age {
        $!age++;
    }

    method happy_birthday {
        "Happy Birthday $!name !".say;
        self.increase_age;
        "you are now $!age years old !".say;
    }
}

```

As you can see, the ```self.increase_age``` resolvs to invoking the other method, something similar when in other languages you
do ```this.increase_age()```. Note also that the twigils are interpolated within strings.


### Inheritance

Allow me to quickly expand the example code with a little inheritance, just to show how to use it in Perl 6. Let's define a *baby* class,
that is the child and extends a person, in the sense it is a "small person". A baby has all the features of a person and was born
from a mother and a father, so something as simple as:

```perl
class Baby is Person {
    has Person $.mother is required;
    has Person $.father is required;
}

```

This is a good definition, since both *mother* and *father* are read-only and must be specified at build time (unluckily this is not fully true in real life, being the mother the only really required, but let's keep the example simple!).
Now you can consume your class using objects as follows:

```perl
my $father = Person.new( name => 'John', surname => 'Richardson', age => 30 );
my $mother = Person.new( name => 'Sandra', surname => 'Loreston', age => 30 );
my $baby   = Baby.new( name => 'Luca', surname => 'Richardson', age => 2,
                         mother => $mother,
                         father => $father );

say $baby.say;
say "Mother: " ~ $baby.mother.say;
say "Father: " ~ $baby.father.say;

```

Note that the baby inherits all methods from the parent person class, that is what we expected.


### Containers

In the above I wrote that accessing a writable member was like using an *lvalue method*, that is not true in Perl 6.
In fact, in Perl 6 each variable (within a class or not) holds a *pointer* to a container, and the container in turn
holds the value of the variable.
**When you call an accessor to a member, it always returns the *container* of the member itself**. So when you define the
member as ```is rw``` what you are truly stating is that the **container must be writable**, not the member itself.
And this leads to how things work:

```perl
$f.name = 'Luca';
```

means that the method ```name``` returns the ```$!name``` container, that is writable, and therefore the assignment puts a new value
into the container. The next time you will call the ```name``` method, the container will be accessed again and the new value will
be presented.

It is something like the following C code pseudo-code:

```c
// ...
char* name(){ return &name; }
// ...
*( f.name() ) = 'LUCA';
```

This concept (containers) is true for all varialbles in Perl 6, not only applies to classes and members.
For more information, [read here](https://docs.perl6.org/language/containers#What_is_a_variable?).

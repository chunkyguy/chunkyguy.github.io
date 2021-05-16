# Objective-C for Swift devs

## Introduction

First thing first, Objective-C isn't like any other language so it requires one to clear their minds of any existing paradigms. Another thing to keep in mind is that unlike Swift Objective-C does not look as pretty in slideshows or condensed articles such as this. To really feel the real power of Objective-C one truly needs to work on full blown project to feel the depth of the language. With that let us dive deeper into the basics of the languages.

## Class and Methods

Every type in Objective-C is a class and every instance is a reference.

## Variables

## Categories and Extensions

## Strings

## Array

## Dictionary

## Optionals

## Error Handling

## C functions

## C structs

## Naming conventions

There's a popular saying which goes along the lines that "code is written once and read many times", and Objective-C adheres to that so much that there's a whole system of conventions that is expected from Objective-C code. Apple has a very well written [document on the naming conventions](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html), I'll summarize them here, but it's highly recommended to read that document.

**1. Prefix for namespace**

Since Objective-C does not provide any notion of namespacing. To avoid type name conflicts popular libraries and frameworks come with their own 2 or 3 character prefix. The 2 character prefixes are reserved for Apple frameworks and it is recommended that others should use atleast 3 characters for prefix.

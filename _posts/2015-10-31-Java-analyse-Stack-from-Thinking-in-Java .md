---
author: "AereXu"
layout: post
category: Java
title: Java analyse Stack from Thinking in Java 
tagline: by Snail
tags: [Java, stack]
modified: 2015-10-31
---

A little analysis of stack from a example from "Thinking in Java" 

<!--more-->

# Introduction

I'm reading "Thinking in Java (Forth Edition)" about Generics and find this example by a glance. But then I get interest of how exactly the stack works.  

If any of my analysis is wrong or imprecise, please tell me.

# Code

Actually the book I read is Chinese edition at page 357 of chapter 15. I guess it will not be considered as a infringement to the publisher if telling the reference.

    public class LinkedStack<T> {
        private static class Node<U>{
            U item;
            Node<U> next;
            Node(){item = null; next = null;}
            Node(U item, Node<U> next){
                this.item = item;
                this.next = next;
            }
            boolean end(){ return item == null && next == null;}
        }
        private Node<T> top = new Node<>();
        public void push(T item){
            top = new Node<>(item, top);
        }
        public T pop(){
            T result = top.item;
            if(!top.end()){
                top = top.next;
            }
            return result;
        }
        public static void main(String[] args){
            LinkedStack<String> lss = new LinkedStack<>();
            for(String s : "Phasers on stun!".split(" "))
                lss.push(s);
            String s;
            while ((s=lss.pop())!=null)
                System.out.println(s);
        }
    }
    // The result is 
    // stun!
    // on
    // Phasers

# Analysis

Obviously, __U__ is exactly the __T__, and a __String__ in this example. That's because of `private Node<T> top = new Node<>();` and `top = new Node<>(item, top);`. Here's all Generics knowledge in here.

## Model
Let me use a diagram to show a instance of Node<T> and LinkedStack<T>.  
![](http://i.imgur.com/UyBIdxx.png)  

## New stack
After you new the lss, you get a totally empty stack. The __Pn__ point to the Node create by `new Node<T>()` 
![](http://i.imgur.com/fwbsoNu.png)  

## Push
Then let's call `void push(T item)` twice, __a__ is the first and __b__ is the second. Notice the __pa__ and __pb__ is generated after calling `new Node<T>(item,top)`.
![](http://i.imgur.com/DB8AZMP.png)  

## Pop
And let's call `T pop()` twice, to see how __b__ and __a__ walks out. Notice the __lss__ is using __next__ (__pb__ in first `pop()`, __pa__ in second `pop()`) to get the real __Node__.
![](http://i.imgur.com/ceYTRhm.png)  


    
    
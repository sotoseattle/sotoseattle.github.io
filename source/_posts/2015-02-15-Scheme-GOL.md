---
layout: post
title: "Game of Life in Scheme"
date: 2015-02-15 10:00
comments: true
categories: Scheme
---

{% img right /images/feb15/gol_scheme.gif 250 %}

I am going through the [Seasoned Schemer](https://www.goodreads.com/book/show/475858.The_Seasoned_Schemer), the second part of the legendary Socratic Scheme books.

As an exercise, and coinciding with an upcoming workshop organized by Seattle Ruby, I have coded Conway's Game of Life in Scheme. It is not perfect and only uses the syntax and concepts learned in the Scheme textbooks up to the use of letrecc (Chapter 12 I think).

The complete code can be found in my repo at the Seattle Study Group ([Code](https://github.com/SeaRbSg/little-schemer/tree/master/sotoseattle/sandbox/game_of_life)).

The animated gif is a visualization of the code written.

<!--more-->
<br/>

```scheme
#lang racket/base

; Cell Utility Functions ------------------------------------------------------

(define (x-coord cell) (car cell))
(define (y-coord cell) (car (cdr cell)))

(define include?  ; does a board include a specific cell?
  (lambda (cell list)
    (letrec
      ((==? (lambda (c2)
          (and (eq? (x-coord cell) (x-coord c2))
               (eq? (y-coord cell) (y-coord c2)))))
       (in? (lambda (l)
         (cond
           [(null? l) #f]
           [else (or (==? (car l)) (in? (cdr l)))]))))
      (in? list))))

(define uniq  ; remove duplicate cells in a list
  (lambda (lat)
    (cond
      [(null? lat) lat]
      [(include? (car lat) (cdr lat)) (uniq (cdr lat))]
      [else (cons (car lat) (uniq (cdr lat)))])))

(define adj  ; 8 adjacent neighboring cells around a cell
  (lambda (cell)
    (define x (x-coord cell))
    (define y (y-coord cell))
    (cons (cons (+ x 1) (cons (- y 1) '()))
    (cons (cons (+ x 1) (cons y '()))
    (cons (cons (+ x 1) (cons (+ y 1) '()))
    (cons (cons x (cons (- y 1) '()))
    (cons (cons x (cons (+ y 1) '()))
    (cons (cons (- x 1) (cons (- y 1) '()))
    (cons (cons (- x 1) (cons y '()))
    (cons (cons (- x 1) (cons (+ y 1) '())) '()))))))))))

(define n-adj-alive  ; count the number of living neighbors around a cell
  (lambda (cell living)
    (letrec
      ((rec (lambda (n cells)
          (cond
            [(null? cells) n]
            [(include? (car cells) living) (rec (+ n 1) (cdr cells))]
            [else (rec n (cdr cells))]))))
      (rec 0 (adj cell)))))

; Rule 1: when a cell will remains alive after a tick of the clock ------------

(define stay-alive
  (letrec
    ((rec (lambda (board living new-board)
      (letrec
        ((survivors
          (letrec
            ((alive? (lambda (cell)
              (cond
                [(eq? 2 (n-adj-alive cell living)) #t]
                [(eq? 3 (n-adj-alive cell living)) #t]
                [else #f]))))
            (lambda (old new)
              (cond
                [(null? old) new]
                [(alive? (car old)) (survivors (cdr old) (cons (car old) new))]
                [else (survivors (cdr old) new)])))))
        (survivors board new-board)))))
    (lambda (board)
      (rec board board '()))))

; Rule 2: when a dead cell becomes alive after a tick of the clock ------------

(define become-alive
  (lambda (living)
    (letrec
      ((perimeter (lambda (alive potentials)
        (letrec
          ((cell-cons
            (lambda (l1 l2)
              (cond
                [(null? l1) l2]
                [(include? (car l1) living) (cell-cons (cdr l1) l2)]
                [else (cell-cons (cdr l1) (cons (car l1) l2))]))))
          (cond
            [(null? alive) potentials]
            [else (perimeter (cdr alive) (cell-cons (adj (car alive)) potentials))]))))
       (germinate
         (letrec
           ((rec (lambda (potentials)
             (letrec
               ((raise? (lambda (cell)
                 (cond
                   [(eq? 3 (n-adj-alive cell living)) #t]
                   [else #f]))))
               (cond
                 [(null? potentials) potentials]
                 [(raise? (car potentials)) (cons (car potentials) (germinate (cdr potentials)))]
                 [else (germinate (cdr potentials))])))))
           (lambda (maybe)
             (rec maybe)))))
    (germinate (uniq (perimeter living '()))))))
```


---
title: Simple Deck of Cards
layout: post
---

This is my little pet project right now, and I have been writing it with ES6 objects. It has been a ton of fun. I want
to make a deck of cards that can play ***any*** card game that the players choose to play, which requires 
practically infinite flexibility.

The goal is to be able to have a deck of cards available to you in your pocket at all times (*eventually a mobile app*).
Right now you can pick which cards are in your game, which suits, decide if Aces are high or low,
how many players are playing, and their names by submitting a form. It won't currently work on CodePen
if your window has a query string, so check for that if you are view on CodePen.

Players can play their cards by clicking, and the card will leave their hand and go into the game's discard pile (not represented visually).
The code lives on my machine in a more organized file structure, and is currently a few development steps ahead of
the Pen, but feel free to ask and I would love to show it off or get constructive crititcism.

This is written completely in vanilla JavaScript without any libraries, which is a personal vendetta of mine that often
leads to JavaScript headaches. I like to know what I'm actually doing under the hood, so I choose to code without libraries sometimes.

[You can gamble away your life's savings here](http://codepen.io/McPhelpsius/pen/vGoOdx).


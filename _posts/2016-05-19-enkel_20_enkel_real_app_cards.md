---
layout: post
title: Creating JVM language [PART 20] - Writing real application with Enkel
categories: [Enkel]
tags: [enkel,jvm,asm,antlr,antlr4,antlr,java,language]
fullview: true
comments: true
---

## Sources

The project can be cloned from **[github repository](https://github.com/JakubDziworski/Enkel-JVM-language)**.  
The revision described in this post is **[5afeaa64a7fb22fad7fe2c30ee440d9a3ff25337](https://github.com/JakubDziworski/Enkel-JVM-language/tree/5afeaa64a7fb22fad7fe2c30ee440d9a3ff25337)**.

## Playing cards drawing simulator

In the previous 19 posts I've been describing the process of creating
my very first programming language. All that hard work I've put into it 
would seem pointless if I'd never used it for anything useful, right?

I came up with an idea of playing cards drawing simulator.
The idea is to provide number of players and number of cards per player.
As an output you get randomized collection of cards for each player. Just like
in real games the cards are being removed from the stack when drawing.

### Card class

```java
Card {

    string color
    string pattern

    Card(string cardColor,string cardPattern) {
        color = cardColor
        pattern = cardPattern
    }

    string getColor() {
        color
    }

    string getPattern() {
        pattern
    }

    string toString() {
        return "{" + color + "," + pattern + "}"
    }
}
```

There is nothing fancy here. Just simple domain object. It is immutable
representation of a playing card. 

### CardDrawer class

```java
CardDrawer {
    start {
        var cards = new List() //creates java.util.ArrayList 
        addNumberedCards(cards) //calling method with 3 arguments (last 2 are default)
        addCardWithAllColors("Ace",cards) 
        addCardWithAllColors("Queen",cards)
        addCardWithAllColors("King",cards)
        addCardWithAllColors("Jack",cards)
        //Calling with named arguments (and in differnet order)
        //The last parameter (cardsPerPlayer) is ommited (it's default value is 5)
        drawCardsForPlayers(playersAmount -> 5,cardsList -> cards) 
    }

    addNumberedCards(List cardsList,int first=2, int last=10) {
        for i from first to last {  //loop from first to last (inclusive)
            var numberString = new java.lang.Integer(i).toString()
            addCardWithAllColors(numberString,cardsList)
        }
    }

    addCardWithAllColors(string pattern,List cardsList) {
        cardsList.add(new Card("Clubs",pattern))
        cardsList.add(new Card("Diamonds",pattern))
        cardsList.add(new Card("Hearts",pattern))
        cardsList.add(new Card("Spades",pattern))
    }

    drawCardsForPlayers(List cardsList,int playersAmount = 3,int cardsPerPlayer = 5) {
        if(cardsList.size() < (playersAmount * cardsPerPlayer)) {
            print "ERROR - Not enough cards" //No exceptions yet :)
            return
        }
        var random = new java.util.Random()
        for i from 1 to playersAmount {
            var playernumberString = new java.lang.Integer(i).toString()
            print "player " + playernumberString  + " is drawing:"
            for j from 1 to cardsPerPlayer {
                var dawnCardIndex = random.nextInt(cardsList.size() - 1)
                var drawedCard = cardsList.remove(dawnCardIndex)
                print "    drawed:" + drawedCard
            }
        }
    }
} 
```

### Running

First we need to compile Card class (no multiple files compilation implemented yet).
Once the Card is compiled CardDrawer is able to find it on classpath (providing we added current dir to classpath)

```bash
java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar: com.kubadziworski.compiler.Compiler EnkelExamples/RealApp/Card.enk
java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar:. com.kubadziworski.compiler.Compiler EnkelExamples/RealApp/CardDrawer.enk

kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java CardDrawer 
player 1 is drawing:
    drawed:{Diamonds,Queen}
    drawed:{Spades,7}
    drawed:{Hearts,Jack}
    drawed:{Spades,4}
    drawed:{Hearts,2}
player 2 is drawing:
    drawed:{Diamonds,4}
    drawed:{Hearts,Ace}
    drawed:{Diamonds,Jack}
    drawed:{Spades,Queen}
    drawed:{Spades,King}
player 3 is drawing:
    drawed:{Diamonds,Ace}
    drawed:{Clubs,2}
    drawed:{Clubs,3}
    drawed:{Spades,8}
    drawed:{Clubs,7}
player 4 is drawing:
    drawed:{Spades,Ace}
    drawed:{Diamonds,3}
    drawed:{Clubs,4}
    drawed:{Clubs,6}
    drawed:{Diamonds,2}
player 5 is drawing:
    drawed:{Hearts,4}
    drawed:{Hearts,Queen}
    drawed:{Hearts,10}
    drawed:{Clubs,Jack}
    drawed:{Diamonds,8}
```

Great! This proofs that Enkel,though in early stage, can already be used for creating some real applications.

# Goodbye Enkel

I had a really great time creating Enkel and sharing the whole process here.
Writing code is one thing, but describing it in a way, 
that other people could understand it is another challenge itself.

I learned a lot during the process, and hope some other people
benefited (or will benefit) from the series too.

Unfortunately this is the last post of the series. The project will however be continued,
so keep track of it! 


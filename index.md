# Scala-Mungo tutorial

## Introduction
Scala-Mungo is a tool which lets you add a protocol/typestate definition to your classes. 
In this tutorial you will learn how to install Scala-Mungo, create a protocol for a class see an example of a program using that class; there are also exercises to try at the end.

## Installing Scala-Mungo

### SBT
Copy in these lines into your build.sbt file:

```markdown
resolvers += Resolver.bintrayRepo("aliceravier", "maven")

autoCompilerPlugins := true

addCompilerPlugin("org.me" % "scala-mungo-prototype_2.13" % "1.9")
val root = ABSOLUTE-PATH-TO-YOUR-PROJECT
scalacOptions += "-P:GetFileFromAnnotation:"+root

libraryDependencies += "org.me" % "scala-mungo-prototype_2.13" % "1.9"
```
Instead of ABSOLUTE-PATH-TO-YOUR-PROJECT, put in the absolute path to your project, i.e. the path to the build.sbt file.

It should look like this:
`val root = "C:\\Year five\\Scala-Mungo-dir\\Test"`

This is used for the plugin to find the location of the protocols you will define.

## Fixes for problems during setup
I have been intellij and sbt for testing and have found that a lot of problems can be fixed with three techinques:
- invalidate caches and restart
- use `sbt run` instead of running the code from the editor
- put all the code into one file (classes and program to run)


## Creating a protocol
A protocol consists of states that instances of the class can be in, and transitions between the states facilitated by method calls. 

### Specification

- The protocol must be defined inside an object which extends "ProtocolLang" and the code for the protocol must be in the main function of that object.

- States are defined with the "in" function which takes a String: the name of the state. Example: `in("init")`

- Transitions from a state can be added underneath a state definition with the "when" method which takes a String of the method signature as an argument, and the "goto" method which takes a String of the state to transition to as an argument. Example: `when("giveMoney(Int)") goto "moneyGiven"` (note that parenthesis aren't needed for the goto method's argument).

- To write a transition which depends on the return value of a method, the "at" and "or" methods are used. The "at" method specifies the return value which enables the transition and takes the return value as a String as an argument. The "or" method lets you add a different return value and takes the String of the next state to go to for the different return value. You can keep adding "or"s to add as many return values as you want. Example:
`
when("authorise()") goto 
    "Authorised" at "true" or 
    "Unauthorised" at "false"
`

- Protocols must contain a unique "init" state which will be the state given to an instance when it is initialised.

- Protocols must contain a unique "end" state which indicates protocol completion, that is, at the end of a program, the object should be in the "end" state. All states must have a path of transitions between themselves and the "end" state.

- You must end your protocol with the "end()" method.

- The protocol file must be called by the name of the object which the protocol is defined it. Example: a protocol defined in the object "ATMProtocol" should have "ATMProtocol.scala" as the file name.


### Example
Let's say we want to create an ATM object which should follow a certain protocol. We want it to be able to take a card in, check if it's authorised, and then give money if it is or eject the card if not. Then we want the transaction to be available again. We need to include an "end" state which we can place right next to the "init" state, where all the transactions should start from.
We come up with the following state machine:
![ATM state machine]({{ site.url }}/assets/ATMProtocol.png)

Now we can write our protocol. We need to extend "ProtocolLang" and create a main method:
```markdown
import ProtocolDSL.ProtocolLang

object ATMProtocol extends ProtocolLang with App{

}
```
This needs to be in a file called "ATMProtocol.scala".

Then we can define the states we want:
```markdown
import ProtocolDSL.ProtocolLang

object ATMProtocol extends ProtocolLang with App{
  in("init")
  in("CardIn")
  in("Authorised")
  in("ShouldEject")
  in("end")
}
```

Then we can define the transitions for all the states:
```markdown
import ProtocolDSL.ProtocolLang

object ATMProtocol extends ProtocolLang with App{
  in("init")
  when("takeCard()") goto "CardIn"
  when("beginNewTransaction()") goto "init"
  in("CardIn")
  when("authorise()") goto 
    "Authorised" at "true" or 
    "ShouldEject" at "false"
  in("Authorised")
  when("giveMoney()") goto "ShouldEject"
  in("ShouldEject")
  when("eject()") goto "end"
  in("end")
  when("beginNewTransaction()") goto "init"
 }
```

And finally we add the "end()" method at the end of the protocol:
```markdown
import ProtocolDSL.ProtocolLang

object ATMProtocol extends ProtocolLang with App{
  in("init")
  when("takeCard()") goto "CardIn"
  when("beginNewTransaction()") goto "init"
  in("CardIn")
  when("authorise()") goto
    "Authorised" at "true" or
    "ShouldEject" at "false"
  in("Authorised")
  when("giveMoney()") goto "ShouldEject"
  in("ShouldEject")
  when("eject()") goto "end"
  in("end")
  when("beginNewTransaction()") goto "init"
  end()
}
```

Then we can run the protocol, which should create a file called "ATMProtocol.ser" inside a "compiledProtocols" directory in the project root. 
Now the plugin can find it and use it to check our ATMs are running correctly!

Make sure to run the protocol before writing code which uses it or the plugin will complain about not being able to find the protocol. In that case, comment out the @Typestate annotation and run the protocol again.

## Using a protocol in a program
Adding a protocol to a class in the program is easy. You only need to add the @Typestate annotation to the class which should be following the protocol. 
The annotation takes a String of the name of the object which the protocol was defined in as an argument.
In our ATM example:

```markdown
import compilerPlugin.Typestate

@Typestate("ATMProtocol")
class ATM(){...}
```

Here is a full example program which uses the ATMProtocol defined above:
```markdown
import compilerPlugin.Typestate

@Typestate("ATMProtocol")
class ATM {
  def takeCard(): Unit ={}

  def authorise(): Boolean ={
    var cardIsValid = false
    //code which checks if the card is valid
    cardIsValid
  }

  def eject(): Unit ={}

  def giveMoney(): Unit ={}

  def beginNewTransaction(): Unit ={}
}

object ATMtest extends App{
  val myATM = new ATM()
  while(true) {
    myATM.beginNewTransaction()
    myATM.takeCard()
    myATM.authorise() match {
      case true =>
        myATM.giveMoney()
      case false =>
    }
    myATM.eject()
  }
}
```
This should not cause errors to be thrown.

Try removing the "myATM.eject()" line. This should throw an error saying that the beginNewTransaction() method was called inappropriately.

If the program isn't running and is complaining about not finding the protocol, make sure you have got a ATMProtocol.ser file inside the compiledProtocols directory. If not, comment out the @Typestate annotation and run the ATM protocol defined in the above section.

## Exercices

### 1: Do the ATM example explained above

### 2: An aliasing example
As a programmer, you want to model the flow of cash in a company. amounts of money dealt with should always be acted on in a certain way: they should be filled, have interest added to them and only then be used. 
Given the following state machine representation, create a protocol in Scala-Mungo for a stash of money.
![MoneyStash protocol]({{ site.url }}/assets/aliasing protocol.PNG)

You also want to have this money used by managers and a database. 
In the code below, a manager and a database are created which both take the same MoneyStash instance as a field. 


*A note on aliasing*

An instance which has two ways of referring to it is called "aliased". So in this case the MoneyStash instance is aliased.
This could cause problems since the manager and database don't know what each is doing on the other's MoneyStash instance. If both applied interest that would be illegal from the protocol's standpoint, but would be invisible from the standpoint of individual variables.
Scala-Mungo tracks all the aliases (variable names) for a given instance so that this doesn't happen.

Copy the code below into a file and check it works with the protocol defined above:
```markdown
@Typestate("MoneyStashProtocol")
class MoneyStash() {
  var amountOfMoney : Float = 0
  def fill(amount : Float ) : Unit = {amountOfMoney = amount}
  def get() : Float = amountOfMoney
  def applyInterest(interest_rate : Float) : Unit = {
    amountOfMoney = amountOfMoney * interest_rate;
  }
}

class DataStorage() {
  var money : MoneyStash = null;
  def setMoney(m : MoneyStash) : Unit = {money = m}
  def store() : Unit = {
    var amount = money.get()
    println(amount)
    // write to DB
  }
}

class SalaryManager() {
  var money : MoneyStash = null;
  def setMoney(m : MoneyStash) : Unit = {
    money = m
  }
  def addSalary(amount: Float) : Unit = {
    money.fill(amount)
    money.applyInterest(1.02f) 
  }
}

object Demonstration extends App {
  val salary = new MoneyStash
  val manager = new SalaryManager
  val storage = new DataStorage

  manager.setMoney(salary)  
  storage.setMoney(salary)  

  manager.addSalary(5000)   
  storage.store()           
}
```

Now try adding a "money.applyInterest(1.02f)" line above the "var amount = money.get()" line in the DataStorage class. This should cause an error when run.

### 3: Create a protocol from a written specification
Now write a protocol for a cat, or any other animal of your choice.

- The animal must be able to complete any number of walk()-slow()-stop()-startAgain() cycles, from init.

- It must also be able to complete any number of walk()-run()-slow()-stop()-startAgain() cycles, from init.

- From init, when calling the sleep() method, it might fall asleep (return true), in which case it should then be able to call awaken() and then startAgain(). It should be able to repeat this infinitely.

- From init, it might also call the sleep() method which would return "false", in which case it shouldn't change anything to the state. It can call sleep():false an infinite number of times.

Remember that protocols must contain a unique "end" state which indicates protocol completion, that is, at the end of a program, the object should be in the "end" state. All states must have a path of transitions between themselves and the "end" state.

Once you have written this protocol, run it and then write a program which uses your animal class and does not error. 
Then write a program which does error.
You may use the class below or write your own one:
```markdown
class Cat{
  def walk(): Unit ={

  }
  def slow(): Unit ={

  }
  def stop(): Unit ={

  }
  def run(): Unit ={

  }
  def sleep():Boolean ={
    true
  }
  def awaken(): Unit ={

  }
  def startOver(): Unit ={

  }
}
```

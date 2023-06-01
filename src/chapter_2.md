# Creating Concept for Project

Proposals are means where we create a technical concept discussing mostly on one approach and an overview of how would we complete the project. But once you are the choosen one then there is a famous saying

> " With great Powers comes great responsibility "

Now, It is time to re-evaluate all possible alternatives which are available and tools or framework which are most compatible with existing project in a way that you do not tweak it upto the extent that it looks boring to maintainers for reviewing.

I had my meeting with MathJud and he gave some really interesting points on the project to get started and some things which I have figured out, both together helped us in making a good plan.

---
I had read and researched through the `RuMa` and `Matrix-SDK-Rust` projects since this is now the part which we will be using for our project instead of GoLang.

Mathjud suggested that I should duplicate the qaul-cli binary and tweak it in whatever way I would need to in order to work on it. Here is the main reason why we chose the qaul-cli specifically for implementing the bridge concept.

- It already has two workers set in place which will check for any activity on entire qaul network each 10ms.
- We have access to CLI which we use to interact with RPC protocol and protobuff messaging.

I was reading through the documentation of the matrix-sdk crate and what I found was a beautifully commented codes for creating the bot inside `examples/` directory. It has been few months, I am looking at some rust projects and this is something really important which I found over here. Learning point is that many rust crates are set of APIs. There is an directory called `/examples` as the name suggests it consists of examples about how to use those APIs. From now-onwards, I will always open up this directory if I am using any external crate to get my things done. I used one of the example and set-up locally another matrix account and run that example and it worked really smooth.

--- 
## Plan for the bridge 

Now, I know about the matrix-sdk and its usage into the project and also tried using the discord bot, I have moved towards creating a better and concrete plan of what I am supposed to do. Here is an excerpt from my notes.

### Version 0 ###
[On Matrix]
1. Create a bot account for Qaul and specify a server to work on.
    homeserver : https://matrix.org
    username : qaul-bridge-bot
    password : #8u0r8f34842
2. Invite the bot to the testing matrix room.

[On Qaul]

3. Create a binary copy of qaul-cli
4. Code the logic to login our bot into the matrix room as soon as the qaul-cli binary is running.
5. Also Code a basic testing functionality (For Eg : Call it !ping command)

[On Matrix]

6. Login with our personal account [@harshil1] and send a message with !ping in the room.
7. In response we should receive all the nodes connected to the network.

### Version 1 ###
Instead of just an echo as response, We should pick the messages from both the ends.
Send "Hi" from qaul and it should first sense in matrix without our personal human activity that there is some event triggered in qaul. Once event is detected, the message should show up into the matrix room.

Next we can reverse engineer the above feature and do the same in qaul by sending a message in matrix room.

### Version 1+ ###
After the above completions, We can think of double puppeting the bot so now our bot is not just `qaul-bridge` but a real username from the qaul node.

---

Finally, We settled on the above given plan and meanwhile, I have had started with the implementation based on that plan.
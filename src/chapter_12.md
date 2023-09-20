# CHAOS Communication Camp

This is a camp for wireless mesh networks happening in Berlin. There was a Matrix Camp with few members from Element being the part of the camp. My mentor has a friend in `Element` who invited us to present our bridge. 

My mentor was into the organizing team and had got no time to test the bridge. And I always tested it locally on my personal network. On the day of presentation, We were doing a live demo in front of people from Matrix on real community node which is a global node in Qaul and interconnects you with the entire qaul community. 

As soon as my mentor created a new user into the community node, there was a crytography error in creating a user and the malicious peer ID was introduced into the network. By default CLI client ignores such uncertain peerID's but the code which I had written by that time was expecting correctness and was filled with `unwrap()`'s. As soon as bad ID entered into my bridge binary's database, All of my code just broke that too in a live presentation.

I took a moment and I was converting every `unwrap()` into `match` statement to just discard wrong things and not panic. But how long can you do that in a live presentation isn't it ? It was an embrassing feeling. I really thank my mentor who kept the viewers engaged into the conversation by asking matrix specific questions and explaining them out approach to create a bridge.

Then as a part of very quick solution, I have deleted the database and created a new binary user and re-run whole bridge and showed it in front of presenters all our basic working functionalities.

This presentation was a bit messy and embrassing since we have never tested on community node and were completely unaware about how a wrong peerID can create this huge of a problem for us.

In the end, The feedback which we received was kind enough and they said that the direction we have choosen is correct. We just need to fix the bugs which we just introduced for the very first time.

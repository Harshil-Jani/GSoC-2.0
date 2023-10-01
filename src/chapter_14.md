# Matrix Communication Summit 2023

The Matrix summit was an awesome outcome of our project showcase. There is a back story which I would like to share between me and my mentor. It was 48 hours before our presentation where we tested again for the second time on the community node. Somehow, We got lots of panicks which were due to cyclic initialization. The problem was that if we don't wait for certain miliseconds then there was a cyclic loop asking for initialization of Matrix from Qaul and Qaul from Matrix. 

We both tried up straight for hours and finally, I implemented the temporary hack that we can wait for few seconds for the both parts to respond.

We did a lot of testing before the actual meeting and we faced latency issues. This was casued because while testing on community node, It finds device via mesh networks. So, Technically someone from Europe would need their device to get routed to mine and forward the connection to my mentor's device and thus the high latency. To resolve this huge latecy, We decided that we will be presenting them on our own local qaul network which would actually be the ideal case for average use.

---

Coming to the presentation part, It went very impressive. There were bunch of people from Element, RuMa, Matrix-SDK etc. in the Matrix Summit and they all were curious to learn first about qaul (given by my mentor) and then I explained them about the need of the bridge.

Then we had a smooth live demo. In the demo we did the most impressive thing. We opened a public chat room in matrix, Invited them all to join the room and interact with the bridge. They were happy to see it working. And finally, We got appraised for creating this beautiful project using their tools and SDKs.

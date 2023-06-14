# Concrete Version-0 usability plan

## Usability Plan

By the time, I reported back to mentor MathJud about my integration of matrix with qaul with a simple command `!qaul` from matrix and It would return a string saying "I am a message from qaul", We got on another call to discuss more concrete planning for the usability. 

He gave some suggestions :
### Relay Bot

The bot receives the invite into a qaul-room.
The bot accepts the invite and opens a public room with the same name in matrix.

#### Public Messaging
- Forward Public Messages from qaul to specific matrix-room.
- Forward Public Messages from Matrix to qaul public messages.

#### Group Chats
The bot will forward messages from the qaul chat room to matrix chat room and vica versa.

### Puppeting
The puppeting shall create a 1-to-1 communication relation between qaul and matrix. In order to make that happen, The matrix bridge needs to have administrative rights on the matrix server in order to create a matrix user for every qaul user writing a message. The bridge also needs qaul users for matrix users that shall be able to communicate over it. The creation of qaul users should be handled restrictive, as they would be announced in the qaul network and create overhead in the routing table.

## Concerns

- Can our Matrix room use same UUID which we use for qaul rooms ?
- As there is no concept of open chat room in qaul and all users needs to be invited by the group admin, Whether it makes sense to let the chat-bot open a qaul room as the bot would need to manually invite qaul users in it ?
- Should we have Auto-Joining for joining any Matrix room willing to do so ? If yes, We may need our own Matrix Homeserver so that we only have trusted people onto the network. [Currently I own the qaul-bot account on my university ID and manually create rooms and accept invites from my another account] If we don't own any homeserver then AutoJoin would not be a good way to do this since the primary concern of qaul may be at risk.
- Puppeting is a next big story and we would address it later in the journey.

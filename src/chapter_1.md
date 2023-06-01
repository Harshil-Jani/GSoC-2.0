# Community Bonding Period

Comunity Bonding period is dedicated for better understanding of the community, project and mentor-mentee connection. And we did all of that very well.

## Finalizing on RustLang over GoLang
Initially we thought that we would run matrix bridge as a daemon process and use `Golang` to create the bridge. But, My mentor whom from now onwards I will be calling as **MathJud** has a friend working in element, which is a matrix client. He suggested us that since the world is taking on for the Rust, Matrix-SDK are actively written in Rust and there is something called as `RuMa` which stands for `Rust Matrix`. RuMa is an amazing work and thus, We have decided to do it in rust only because our Qaul has its entire backend in Rust.

RuMa : [ruma.io](https://ruma.io/)

## BattleMesh Conference Insights
**MathJud** was attending a conference in Spain which was Wireless Battlemesh. There he gave a talk on qaul and its feature and future. In between the conference, I was expecting to do a meeting but since he was bit occupied over there we rescheduled it. I saw the recording later and found a good archietectural picture of the qaul's tech stack with libqaul running in backend and Flutter used for GUI. I was not present over there physically but it looked really good.

Conference Information : [Battlemesh V15](https://www.battlemesh.org/BattleMeshV15#Talk_Schedule_and_Workshops)

## Writing first blog post in freifunk community
All of the GSoC students were given access for the freifunk wordpress and were expected to publish 3 blogs minimum each with their respective deadline. I wrote my very first blog post giving a general overview of my work.

Freifunk Blog : [Qaul Matrix Bridge GSoC 2023](https://blog.freifunk.net/2023/05/14/gsoc23-qaul-an-internet-independent-communication-application/)

While MathJud was busy with conference, I wrote this blog and sent him for review. After getting it reviewed, I had published this blog.

## Trying Discord<->Matrix Bridge

The law of universe is that you need to try something out before you dive into it. Before we do imagination in clouds for the project, I thought about trying one of the existing bridge. I tried with a Typescript written discord bridge to understand the workflow. There should be a bot which you need to invite to both the ends and then rest it can take forward. Using this as an example, we created a concept for how we would approach to deal with our bridge bot.

Discord Matrix Bridge : [t2bot](https://github.com/t2bot/matrix-appservice-discord)

## Coding
I am aware that we are still in community bonding phase, but in software world deadlines and unknown problems are hard to face. So, Any great and good programmer would always advice to start early on coding. And so did my mentor MathJud. We started implementing the solution code wise once we had a plan ready about how to implement things as expected.

I will talk about coding in the upcoming chapters and in this book each chapter will represent a block of milestone achieved in our project. So separating it out into different chapters will be the best.

Linking Pull Request Raised : [Adding a relay bot for matrix bridge](https://github.com/qaul/qaul.net/pull/563)
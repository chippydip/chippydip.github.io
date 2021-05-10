---
layout: post
title:  "New Blog - New Game"
---

I'm starting a new blog! I've been an avid gamer and huge space nerd all my life. Four years ago I started working on a design doc for a game I wanted to make and a month later spent a few days prototyping the first bits of [that game](https://github.com/chippydip/spacegame/tree/c8238950f528aa38eed33f57299736d604915709). Unfortunately life got in the way and I didn't stick with it. I've decided to give this game another shot, and in order to help keep myself motivated, I've decided to also start a blog to document my progress. My thinking is that if I commit to writing one post per week I'll be more inclined to make sure I actually do something worth writing about. That may not alway be coding, it could be thinking about game design, researching how something could be implemented, or anything else as long as it's working toward the goal of creating this game.

So, what is this game? Glad you asked! It's inspired by [Aurora 4x](https://aurora2.pentarch.org/), often described as the [Dwarf Fortress](http://www.bay12games.com/dwarves/) of the [4x](https://en.wikipedia.org/wiki/4X) world, Aurora is a game about space colonization and conquest with all of the visual flair of playing with spreadsheets. I love the idea of building an interstellar civilization from the ground up, planning when and how to expand, and dealing with challenges along the way. I never got very far into the actual game, though, since I found the level of micromanagement to be too much for my taste. I like the idea of being able to control every minute detail, but want automation and smart defaults so I can focus on the areas I care about and trust that the rest of my empire will continue to hum along if I've set it up properly. My initial design ideas are:
* Automation with smart defaults - micromanaging should be a choice, not an obligation
* Hard-er science fiction - I'd like the game to be an exploration of what *may* be possible in the future
* Multiplayer? - this will be tough to get right, so more of a stretch goal, but supporting multiple player-controlled empires in real-time would be really neat

I'm planning to mostly start over with the implementation, probably drop Three.js and work with WebGL directly. I'm also planning to drop the Go backend and keep the entire thing in TypeScript/React and look into WASM for critical path code should that be needed. Working through these and other tech decisions will be something I plan to include in future blog posts.

I've got a lot more ideas and even more designs left to write, so this will be a long-term hobby project, but I'm happy to be starting it up again. I'm looking forward to documenting my progress here on a weekly basis, so stay tuned for more updates!

## Last Week
* Setup this blog - this is just a simple Jekyll blog hosted on GitHub pages, I may look at switching to Hugo in the future, or other options, but maybe not. This was good enough to get going so I can focus on what I really care about: building a game!
* Wrote my first blog post - I'm happy to have the first one under my belt, it's nothing fancy, but I think it's a good start

## Next Week
* Write some code - I plan to rip out most of the old code, I'll probably drop Three.js and just do my own rendering. I'd like to experiment with some different orbital path drawing options, but may start with just the orbiting bodies to get something up and running (again) as a jumping off point.

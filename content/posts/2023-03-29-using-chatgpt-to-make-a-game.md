---
type: post
title: 'making a game with chatgpt'
date: '2023-03-29'
author: arges
tags:
- chatgpt
- ai
- react
- games
---

There is obviously a lot of hype around ChatGPT and GPT4 right now. I wanted to
take it on a test drive and see if it could help me rapidly prototype and write
some code. I actually wanted to try an idea my daughter had for a game. She
has been very interested in how businesses work and said it would be cool if
there was a game where you had to run a lemonade stand.

I thought this would be cool to write as a game that runs in a browser.
Drawing heavy inspiration from Paperclips Unlimited, I wanted to make something
barebones and scrappy. The first thought was to make a React app so this game
could easily be played on her computer and make it easier for her to modify.
The trouble is, I don't have much modern web development experience! So let's
augment my brain with AI.

For the first attempt, I just told ChatGPT some really basic ideas like, "Make
a React App that has a button and displays a number". This spit out some reasonable
looking code. However, I had no idea how to run it. Next question was, "How can
I run this on my computer?" Maybe I don't need AI and I can just go straight to
stack overflow. Regardless interacting with ChatGPT worked well and while I was
tempted to open a new browser tab and start searching, it was kind of nice to
have this dialog in a single chat window. I feel like my brain is well tuned to
sifting through search results and picking out the obvious wrong answers.
However with the chat interface (and perhaps the text delay), it felt like I
was talking with a co-worker.

Once I got the app up and running, I then started adding features to the
original program. This is where I started to get blown away. Not only was the code
getting generated that I could just copy and paste and run, but there was also an
explanation about what was being generated. Although in subsequent iterations
the generated code would "forget" things I had already added, it was easy to merge
iterations of changes together. This felt like some sort of bizzaro pair
programming exercise. Wow.

At some point I had a draft program. I didn't talk to ChatGPT about making this
a game but really just described how I would start programming this by adding
elements that would start adding up to the full game. This is the draft:
```
import React, { useState, useEffect } from 'react';

function App() {
  const [balance, setBalance] = useState(0);
  const [customerRate, setCustomerRate] = useState(1); // Set default rate to 1 second

  useEffect(() => {
    const interval = setInterval(() => {
      if (balance > 0) {
        setBalance(balance - 1);
      }
    }, customerRate * 1000);

    return () => clearInterval(interval);
  }, [balance, customerRate]);

  const handleAddDollar = () => {
    setBalance(balance + 1);
  };

  const handleDeductDollar = () => {
    if (balance > 0) {
      setBalance(balance - 1);
    }
  };

  const handleCustomerRateChange = (event) => {
    setCustomerRate(parseInt(event.target.value, 10));
  };

  return (
    <div>
      <h1>Business Expenses</h1>
      <h2>Balance: ${balance}</h2>
      <button onClick={handleAddDollar}>Add $1</button>
      <button onClick={handleDeductDollar}>Deduct $1</button>
      <div>
        <label htmlFor="customerRate">Customer rate (seconds):</label>
        <input type="number" id="customerRate" value={customerRate} onChange={handleCustomerRateChange} />
      </div>
    </div>
  );
}

export default App;
```

Now I decided I had a good base, and I could go into manual editing mode. As I
started to hack, I realized I didn't know good React patterns, so I would
occasionally ask how I would do something in React given a more familiar language
like Go. This was extremely useful and I was able to get really fast results
doing this.

I had a basic game working in about 10 minutes. Then I started to realize, ok I
need to do some refactoring. Instead of relying on myself, I started feeding
segments of the code back into ChatGPT and asking it to do it. In particular I
had a bunch of text separated as headers, and I asked if it could be 'turned
into a table', and it worked.

A bit more hacking and I had a 'game'. I will admit it isn't very fun, but I'm
pretty amazed that was able to make something in a languge I knew almost
nothing about in around an hour. 

Now for the speculation part. Is this a game changer? Absolutely yes. Do I
think it will eliminate humans doing programming? Not soon. This feels like
going from the Dewey decimal system to a search engine. It is a big leap, but
people still need to choose which books they want to read.

I think as software developers, we'll need to get better at describing what
actually needs to be built. I also wonder as new languages, frameworks and
patterns happen, will the models have a large enough corpus to use these new
technologies? For this example I choose a well trodden path, but perhaps as a
newbie I didn't realize I was using some outmoded pattern.

I think of this AI as another person to work with. Results don't lie, but you
still have to expect mistakes... Ironically just like working with a human.

Here's a screenshot of the game:
![Screenshot](/images/game1.png)

You can see the source code here: https://github.com/arges/lemonade-stand

---
title: 'A few reckies'
description: 'Like a reckon, but smaller'
pubDate: 'Feb 22 2023'
---

While we've been talking a lot about the bigger changes we're making in the challenges. I've been keeping some reckons to myself, here's a few of the smaller changes I think reckon we should make. I call them "my little reckies".

## Let's swap Webpack and Babel for Vite

I had a bunch of conversations about this a while back. Essentially Webpack has two big drawbacks compared to its competitors.

- Bespoke config: to build our challenges, we need to configure babel and webpack, and when students start adding css etc to their build we need to make even more changes
- Many dependencies: We end up bringing in a lot of packages that webpack or babel depend on and these add to install time and need to be managed
- It's really slow: any of the proposed replacements are much faster

Many modern bundlers or transpilers understand JSX and typescript out of the box.

Previously, we looked at couple of good options:

1. vite
2. esbuild

Since then [vite](https://vitejs.dev/) has become the mainstream choice in this space, Rohan has been using vite in teacher led projects and so has the Tāmaki team.

So in short:

Let's remove Webpack and Babel from all the challenges and replace them with vite

## Let's replace ES Lint and Prettier with Rome

Eslint and prettier are both doing a fine job, but they have almost the exact same drawbacks as webpack.

- Bespoke config: to lint our challenges we maintain a package called @devacademy/eslint-config
- Many dependencies: We end up bringing in a lot of packages that eslint or prettier depend on and these add to install time and need to be managed
- It's really slow: any of the proposed replacements are much faster

[Rome](https://rome.tools/) is a new toolset that understands TypeScript and JSX out of the box, and operates well on our codebase with no configuration.

It runs super fast. I've tried using it in the challenges monorepo and it completes instantly. More importantly, it gives amazing warnings:

This is a snippet from running `npx rome check` in the monorepo.

```
packages/sweet-as-beers-rtk-solution/client/components/Cart.tsx:58:9 lint/a11y/useButtonType ━━━━━━━━━━

  ✖ Provide an explicit type prop for the button element.

    56 │       <p className="actions">
    57 │         <a onClick={handleClick}>Continue shopping</a>
  > 58 │         <button>Update</button> {/* TODO: implement updates */}
       │         ^^^^^^^^
    59 │         <button className="button-primary">Checkout</button>
    60 │       </p>

  ℹ The default  type of a button is submit, which causes the submission of a form when placed inside a `form` element. This is likely not the behaviour that you want inside a React application.

  ℹ Allowed button types are: submit, button or reset
```

I think these informative warnings will be great for both students and teachers.

Rome would also replace prettier. It formats in a pretty similar way, but runs much faster. Which brings me to one of the reasons I haven't been talking about this change much.

_Rome formats with semi-colons_ so this means that we have to have the boring semi-colon conversation again.

So ... I don't know that I have it in me _right now_ to argue about punctuation, but sometime soon I'm going to shotgun a can of monster and we'll have it out because the upsides on this thing are too good.

## Let's do async/await officially

Right now I think we're a bit vague about when we're using async/await vs. `.then(...)` and `.catch(...)`.

Some campuses are teaching `async`/`await` really early, and some maybe not at all.

I think we should all teach it early. Maybe as early as Pupparazzi.

Ironically, I think a lot of the students don't have as hard a time with it as we do. When I learned `async`/`await` it was "new syntax" and I had already committed a good chunk of my brain to learning how to get things done with `.then(...).catch(...)`. For students, it's just "syntax" and they're still in the phase of learning syntax.

Pros:

- it makes error handling and cleanup a lot simpler
- they're going to see it in documentation
- all the code at their first job will use it

Cons:

- your functions have colors now
- more syntax to learn

Anyway, you can all come beat me up in the dojo after I'm finished having all the semicolon fights.

## I think we should use sqlite in production

Currently we switch to postgres **just for production**, I think this is a leftover from when we were on Heroku.

Now with the dokku storage plugin, we can use sqlite3 the whole way (there's a problem in the current instructions but I'll have this sorted shortly).

This will avoid a lot of stress during the last couple of weeks when students are suddenly dealing with the not-that-well-documented differences between knex providers.

Importantly, using a different database engine in production vs. your tests I think sets a really bad example for students in how they can think about testing.

Your tests should give you confidence to ship your code to production. If your tests run on sqlite and your app runs on postgres, how can your tests give you that confidence?

## Don't panic

Nothing here is on the Roadmap yet, precisely because we haven't talked about them enough or made a decision, this is just my personal wishlist.

# Development log

All of the steps I'm taking to build this app, from start to finish.

- [Day 1: Concept and tech](#day-1)
- [Day 2: Listing views, schema design, running mock server](#day-2)

<h3 id="day-1">Day 1: Concept and tech</h3>

It's Friday 5/13, and we have decided to come up with a new example app.

[We wrote down all of the stuff we want to demonstrate about building an app with Apollo.](README.md)

I've also made a list of technology choices, listed in the README:

- Apollo server - to put a nice unified API on top of our GitHub and local data
- Apollo client - to load that data declaratively into our UI
- React - it's a great way to build UIs, and has the best integration with Apollo and Redux
- React router - it seems to be the most popular React router today, and has some great hooks and techniques for SSR. It seems like James Baxley has had some success with implementing this stuff already with React Router.
- Webpack - the Meteor build system is by far the most convenient, but comes with a dependency on mongo, its own account system, etc. Since we want to learn how Apollo works without all of these things, we're going to not use it even though it would reduce the complexity of the code.
- Babel - to compile our server code.
- Redux - to manage client side data, perhaps we can also use Redux Form to manage the submission form, but we'll see when we get there.
- Passport.js for login - this seems to be the most common login solution for Express/Node, and has a great GitHub login solution. Jonas has already written a tutorial for using that with GraphQL.
- SQL for local data - We'll use SQLite for dev, and Postgres for production. TBD - should we use Sequelize or some other ORM? Or just Knex for query building?
- Bootstrap for styling - I already know how to use it, and I don't want the styling to be the focus of the app. Let's keep it as simple as possible.

I drew a [quick mockup of the desired UI](mockup.jpg). It's quite simple, and should be easy to implement in bootstrap.

<h3 id="day-2">Day 2: Listing different views and designing schema</h3>

We want to make this app as simple as possible, but also useful. I think I'll design it the way I would a real app, and then simplify if necessary. Here are some ideas for different views in the app:

- Home page: "Hot" feed, which ranks by number of upvotes and time since posting
- "new" page, which lists stuff that was just posted, but hasn't had time yet to be upvoted
- Submission page, which is a form that lets people put in info about the repository - perhaps it's just a link to the repo, maybe a comment as well?
- Repo page, where you can look at a particular repository, and its upvotes and comments

Given this, let's design our GraphQL schema for the app. I think this is a great place to start the process because it will let us set up some mocked data, and work on the UI and API in parallel as we need to.

#### Repository

This represents a repository on GitHub - all of its data is fetched from the GitHub API. I think it will be convenient to separate data from different APIs into different types in my schema, rather than having one type that merges both the local data (upvotes etc) and the GitHub data (description etc). This can theoretically have [all of the fields that GitHub returns](https://developer.github.com/v3/repos/#get), but we're probably interested mostly in high-level data like:

- Repository name
- Organization/author avatar
- Description
- Number of stars
- Number of issues
- Number of PRs
- Number of contributors
- Date created

#### Entry

This represents a GitHub repository submitted to GitHunt. It will have data specific to our application:

- User that posted this repository
- Date posted
- The related repository object from GitHub
- Display score (probably just upvotes - downvotes)
- Comments
- Number of comments

#### Comment

Just a comment posted on an entry.

- User that posted this comment
- Date posted
- Comment content

#### User

Someone that has logged in to the app.

- Username
- Avatar
- GitHub profile URL

#### In GraphQL schema language

Now let's put it all together to see what our GraphQL schema might look like. Let's keep in mind that this is just a sketch - the mechanics of our app might require some changes.

```graphql
# This uses the exact field names returned by the GitHub API for simplicity
type Repository {
  name: String!
  full_name: String!
  description: String
  html_url: String!
  stargazers_count: Number!
  open_issues_count: Number

  # We should investigate how best to represent dates
  created_at: String!
}

# Uses exact field names from GitHub for simplicity
type User {
  login
  avatar_url
  html_url
}

type Comment {
  postedBy: User!
  createdAt: String! # Actually a date
  content: String!
}

type Entry {
  repository: Repository!
  postedBy: User!
  createdAt: String! # Actually a date
  score: Number!
  comments: [Comment]! # Should this be paginated?
  commentCount: Number!
}
```

Looks pretty good so far! We might also want some root queries as entry points into our data. These are probably going to correlate to the different views we want for our data:

```graphql
# To select the sort order of the feed
enum FeedType {
  HOT
  NEW
}

type Query {
  # For the home page, after arg is optional to get a new page of the feed
  # Pagination TBD - what's the easiest way to have the client handle this?
  feed(type: FeedType!, after: String): [Entry]

  # For the entry page
  entry(repoFullName: String!): Entry

  # To display the current user on the submission page, and the navbar
  currentUser: User
}
```

OK, one last thing - we need to define a few mutations, which will be the way we modify our server-side data. These are basically just all of the actions a user can take in our app:

```graphql
# Type of vote
enum VoteType {
  UP
  DOWN
  CANCEL
}

type Mutation {
  # Submit a new repository
  submitRepository(repoFullName: String!): Entry

  # Vote on a repository
  vote(repoFullName: String!, type: VoteType!): Entry

  # Comment on a repository
  # TBD: Should this return an Entry or just the new Comment?
  comment(repoFullName: String!, content: String!): Entry
}
```

It's not yet clear to me what return values from mutations should be. There are a few possible approaches:

1. Return parent of thing being modified - then we can incorporate the result into the store more easily
2. Return the thing being modified - then we have to figure out where it goes into the store
3. Return the root query object itself, so that we can refetch anything we want after the mutation, in one request

I'd like to try all three eventually and see how they feel in this app. Apollo Client should probably have one or two situations that it deals with the best, so that we can recommend people use that for best results.

#### Getting mocked server running

One great function of Apollo Server is the ability to easily mock data so that you can run some queries against your server without writing any resolvers. Let's get that going.

To do that, we need to set up some build tooling for our server. This would be extra trivial if we used Meteor, but we'll try to go the more generic route and set up babel ourselves. I googled and found this great simple setup: https://github.com/babel/example-node-server

First, let's install some packages:

```txt
npm init
npm install --save-dev nodemon babel-cli babel-preset-es2015 babel-preset-stage-2
```

Let's add a `start` script to our app. We're using `nodemon` so that our app restarts automatically if we change server code.

```js
"scripts": {
  "start": "nodemon index.js --exec babel-node --presets es2015,stage-2"
}
```

Then if we set up some "hello world" index file, we can run `npm start` and confirm that compilation worked.

Let's get Apollo Server going!

```txt
npm install --save express apollo-server
```

Then I massaged the boilerplate from the starter kit: https://github.com/apollostack/apollo-starter-kit

See the code in this commit: https://github.com/apollostack/GitHunt/commit/5f5ad68b28d15139e10d653c1ad24b15535d2706

Now, I have a mocked version of my schema going, after some syntax errors like writing `Number` instead of `Int` in my sketch above. The default mocking just uses `"Hello world!"` for every string so it's not that exciting, but at least it means the server is running properly and the schema is being loaded!

![Basic graphiql!](screenshots/default-mock.png)

Now there's a decision to make - should we write a mocked backend and then implement the UI, or wire up the actual backend right now?

<h3 id="day-3">Day 3: Start wiring up real backend</h3>

OK, setting up a client build system is a bit annoying - since we can test our server via GraphiQL, let's just use that and connect our schema to a real database.

First, let's split up our schema into multiple files, which correspond to the different models we need to implement in our DB. We are going to follow some of Jonas' recommendations from [the proposed connector spec on GraphQL-Tools](https://github.com/apollostack/graphql-tools/blob/bc7d106cebbb6b03512df7282d4c91f19edb464e/connectors.md). This means we need a few things for our backend:

1. One model for each GraphQL type we want to fetch: Repository, User, Comment, and Entry
2. One connector for each type of backend we are fetching from: GitHub API, and SQL

Probably we want to start from the connectors, because those are more general. We should probably test them in some way. According to the connector document, we want our connectors to definitely do two things:

1. Batching - when we want to fetch multiple objects of the same type, we should batch them into one request if possible (for example, by running a query with an array of IDs, rather than many queries with one ID each)
2. Caching - if we make multiple requests for the same exact object in one query, we should make sure we only do one request if possible and reuse the result. So if we ask for a particular item from the GitHub API, we don't want to fetch it multiple times per request. In our case, we want to implement one more thing in our GitHub connector: ETags and conditional requests, which will let us check the API for changes without running into our rate limit: https://developer.github.com/v3/#conditional-requests

Let's start with the GitHub connector. I think we basically need just one method on the connector:

```
GitHub.getObject(url);
```

Since the GitHub API doesn't support any batching (we can't get multiple repository objects with one request, for example) we can't take advantage of having any more detailed information than the URL in the connector itself. Converting the object type and ID into the URL is something we can do in the model, which will be type-specific.

Let's go over the functionality we want in this connector:

1. It should not load the same GitHub URL twice in the same request, using something like [DataLoader](https://github.com/facebook/dataloader)
2. It should keep track of `ETag` headers for as many GitHub objects as possible, and send them with requests to both speed up API accesses and avoid hitting the rate limit
3. It doesn't need to send any authentication information other than our API key, because none of the data we are requesting is per-user (although we might want to add this in the future, since the rate limits on the GitHub API are per user)

This means that the connector needs to have both per-request and per-server state - per-request for (1) and per-server for (2). We can make further optimizations later if we end up hitting the GitHub API limit but this seems like a good start.
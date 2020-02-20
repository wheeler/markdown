# An introduction to the Dashboard Performance problem


If you test the frontend of the app locally you may have noticed that loading the initial dashboard after logging in is pretty slow.

You may have also noticed that when you're on some particular page in the Single Page App (SPA), hitting reload to do a hard refresh is quite slow too, even though navigating to that particular page doesn't take long to navigate to within the app is pretty slow.

These are a result of the "dashboard loading". Any page within the main SPA is considered to be within the dashboard. The dashboard is primarily responsible for handling "what company/role is this user looking at". It renders the app's chrome (top bar, nav bar, etc) and the main content.

## About "good" tech debt

When Gusto stared as ZenPayroll, it was a tiny startup with the goal of taking an idea and writing enough software to prove a product-market fit (aka get paying customers). When startups are small this is the most important thing. Perfect architecture and performance are a bad trait - your goal is to enter a rapid growth phase soon and you cannot predict what structure that will demand. At this moment it is often better to focus on product and the customer than the technology. If the product isn't right for the customer's needs pretty soon the technology won't matter. This is the essence of "good tech debt".

In our case an example of this is (with some handwaving) this:
At some point we effectively chose to load all the data we might need to know at the initial loading of the dashboard. Let's say this is the information about the company (including taxes, config, etc) + employees. Loading this up front means never having to answer the question "do we need to load the data we need for ____?". As a small startup that's great - customers don't have lots of data, and it's a large amount of coding overhead we can skip.

The reality of "good tech debt" is at some point is you will either need to pay it off at some point. But there's a curve and you have to figure out the right time. If you do it too early it's closer to never having had the benefit - or you're missing more valuable opportunity to move the next essential piece of product forward.

In my opinion we should have been more aggressively improving dashboard performance well before now. Gusto clearly has a market fit - our customers love us and recommend us and we grow every year. Poor performance is a very difficult item to gauge the customer sentiment. At some point of extreme slowness, yes, customers will complain, but at lower levels customers won't complain but will think less of the product. Think about us - how long does rubymine indexing have to take before we actually try to do work to make it faster. It depends on the operation.

## The dashboard problem

The "dashboard" loads expensive things that it may not need. All company information, all employees (even terminated), lots of constants and environment information, and code to render pages and manage business logic the user may never touch on their visit.

While loading extra code isn't great it is not the focus of this discussion because that comes from a CDN (which means it puts no load on our app servers) and can be cached. People are using codesplitting in our SPA, but I wouldn't encourage anyone to add more right now because when we convert to React Router it will be much easier to do so cleanly.

The main concerns surfaced during Alert 150 was the data heavy Rest fetches that are made on each dashboard load. These are slow, have large amounts of data, and demand both app and DB resources.

Looking at New Relic just now - the 24Hr "most time consuming" web transactions: 5 of the top 11 items are done on the dashboard. Those items total to > 25% of our web worker time. (Note that not all the traffic to these endpoints comes from the dashboard).

Specifically during alert 150 it was noted that for certain very large companies the employee fetch was very expensive. There's also a detail which is we added a "performance" enhancement to the dashboard loading at one point where it breaks the EE fetch into pieces to load them in parallel. The larger the customer the more queries in parallel. It is hard to handle "autoscaling" of workers if one customer suddenly shows up and there are suddenly way more requests and those requests are way more expensive.

## What can we do?

### The real, long term solutions here are: 

1. We want to remove data fetches from the "dashboard" load that are not used to render the outer chrome of the page. Anything needed by the "welcome" screen should be fetched by that page component - anything needed by any other page should be fetched by that page component.
    - This is quite tricky right now because it's very hard to tell what pages might be relying on dashboard data.
    - I did an experiment of just removing the Dashboard fetch-all-EEs and many many "feature" specs fail, indicating pages that just assume EEs are loaded.
    - Only "feature" specs and reading the code can currently expose these - any components built assuming data in the store has a unit spec written pre-populating data in the store.
1. When specific page components need company/employee/payroll information they fetch only what they need to know (or at least the smallest available fetch that will satisfy).
1. We should further reduce the size of expensive fetches by converting to Apollo/GraphQL or sometimes creating reduced rabl views of expensive models or scopes to fetch fewer records.
1. (Further out) These specific pages should be able to rely on information already cached in the frontend (stores or apollo cache). They should know when a refetch is required and/or gracefully handle a background refetch after the cached value is displayed. Doing these things will futher reduce server load and increase the user's perception of app speed.

### Simple stuff you can do now:
- When possible, don't transmit strings and constant content through rest endpoints - write them as constants in your JS so they become part of the code cache not the Rest data
  - For example, Bo recently moved a good chunk of truly app-constant items into a webpack json file.
- Use Apollo/GraphQL for data access. GraphQL naturally allows you to request only what you need and nothing else. This helps reduce the amount of data the backend has to prepare, and reduces the amount of data on the wire to the client.
  - Author GraphQL schema from a frontend perspective. If your page wants to display "what percentage of survey replies were positive" make a GraphQL field with that information. In Rest this is satisfied two ways, each with drawbacks - send all the replies to the frontend and calculate the value there (simple but expensive) - or add the percentage field to the rabl (cheap for this page but means every fetch does this calculation whether or not it wants it).
- Don't add stuff to "God" model rabl (Company Employees Payrolls). If you're adding something you're likely only using it in your one new page - but it will be fetched for every page.
- When using Backbone or *Flux store data on a "page" access this information at the outermost layer. Another way of putting this is that it is best to only read from stores in files we would commonly call `___PageContainer` (aka the thing that is rendered by the Backbone Router). When various inner components read from stores (either off the Backbone globals or through a store import) it gets very hard to answer the question "what is responsible for fetching the data this is reading".
  - This also avoids some problems we have where a page does serial fetches; one data fetch must complete before the inner component tries to render which kicks off another fetch. The page would be rendered more quickly if they were both kicked off at the top.
- If your page reads from a store (Backbone or *Flux) - your page needs to be responsible to make the appropriate fetch.

### Harder stuff you can do now:
- If your page only reads a small amount of information about a company/employee/payroll etc, try converting it to GraphQL
- If you must use Company/Employees/Payrolls - don't assume it is already loaded.

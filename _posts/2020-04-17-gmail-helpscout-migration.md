---
layout: single
title: "Migrate from Gmail to HelpScout with R"
date: 2020-04-17
type: post
published: true
categories: ["R in Production"]
tags: ["helpscout","gmailr","R"]
excerpt: "How R allowed The Carpentries to migrate emails from Gmail to HelpScout using their web APIs"
---

## Preamble

* This is a long and somewhat dense post. Even if you do not have to migrate
  emails from Gmail to HelpScout, I hope this post will be useful to you, as the
  general approach could be interesting to other problems that involve working
  with APIs.
* The full code I actually used for the email migration is available at:
  <https://github.com/carpentries/emailmigration> and I include links pointing
  to functions in the GitHub repo[^1] throughout the post below to illustrate
  my points.


## The problem and its solution

At [The Carpentries](https://carpentries.org), [Regional
Coordinators](https://carpentries.org/regionalcoordinators/) help us organize
workshops across the globe. In the past, each Regional Coordinator was
set up with a Gmail account (through The Carpentries's GSuite plan). However, as
the number of Regional Coordinators grew, and as some geographic areas have more
than one Regional Coordinator, the Gmail account model was starting to cause
some issues. 

The Carpentries Core Team has been using HelpScout for a while and is a much more suitable tool to manage emails and inboxes as a team.

The main challenge with transitioning the Regional Coordinators to using HelpScout was to import the old messages from Gmail to HelpScout. To tackle this problem, I used R and this blog post describes the approach I took.

## Technical overview

Before doing anything else, we used the GSuite data migration tool to transfer
all emails for each Regional Coordinator account into a single account. Having
all the emails to import in the same place makes things easier.

This post goes through the steps I took to perform this migration:

1. Figure out authentication with the Gmail API, and with the HelpScout API
1. Get familiar with the HelpScout API and write R functions to perform the
   tasks needed
1. Convert Gmail threads into HelpScout conversations
1. Test migration on 100 Gmail threads
1. Perform the full migration

Choice of packages and approach:

* Working with the Gmail API is made much easier with the wonderful
  [`gmailr`](https://gmailr.r-lib.org/) package.
* I didn't find an already made package to work with the HelpScout web API so I
  wrote a few functions to interact with the endpoints I needed using the
  [`httr`](https://httr.r-lib.org/) package.
* The mechanics of converting the data coming from the Gmail web API into the
  format needed by the HelpScout API to import the conversation was done using
  the [`R6`](https://r6.r-lib.org/) package. The R6 classes and methods made it
  easier to separate storing each element needed by the HelpScout API as private
  elements and the actual formatting that was handled with methods.
* When working with web APIs a lot can go wrong: there is a weird data
  format that your code didn't know how to handle, your internet connection goes
  down, you reach the rate limit, etc. Therefore, I used the
  [`storr`](https://richfitz.github.io/storr/) package to cache (1) the R6 objects that
  act as the bridge between the 2 APIs; (2) the responses from the HelpScout API
  to make sure all the threads were converted correctly.
* I organized all the code as a barebone package. It makes code management
  easier and is a good habit to take. Here it was a one-off task but if it was
  something that I'd use regularly, it means that I could develop tests, write
  documentation, and enable continuous testing. I could then write and update my
  code, and rely on `devtools::load_all()`.

## 1. Authentication

### 1.1. Gmail API

The instructions in the `gmailr` package's
[README](https://gmailr.r-lib.org/#setup) are clear. You can use the `gm_threads()` function, for instance, to check that the authentication is working as expected.

### 1.2. The HelpScout API

The HelpScout API uses the OAuth 2.0 protocol. The `httr` package handles this well.

Create a new app within HelpScout, and use `https://localhost:1410/` for the redict URL. Take note of the key and secret. Use this information to create a new app object in R with `httr`:

```r
hs_app <- httr::oauth_app(
  "helpscout",
  key = "<your app key here>",
  secret = "<your app secret here>"
)
```

and then use this object to do the authentication online:

```r
hs_token <- httr::oauth2.0_token(
  httr::oauth_endpoint(
    authorize = "https://secure.helpscout.net/authentication/authorizeClientApplication",
    access = "https://api.helpscout.net/v2/oauth2/token"),
  app = hs_app)

htoken <- httr::config(token = hs_token)
```

We can then use the `htoken` object across all our calls to the HelpScout web API.


## 2. Getting started with the HelpScout web API

When working with a new web API, first read the documentation to understand how things are set up. From this initial reading, it became clear that Gmail and HelpScout use different words for related concepts.

HelpScout    | Gmail
-------------|---------
thread       | message
conversation | thread

Keeping this straight in my mind took some time... and because I'm more used to the terms used by Gmail, I used this vocabulary in my function names (for the most part).

Another thing that I needed was HelpScout's internal identifier for the mailbox into which the emails were being imported. So the first function I wrote against HelpScout's API was `hs_mailbox_id()` which returned the internal identifier for the mailbox that was of interest to me.

The second thing I needed to do was to make sure I understood how to use the API to import an actual conversation. I started with fake data I could control to ensure that I had something simple that I knew worked and I could compare against when things didn't work with real data. Even if the documentation of an API is good, there are, more often than not, small details that are not described that you need to figure out. Having this data as a starting point is useful for these tests.

The actual code to create a new ~~thread~~ conversation in HelpScout ended up being:

```r
hs_create_thread <- function(thread, hstoken) {
  body <- jsonlite::toJSON(thread, auto_unbox = TRUE)

  httr::POST(
    "https://api.helpscout.net",
    path = "/v2/conversations",
    body = body,
    htoken,
    httr::content_type("application/json; charset=UTF-8")
  )
}
```

This is not the code I would have written if it was part of a package intended for others to use. For instance, I would have wanted to check the response of the API after each request. But for my particular use case, it made it easier to return this response and inspect manually after the fact once I had confirmed that this code was working for most requests.


## 2. Extracting the content of the emails from Gmail

This was the most time-consuming part as there were lots of unexpected details that came up to get a smooth conversion between the two APIs.

### 2.1. Things that were easy

* The `gmailr::gm_subject()` worked every time to get the subject of the
  threads for all the messages.

### 2.2. Things that were almost easy

* [Extracting the people involved in the conversation](https://github.com/carpentries/emailmigration/blob/master/R/gmail.R#L128-L150). The `gmailr::gm_to()` and
  `gmailr::gm_from()` worked well to extract the email addresses. The small
  catch was that some email addresses were formatted as `FirstName LastName
  <email@address.rr>`, others had only `email@address.rr`, and when multiple
  people were involved a comma separated them. However, in some cases, people
  have a comma in their names.
* [Extracting the date](https://github.com/carpentries/emailmigration/blob/master/R/gmail.R#L121-L126). The `gmailr::date()` returns the date from the email in
  [Unix time](https://en.wikipedia.org/wiki/Unix_time). The `anytime`
  [package](https://cran.r-project.org/web/packages/anytime/index.html) is
  useful at converting Unix time into other formats, including the `iso8601`
  that was expected by the HelpScout API. I still had to manually add a final
  `Z` to the character string.

### 2.3. Things that were not so easy

* [Extracting the email attachments](https://github.com/carpentries/emailmigration/blob/master/R/gmail.R#L14-L24). The attachments themselves are not returned
  by the API. Instead, the API returns an URL that points to the address where
  the attachments can be retrieved. The HelpScout's API accepts the attachments
  as [base64-encoded](https://en.wikipedia.org/wiki/Base64) strings. The
  `gmailr` package helped to retrieve this data, but the data returned by the
  Gmail API is base64url encoded. Thankfully, converting to regular
  base64 is a short regular expression substitution away once you know the
  difference between the two.
* The thing that was the most puzzling was parsing the actual body of the
  emails. The `gmailr::gm_body()` worked for only a small fraction of the emails
  I had to deal with. After many trials and errors, [I wrote a function](https://github.com/carpentries/emailmigration/blob/master/R/gmail.R#L48) to
  reliably retrieve the content of the emails[^2]. There were many situations to deal with as the messages can be:
  - "multipart" the body of the email is provided both in plain text format or
     in HTML format which allows for email clients that don't support
     HTML-formatting to provide the plain text version of the message;
  - either only plain text or in HTML format
  - provided as attachments (what some email clients do when you forward a
    message).

  Depending on the situation, the location of the body of the email within the
  deeply nested list that was returned by the Gmail API could vary. I ended up
  writing a recursive algorithm that traversed the list to find and retrieve the
  relevant content of the emails.

  The last catch was that plain text messages that included an URL were
  interpreted by the HelpScout API as being HTML-formatted. It meant that the
  whitespace to indicate the line breaks were ignored making the body of the
  messages large blocks of texts that were very hard to read and follow. I
  relied on the `commonmark::markdown_html()` to [convert these plain text
  messages](https://github.com/carpentries/emailmigration/blob/master/R/gmail.R#L1-L6) into HTML that then looked good once they were uploaded onto
  HelpScout using the API.

## 3. Conversion between Gmail and HelpScout

Now that I had access to all the relevant information from the emails, I needed to format it so it could be imported by the HelpScout API. For this, I used the R6 object-oriented programming system.

Each element coming from the Gmail API was individually stored as a private field, and an accessor method (`$get()`) created the list in the format needed to be ingested by HelpScout's API.

I used 3 classes for this:

* [one for the HelpScout conversations](https://github.com/carpentries/emailmigration/blob/master/R/HelpScout-classes.R#L85) (the Gmail threads)
* [one for the HelpScout threads](https://github.com/carpentries/emailmigration/blob/master/R/HelpScout-classes.R#L30) (the Gmail messages)
* [one for the attachments](https://github.com/carpentries/emailmigration/blob/master/R/HelpScout-classes.R#L6)

This modularity helped debugging and limited the complexity of each class.

Because all the emails are going to be in the same inbox in HelpScout, I wanted an easy way to tag the conversations based on the team of Regional Coordinators that were involved. The R6 system was useful for this because once the email information was stored within the object, I could use a private method called by the accessor to extract all the people involved, and add tags in HelpScout to help Regional Coordinators find past conversations that are relevant to them.

It was one of the first times I used R6[^3] for a real task and I could see its potential. If the code written here were for public consumption, it would have provided a good framework to add more tests on the data structure of the individual elements that were coming from the Gmail API to ensure that the output from the accessor method was always formatted correctly before trying to convert it in the format required by HelpScout's API.



## 4. Caching

My previous experience working with web APIs have taught me that things can go wrong, and it is always a good idea to keep track (on disk and not only on memory) of the requests that have been tried and the ones that have not, and the requests that succeeded and the ones that failed. Especially, when your scripts do thousands of API calls, you don't want to have to run everything again once your script fails because your internet connection goes down for a short while, or the data is not formatted properly because you are dealing with an edge case.

For this, I use the [`storr` package](https://richfitz.github.io/storr/) and its functionality to rely on hooks to retrieve external data. `storr` is a key-value store. It is not that different than using variable names to store objects in memory as you normally do in your R terminal:

```r
## setting a variable
cat_name <- "Felix"

## getting the content of the variable
cat_name
```

When using a `storr` store:

```r
## defining the storr
st <- storr::storr_rds(path = "cache")

## setting a variable
st$set("cat_name", "Felix")

## getting the variable name
st$get("cat_name")
```

The difference is that `storr` provides different backends for storing your object and, if like in this example, you use `storr_rds`, your objects are stored as `rds` files on your disk and are available beyond your current R session. How does that help with the problem here?

A great feature of `storr` is that you can set up your store to call a function to create the object instead of providing it directly with `$set()`.

It means that you store the content of a variable, your key into the store, and you can retrieve it:

```r
## the hook function
fetch_hook_random_cat_name <- function(key, namespace) {
  sample(c("Felix", "Garfield", "Tigger", "Mowgli"), 1)
}

## defining the storr
st <- storr::storr_external(
  storr::driver_rds(path = "cache"),
  fetch_hook_random_cat_name
)

## the first time you call a key, it will run the hook function
st$get("cat_name")

## subsenquently, it will return the value stored in the store
st$get("cat_name")
```

The hook function always takes the two arguments `key` and `namespace` but they don't need to be used in the body of the function as in the example above.

We can extend this approach to store the output of time-consuming computations or the results of API calls[^4]. For instance, here, I created [a store](https://github.com/carpentries/emailmigration/blob/master/R/caching.R#L7) to keep the output of the function `convert_gmail_thread()`, and used `get_gmail_thread()` as a wrapper to access the store.

```r
fetch_hook_gmail_threads <- function(key, namespace) {
  convert_gmail_thread(key)
}

store_gmail_threads <- function(path = "cache/threads") {
  storr::storr_external(
    storr::driver_rds(path),
    fetch_hook_gmail_threads
  )
}

get_gmail_thread <- function(thread_id, namespace) {
  store_gmail_threads()$get(thread_id, namespace)
}
```

When calling `get_gmail_thread()`, using a `thread_id` that had not been retrieved using the Gmail API before, the function `convert_gmail_thread()` will be called, getting all the information needed for this particular thread, and storing it in an R6-class object. If another part of the script fails, we do not need to redo the calls to the Gmail API, instead the cached copy within the store will be retrieved.

I used a similar approach to [store the responses from the HelpScout API](https://github.com/carpentries/emailmigration/blob/master/R/caching.R#L71), and wrapped at the same time the call to the `get_gmail_thread()` function above. A slightly simplified version of what I used is:

```r
fetch_hook_hs_response <- function(key, namespace) {
  res <- get_gmail_thread(key, namespace)
  hs_create_thread(res$get(), htoken)
}

store_hs_responses <- function(path = "cache/hs_responses") {
  storr::storr_external(
    storr::driver_rds(path),
    fetch_hook_hs_response
  )
}

get_hs_response <- function(thread_id, namespace) {
  store_hs_responses()$get(thread_id, namespace)
}

```

So, what's happening here? I use the Gmail thread ID as a single point of entry for the entire script (retrieve the thread from the Gmail API, convert it to the format expected by the HelpScout API, upload the thread to HelpScout). Depending on whether the queries have already been made and stored in the cache, the script will retrieve the data from the API or the objects stored on disk in the cache.

What does the `namespace` argument do? Using namespacing in `storr` allows you to organize your objects in your store. Especially, it allows you to have objects with the same name but with different values. Here, I planned to use namespaces to keep track of my different attempts. If the first attempt would have failed for some threads, I could fix the problem in the code, and re-attempt the HelpScout API calls just for the ones that failed under a different namespace.

## 5. Putting it all together

Once I had most of the pieces together, I started by testing the code on the first 100 threads (as it's the default number of threads that `gmailr` returns). That was a manageable number to see how the script behaved while being large enough that many different types of messages would be encountered. At that time, I didn't use the caching system.

Once the first 100 threads could be imported successfully in HelpScout, I wrote a function to retrieve the identifiers for all the threads in the inbox that needed to be imported, and iterated on these identifiers to call the `get_hs_response` function:

```r
get_all_threads <- function() {
  
  first_it <- gm_threads()
  next_token <- first_it[[1]]$nextPageToken
  
  res <- append(list(), first_it)
  
  while (length(next_token) > 0) {
    tmp <- gm_threads(page_token = next_token)
    res <- append(
      res, tmp
    )
    next_token <- tmp[[1]]$nextPageToken
    message("next token: ", next_token)
  }
  
  res
}

threads <- get_all_threads()

threads_ids <- purrr::map(
  threads,
  ~ purrr::map_chr(.$threads, ~ .$id)
) %>%
  unlist()

hs_res <- purrr::walk(
  threads_ids,
  ~ get_hs_response(., namespace = "v2020-04-10.1")
)
```

As part of the hook function that takes care of uploading conversations to HelpScout, I check whether the upload was successful and based on that I created and assigned a Gmail label to the thread. This was an additional safeguard that I could use to flag threads that didn't import successfully.

Once the upload completed, I could then inspect the content of the store:

```r
## Retrieve the threads_ids from the store
idx <- store_hs_responses()$list(namespace = "v2020-04-10.1")

## Retrieve the status code for the HelpScout API responses
is_error <- purrr::map_lgl(
  idx,
  ~ httr::status_code(
    store_hs_responses()$get(., namespace = "v2020-04-10.1")
  ) >=  400
)

## How many calls failed?
sum(is_error)

## Which thread_ids failed?
idx[is_error]
```

and double check that it was the same threads that were labeled with `failure-<namespace>` in Gmail.

## Lessons learned

As often with using programming to solve problems, what might seem like a simple task: "Transfering emails from one system to an other" is a collection of small problems. Being able to break down the big problems into small ones, and knowing how to address them comes with experience. Experience will help you recognize problems similar to some you have already solved, and reflecting on these past experiences will help you identify the algorithms, packages, and general code organization that are most likely to help you solve your problem.

In The Carpentries Instructor Training, when [we teach about expertise](https://carpentries.github.io/instructor-training/03-expertise/index.html), we talk about how the mental model of experts is denser and more connected. These features make it more difficult for experts to teach beginners because they have forgotten what it is like to not know how to break down a large problem into multiple small ones. The problem here is not just "migrate a bunch of emails between two systems", there is a lot more to it. I wrote this blog post with the intent to demonstrate the approach I took to break down a problem into small ones and, in the process, describe the tools and techniques I chose to address them.

Expertise is subjective and relative, and I certainly do not claim that the approach I chose here is the best, the most efficient or the most elegant. There is certainly room for improvement. For instance, parts of the code could be re-factored to make it more organized, parts could be rewritten to be more [defensive](https://en.wikipedia.org/wiki/Defensive_programming), and there is no documentation (besides this blog post) and barely any comments.

I am interested in hearing your perspective and thoughts on how the problem could have been approached differently and the tools you would have chosen to address it. If this post was useful to you to help you solve a different problem, I would also love to hear about it! Leave a comment below or contact me using the info provided on the left of this page.


### Footnotes

[^1]: You may notice that the Git history for the repo includes the key and secret for the HelpScout OAuth authentication. By themselves, these are not enough to access any data, as you also need to authenticate with a valid HelpScout account within our organization. These credentials have also been revoked.
[^2]: I'll be submitting a pull request to `gmailr` soon.
[^3]: If you are interested in learning more about the object-oriented programming R6 system, the [chapter about it](https://adv-r.hadley.nz/r6.html) in the "Advanced R" book by Hadley Wickham is a great place to start.
[^4]: If you are interested in learning more about `storr`, read the [documentation for the package](https://richfitz.github.io/storr/articles/storr.html) and the [vignette on external data](https://richfitz.github.io/storr/articles/external.html) that initially helped me get started with this amazingly useful package.

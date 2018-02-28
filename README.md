# The Daily XHub WTF

## Thursday Feb 27: Real programmers write in binary

I was investigating `/user/state` saving, where a simple
JSON object was sent to the server.  Sometimes the JSON
was as follows:

```js
{ [projectId + "-jira-reminder"]: "true" }
```

Somewhat odd to save a key-value pair when you're really saving
just a single ID.  Backend guys told me that the API allows
frontend to save several JIRA-reminders at once, but of course
the frontend was only ever sending one.  And why is the value
`"true"` not `true`?

At other times though, the contents of the user state was a
tutorials field:

```js
{ tutorials: newTutorials.join("") }
```

But what's inside that string? A few lines earlier something odd
is going on:

```js
const newTutorials = tutorials.split("");
_.forEach(views, view => newTutorials[view] = "1");
```

So we split up the old tutorials into characters, and set some
of these characters to 1. Hmm... luckily we have some tests
that contain the expected data for tutorials:

```
Given response "emptyObject" to "PUT /rest/user/state" with body
    """
        { "tutorials": "00110" }
    """
```

## Tuesday Feb 20: How many spans could there be?

After adding an additional statistics bubble for request node,
suddenly a test failed:

```
Scenario: Flatten stack trace button
    ...
    And I see "StandardHostValve.invoke" as first subnode
```

This test checked which node appears as a first node under
the request node.  So why does this fail?  After all, I only
changed the minor things about request node itself.  So
taking a look at the code behind this test scenario:

```js
this.Then(/^I see "([^"]*)" as first subnode$/, title => {
    const selector = "[data-test-node-title] span";

    browser.waitForExist(selector);
    expect(browser.getText(selector)[6]).toContain(title);
});
```

The selector `[data-test-node-title]` kinda selects the
title part of the request node (though it also matches the
titles of all other nodes).  But the span... or rather the
sixth span... Magic numbers in all their beauty.

Well. It's a great example of how to write a really brittle
web-page test: count the number of spans up to the span
that interests you.

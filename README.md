# The Daily XHub WTF

## Tuesday Feb 20

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


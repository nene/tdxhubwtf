# The Daily XHub WTF

## Friday Apr 6: JavaScript needs no classes

Functional programming is all the craze these days.
You don't need OOP, when you can do functional.
If you really insist on having a method,
you can attach one to an object:

```js
dataRange.isVersionsSelected = isVersionsSelected.bind(dataRange);
```

And then you could even destructure that function
out of the object and call it separately:

```js
function paramsFromDataRange({ start, end, prevStart, prevEnd, isVersionsSelected }) {
    if (isVersionsSelected()) {
        return {
            // Parameters for version-range
        };
    }

    return {
        // Parameters for period range
    };
}
```

Or when you're not sure that your object has that method,
you can take measures that your code it still works:

```js
if (isVersionsSelected.call(action.dataRange)) {
    ...
}
```

Only a Java-minded-OOP-maniac would think of replacing all these
conditional branches sprinkled throughout the codebase with
two classes where one implements a range of versions,
and another that implements range of dates:

```js
class VersionRange {
    urlParams() { ... }
}

class PeriodRange {
    urlParams() { ... }
}
```

When you leave out React components, the whole XHub codebase
contains just three classes. Everything else is contained in
nice and pretty functions. These functions might be several
hundred lines long, but they're functions.
Functional programming FTW!

## Friday Mar 23: Delete it! 4real!

Today I stumbled upon the following wonderful snippet:

```js
item.exceptions = undefined;
delete item.exceptions;
```

I thought: somebody must have first implemented the removal
of the field by setting it undefined, and later somebody else
decided to completely delete the field, without bothering to
remove the first line.

Then I looked at the change log, and saw that these two lines
were added explicitly in a commit titled "performance optimizations".

## Tuesday Mar 21: React as a data-structure

I was investigating how our `DurationTab` component works:

```js
<AbstractTab
    tabId="duration"
    title="Slow Requests"
    issues={ issues }
>
    <TabColumn type="entryPoint" name="ENTRY POINT">
        { this.renderEntryPointColumn }
    </TabColumn>
    <TabColumn type="scope" name="SCOPE">
        { this.renderScopeColumn }
    </TabColumn>
</AbstractTab>
```

Besides the odd thing that we're passing a function
to the TabColumn through children property, this looks pretty nice.

I wonder how this TabColumn works:

```js
export default class TabColumn extends PureComponent {
    render() {
        return this.props.children;
    }
}
```

Oh... so it ignores all the props besides `children`
(which is a function, as you remembered earlier).
But the code definitely relies on these props being
passed to TabColumn.

So, let's take a look inside AbstractTab itself, where
among other things we find:

```js
<tbody>
    { visibleItems.map((row, i) => {
        // ... SKIP ...
        return (
            <Row
                key={ row.id }
                row={ row }
                renderers={ children }
            />
        );
    } ) }
</tbody>
```

So the TabColumn elements get passed to Row component,
which loops over them and uses the function from children
as a constructor of a react component:

```js
<tr className="application-tab-data-row">
    { React.Children.map(renderers, element => {
        const { type, children: RowRenderer } = element.props;

        return <RowRenderer key={ type } { ...this.props } />;
    }) }
</tr>
```

And one of these render-functions
(that was referenced in the top-most code-snippet)
is defined like so:

```js
class DurationTab extends PureComponent {
    // ... snip ...

    renderScopeColumn = ({ row }) => {
        const { issues } = this.props;

        return (
            <td className="application-tab-column">
                { issues.thresholds.scope > row.lastDurationStats.scope ? (
                    <span>outlier</span>
                ) : (
                    <span><PrettyNumber value={ row.lastDurationStats.scope } decimal={ 1 } />%</span>
                ) }
            </td>
        );
    }
}
```

On one hand this function is used as a React functional component.
So all its props should come through parameters.
But it also makes use of `this.props`. How come?

Turns out that this code is making use of proposed ECMAScript feature
for defining class instance variables. Effectively the above code
translates to:

```js
constructor() {
    this.renderScopeColumn = ({ row }) => { ... };
}
```

What the original programmer really wanted to do, was:

```js
<AbstractTab
    tabId="duration"
    title="Slow Requests"
    issues={ issues }
    columns={ [
        {type: "entryPoint", name: "ENTRY POINT", render: this.renderEntryPointColumn.bind(this)},
        {type: "scope", name: "SCOPE", render: this.renderScopeColumn.bind(this)},
    ] }
/>
```

But clearly, that would not have been Reacty enough.

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

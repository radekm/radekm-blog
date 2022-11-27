---
title: "Programmatically clean up tabs in Firefox"
date: 2021-02-08
tags: ["programming", "firefox"]
---
How to use Firefox API for extensions to find and close unwanted tabs.

Tabs in Firefox can be manipulated via `browser.tabs` collection.
This collection is available only for extensions.
So open `about:debugging` choose `This Firefox` and from the list of extensions
`Inspect` one which has permission `tabs`. I chose `uBlock Origin` ad blocker
since it needs lots of permissions, so I assumed it has `tabs` permission.

## Saving open tabs into a variable

First we save a list of open tabs into a variable `tabs`:

```javascript
var tabs = []
```

I'm using `var` since `let` prevents redeclaration which makes copy/paste annoying.
The following function saves current tabs into `tabs` variable:

```javascript
function refreshTabs() {
    browser.tabs.query({currentWindow: true}).then(result => {
        tabs = result
        console.info("Tabs from current window saved into variable tabs")
    })
}
```

We have to call it now and whenever currently open tabs change.

## Close duplicate tabs

To find duplicates we will group tabs by URL:

```javascript
function groupTabsBy(tabs, f) {
    const grouped = new Map()

    for (const tab of tabs) {
        const key = f(tab)
        if (!grouped.has(key))
            grouped.set(key, [])

        grouped.get(key).push(tab)
    }

    return grouped
}
```

After the tabs are grouped in a map, we're only interested in groups with multiple tabs.
So we need to filter the map:

```javascript
function filterMapByValues(map, f) {
    const filtered = new Map()

    for (let [k, v] of map) {
        if (f(v))
            filtered.set(k, v)
    }

    return filtered
}
```

We end up with `URL -> group` map where each group contains at least 2 tabs:

```javascript
var dupesByUrl = filterMapByValues(groupTabsBy(tabs, tab => tab.url),
    tabs => tabs.length > 1)
```

Now from each group we have to close all tabs except one:

```javascript
function closeDupes(dupeMap) {
    for (let [k, tabs] of dupeMap) {
        const ids = tabs.slice(1).map(tab => tab.id)
        console.info("Removing", ids.length, "duplicates for key", k)
        ids.forEach(id => browser.tabs.remove(id))
    }
}

closeDupes(dupesByUrl)
```

Finding duplicates tabs by title is analogous:

```javascript
var dupesByTitle = filterMapByValues(groupTabsBy(tabs, tab => tab.title),
    tabs => tabs.length > 1)
```

Before closing these we should be little careful since two different (non-duplicate)
tabs may share the same title -- especially if the title is not very distinctive.

## Close tabs by URL or by title

When researching a topic I usually open many tabs on a single domain
or with a certain keyword in their title. So here is a code
which can find and close these.

By URL:

```javascript
tabs.filter(tab => tab.url.includes("//mail.google.com/")).forEach(tab => {
    console.log(tab.title, tab.url)
    // Uncomment to close.
    //browser.tabs.remove(tab.id)
})
```

By title:

```javascript
tabs.filter(tab => tab.title.toLowerCase().includes("google")).forEach(tab => {
    console.log(tab.title, tab.url)
    // Uncomment to close.
    //browser.tabs.remove(tab.id)
})
```

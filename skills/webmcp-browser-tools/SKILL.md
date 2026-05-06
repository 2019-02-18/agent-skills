---
name: webmcp-browser-tools
description: Discover, inspect, and execute WebMCP tools registered on the current browser page. Use when the user mentions WebMCP, navigator.modelContext, browser-native AI tools, or wants to interact with tools exposed by a web page through the Web Model Context Protocol.
---

# WebMCP Browser Tools

## Overview

WebMCP (Web Model Context Protocol) is a browser-native API that allows web pages to expose JavaScript-based tools to AI agents via `navigator.modelContext`. Pages that register WebMCP tools can be thought of as MCP servers running client-side.

This skill guides the agent through detecting, listing, inspecting, and invoking WebMCP tools on the current page.

## Prerequisites

WebMCP is experimental as of 2026. It requires:
- Chrome with the flag `chrome://flags/#enable-webmcp-testing` set to **Enabled**
- The page must have registered tools via `navigator.modelContext.registerTool()`
- Optional: WebMCP DevTools extension for enhanced visibility

## Detecting WebMCP Support

Before discovering tools, verify the page supports WebMCP by checking for `navigator.modelContext`:

```javascript
// In browser console or via evaluate()
typeof navigator.modelContext !== 'undefined'
```

If undefined, WebMCP is not available. Inform the user they may need to enable the Chrome flag or visit a different page.

## Listing Registered Tools

Use JavaScript evaluation in the browser to enumerate tools exposed by the page.

### Method 1: Direct ModelContext inspection

```javascript
// Attempt to access the internal tool map or any exposed listing method
const modelContext = navigator.modelContext;

// Some implementations expose a tools map or listTools method
const tools = await modelContext.listTools?.() 
  ?? (modelContext.tools ? Array.from(modelContext.tools.values()) : [])
  ?? [];

JSON.stringify(tools.map(t => ({
  name: t.name,
  title: t.title,
  description: t.description
})));
```

### Method 2: DOM observation (fallback)

If the page uses Declarative WebMCP (`<form toolname="...">`), scan the DOM:

```javascript
Array.from(document.querySelectorAll('form[toolname]')).map(form => ({
  name: form.getAttribute('toolname'),
  title: form.getAttribute('tooltitle') || form.getAttribute('toolname'),
  description: form.getAttribute('tooldescription') || ''
}));
```

## Inspecting Tool Details

For each discovered tool, extract its complete definition including the input schema:

```javascript
const toolName = 'code_read'; // example tool name
const modelContext = navigator.modelContext;

// Retrieve full tool definition if accessible
const toolDef = modelContext.tools?.get(toolName);

JSON.stringify({
  name: toolDef.name,
  title: toolDef.title,
  description: toolDef.description,
  inputSchema: toolDef.inputSchema,
  readOnlyHint: toolDef.annotations?.readOnlyHint ?? false,
  untrustedContentHint: toolDef.annotations?.untrustedContentHint ?? false
}, null, 2);
```

The `inputSchema` is a JSON Schema string describing the parameters the tool accepts.

## Executing a Tool

Tools are executed by providing a valid JSON object matching the tool's `inputSchema`. Use the browser's JavaScript execution capability:

```javascript
const modelContext = navigator.modelContext;

// Example: executing a tool imperatively registered
const result = await modelContext.tools.get('code_write').execute({
  path: 'index.html',
  content: '<h1>Updated</h1>'
});

JSON.stringify(result);
```

### Execution via Declarative Forms

For declarative tools (`<form toolname="...">`), fill and submit the form:

```javascript
const form = document.querySelector('form[toolname="search"]');
form.querySelector('input[name="query"]').value = 'WebMCP';
form.requestSubmit();
```

## Tool Definition Reference

A `ModelContextTool` registered imperatively has the following structure:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | 1-128 chars. ASCII alphanumerics, `_`, `-`, `.` |
| `title` | string or null | Human-readable title for UIs |
| `description` | string | Natural language description |
| `inputSchema` | object | JSON Schema for tool parameters |
| `annotations.readOnlyHint` | boolean | True if tool does not modify state |
| `annotations.untrustedContentHint` | boolean | True if tool handles untrusted content |

## Workflow

When the user wants to interact with WebMCP tools on a page:

1. **Detect**: Evaluate `navigator.modelContext` exists
2. **List**: Enumerate all registered tools with names and descriptions
3. **Inspect**: For the tool(s) of interest, retrieve full schema and hints
4. **Execute**: Build valid arguments from the schema and invoke the tool
5. **Report**: Present the execution result to the user

## Security Considerations

- `untrustedContentHint` indicates the tool processes untrusted input; validate arguments carefully
- Respect `readOnlyHint` but do not rely on it for safety
- Tools execute in the page's origin with its permissions
- User confirmation is advisable before executing state-mutating tools

## Resources

- W3C Community Draft: https://webmachinelearning.github.io/webmcp
- WebMCP DevTools (Chrome Extension): https://github.com/2019-02-18/WebMCP-DevTools
- Chrome flag: `chrome://flags/#enable-webmcp-testing`

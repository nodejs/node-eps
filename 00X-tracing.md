| Title  | JS Tracing APIs |
|--------|-----------------|
| Author | @jasongin       |
| Status | DRAFT           |
| Date   | 2017-01-12      |

## 1. Overview
This document proposes adding new JavaScript APIs for high-performance tracing, built on top of
the native trace event framework from Chromium that is already integrated into the Node engine.

There are three major parts to this proposal:
  * Tracing emitter APIs
  * Tracing listener APIs
  * Console tracing API extensions

The listener APIs and console API extensions are optional: either or both of those could
conceivably be rejected while still accepting the other part(s).

### 1.1 Justification
There are benefits to having tracing APIs built-in to the Node platform, compared to implementation
as an external module:
 * **Holistic view** - By leveraging the same tracing system across all layers of the stack
   (v8, Node, libraries, and applications), diagnostic tools can offer a complete and
   integrated view of the system.
 * **Standardization** - If high-performance tracing APIs are built-in to Node core, then popular
   third-party libraries are much more likely to instrument their code using the the APIs. As the
   ecosystem standardizes on a common system for tracing, applications developers benefit from
   improved diagnostic capabilities.
 * **Performance** - High performance is generally achieved by doing as much of the tracing work
   as possible in native code and off of the main thread. While much of that could be accomplished
   using an external native module, there may be some additional optimizations that can be done
   only through integration with the GC and/or JIT.

### 1.2 Background
The [Node.js Diagnostics WG](https://github.com/nodejs/diagnostics) became aware of a
high-performance tracing framework being integrated from Chromium into v8. They agreed that the
Node.js engine could also benefit from using the same tracing framework. See the
[meeting notes from initial tracing discussion](https://github.com/nodejs/diagnostics/blob/master/wg-meetings/2016-06-01.md)
and the [related WG issue](https://github.com/nodejs/diagnostics/issues/53).

Integration of the C++ trace event framework into Node was
[completed recently](https://github.com/nodejs/node/pull/9304). It can be
[enabled at runtime via node command-line options](https://github.com/nodejs/node/blob/master/doc/api/tracing.md).

Note the Node engine (C++ code) is not yet *instrumented* with calls to the tracing macros. That
work is planned, but is outside the scope of this proposal.

## 2. Tracing emitter APIs
The tracing emitter APIs enable JavaScript code to emit tracing events of any of the supported
types along with associated event metadata. The APIs are primarily intended to be used in the
following scenarios:
 * Instrumentation of application code, enabling application developers to collect performance
   and diagnostic tracing data about their application code.
 * Instrumentation of library code, enabling developers of applications consuming the library
   to collect performance and diagnostic tracing data about the library, either separate from or
   combined with application and engine tracing.

The tracing emitter APIs might also be used as a low-level output format for higher-level tracing
or logging frameworks. For example, logging frameworks that enable pluggable "sinks" for log events
may support Node.js tracing as another sink option. However, since these APIs are primarily
designed for performance tracing, they are not optimized for logging scenarios. For example,
they don't define any log-level metadata or filtering based on it.

### 2.1 New core module
The proposed new tracing emitter APIs (and listener APIs) belong in a new Node.js core module:
```javascript
const tracing = require('tracing');
```
The "tracing" module name is [already reserved](https://www.npmjs.com/package/tracing) for this
purpose. Alternatively, "trace" might be a good name, however that would have to be reconciled
against the existing ["trace" npm package that generates long stack traces using AsyncWrap](https://www.npmjs.com/package/trace).

### 2.2 Example use
```javascript
// Enable recording for the 'example' category. (Can also enable via node command-line.)
const category = 'example';
tracing.enableRecording(category);

tracing.emit(category, { eventType: 'instant', name: 'myEvent' });

const scopeId = tracing.emit(category, { eventType: 'begin', name: 'myTask' });
tracing.emit(category, { eventType: 'end', name: 'myTask', id: scopeId });

tracing.emit(category, { eventType: 'count', name: 'myCounter', value: 3 });
```

### 2.3 Emitter API specification
APIs are specified in TypeScript syntax for clarity in this doc. Actual implementation is of course
plain JS.

```typescript
/**
 * Defines the valid string values for the eventType property of a TracingEvent object.
 */
type TracingEventType = "instant" | "begin" | "end" | "count";

interface TracingEvent {
  /**
   * Required type of event; must be one of the allowed event type string values.
   */
  eventType: TracingEventType;

  /**
   * Event category or category list. May be omitted if the TracingEvent object is passed to
   * Tracing.emit() where the category is specified as a separate parameter.
   */
  category?: string | string[];

  /**
   * Name of the event (not necessarily unique). Required for all event types.
   */
  name: string;

  /**
   * Integer identifier that can be used to correlate paired "begin" and "end" events or multiple
   * "count" events and distinguish them from other timings or counters that use the same name.
   * For a "begin" event a scope identifier is auto-generated and must be supplied again with
   * the corresponding "end" event. For "count" events a counter identifier is optional. This
   * property is not used with "instant" events.
   */
  id?: number;

  /**
   * Required counter value for "count" events; not used for other event types. This must be either
   * a number, a dictionary of name-number pairs, or a function that returns one of those. If a
   * function, it will be invoked to retrieve the value(s) only if tracing is enabled for the
   * category. (The tracing system currently requires multi-value counters to have exactly two
   * values.)
   */
  value?: number | { [name: string]: number } | () => (number | { [name: string]: number });

  /**
   * Optional arguments for "begin", "end", and "instant" events. If specified, this must be a
   * either a dictionary of name-value pairs or a function that returns a dictionary. If a
   * function, it will be invoked to retrieve the args only if tracing is enabled for the category.
   * Not used for "count" events.
   */
  args?: { [name: string]: any } | () => { [name: string]: any };
}

interface Tracing {
  /**
   * Checks whether tracing is enabled for a category or for any categories in an array. Tracing
   * is enabled for a category if either recording is enabled for the category or there are one
   * or more listeners for the category.
   *
   * @param category Required tracing category name or array of category names.
   * @returns True if tracing is enabled for the category or for any of the categories in the
   *   array.
   */
  isEnabled(category: string | string[]): boolean;

  /**
   * Gets a list of tracing categories that are enabled for recording.
   */
  recordingCategories(): string[];

  /**
   * Enables or disables recording for a tracing category or categories. Tracing event recording
   * is automatically started when one or more categories are enabled via this method or via the
   * node command-line. Recording is automatically stopped when it is disabled for all
   * categories.
   * *
   * @param category Required tracing category name or array of category names.
   * @param enable Optional value specifying wiether recording will be enabled for the specified
   *   category or categories. Defaults to true.
   */
  enableRecording(category: string | string[], enable?: boolean): void;

  /**
   * Emits a tracing event using an event object.
   *
   * @param e The event to be emitted. The event object must specify one or more categories.
   * @returns Nonzero event id (either provided or generated) if the event was emitted, or null if
   *   the event was not emitted because tracing was not enabled for any of the event categories.
   */
  emit(e: TracingEvent): number | null;

  /**
   * Emits a tracing event using an event object, with categories specified separately.
   *
   * @param category Required tracing category name or array of one or more category names for
   *   the tracing event. Overrides any categories specified in the event object.
   * @param e The event to be emitted.
   * @returns Nonzero event id (either provided or generated) if the event was emitted, or null if
   *   the event was not emitted because tracing was not enabled for any of the event categories.
   */
  emit(category: string | string[], e: TracingEvent): number | null;
}
```

### 2.4 Event-type-specific emit APIs
The `emit()` method defined above mostly follows the familiar `EventEmitter` pattern for emitting
events as objects. However some alternative methods are proposed for each type of event, that take
separate parameters instead of an event object. Using these methods, the four emitted events in the
previous code example can be rewritten more concisely:

```javascript
tracing.instant(category, 'myEvent');

const scopeId = tracing.begin(category, 'myTask');
tracing.end(category, 'myTask', scopeId);

tracing.count(category, 'myCounter', 3);
```

These methods are slightly more efficient because they doesn't require creation of an object for
the event. And for simple cases they are easier to write and read.

```typescript
interface Tracing {
  /**
   * Emits a tracing event of type "instant" using an event name and optional event args.
   *
   * @param category Required tracing category name or array of one or more category names for
   *   the tracing event.
   * @param name Required name for the event.
   * @param args Optional dictionary of name-value pairs, or a function that returns a dictionary.
   *   If a function, it will be invoked to retrieve the args only if tracing is enabled for the
   *   category.
   * @returns True if the event was emitted; false if the event was not emitted because tracing
   *   was not enabled for any of the event categories.
   */
  instant(
    category: string | string[],
    name: string,
    args?: { [name: string]: any } | () => { [name: string]: any }): boolean;

  /**
   * Emits a tracing event of type "begin" using an event name and optional args.
   *
   * @param category Required tracing category name or array of one or more category names for
   *   the tracing event.
   * @param name Required name for the event.
   * @param args Optional dictionary of name-value pairs, or a function that returns a dictionary.
   *   If a function, it will be invoked to retrieve the args only if tracing is enabled for the
   *   category.
   * @returns Nonzero generated scope id if the event was emitted, or null if the event was not
   *   emitted because tracing was not enabled for any of the event categories.
   */
  begin(
    category: string | string[],
    name: string,
    args?: { [name: string]: any } | () => { [name: string]: any }): number | null;

  /**
   * Emits a tracing event of type "end" using an event name, sope identifier, and optional args.
   *
   * @param category Required tracing category name or array of one or more category names for
   *   the tracing event.
   * @param name Required name for the event.
   * @param id Required integer scope identifier that can be used to correlate paired "begin"
   *   and "end" events and distinguish them from other events with the same name. This must be
   *   the same as a scope id that was generated by a call to the "begin" method. This scope id
   *   may be null if the "begin" call did not emit an event; in that case this call is also a
   *   no-op.
   * @param args Optional dictionary of name-value pairs, or a function that returns a dictionary.
   *   If a function, it will be invoked to retrieve the args only if tracing is enabled for the
   *   category.
   * @returns True if the event was emitted; false if the event was not emitted because tracing
   *   was not enabled for any of the event categories.
   */
  end(
    category: string | string[],
    name: string,
    id: number | null,
    args?: { [name: string]: any } | () => { [name: string]: any }): boolean;

  /**
   * Emits a tracing event of type "count" using an event name and optional id and args.
   *
   * @param category Required tracing category name or array of one or more category names for
   *   the tracing event.
   * @param name Required name for the event.
   * @param value Required counter value, either a number, a dictionary of name-number pairs, or a
   * function that returns one of those. If a function, it will be invoked to retrieve the value(s)
   * only if tracing is enabled for the category. (The tracing system currently requires
   * multi-value counters to have exactly two values.)
   * @returns True if the event was emitted; false if the event was not emitted because tracing
   *   was not enabled for any of the event categories.
   */
  count(
    category: string | string[],
    name: string,
    value: number | { [name: string]: number } | () => (number | { [name: string]: number })): boolean;
}
```

## 3. Tracing listener APIs
The tracing listener APIs enable JavaScript code to listen to near-real-time tracing events in
specific categories, rather than waiting for the events be recorded to a file for offline analysis.
Tracing events emitted in both JavaScript can C++ can be received by both JavaScript and C++
listeners.

Relatively few applications and libraries are expected to use the listener APIs (as compared
to the much broader applicability of the emitter APIs). However, the ability to listen to tracing
events could be very valuable to certain application-monitoring, profiling, and diagnostic tools
and SDKs.

To support listeners, the `Tracing` class (from the previous section) extends a new
`CategoryEventEmitter` base class. That class is similar to `EventEmitter`, but does not extend
`EventEmitter` because it has different behavior related to the multi-category support.
`CategoryEventEmitter` is designed to be non-tracing-specific, so it could be useful in other
situations where multi-category event support is desired, however no specific other use cases
are known at this time.

Listening works whether or not the category is being recorded; in other words, adding a listener
automatically enables the category for listening mode (but not necessarily recording), and removing
the last listener disables the category for listening. So when nobody is listening (or recording)
no events are emitted and the cost of tracing emit calls is near zero.

### 3.1 Example use
```javascript
// Single-category listening and emitting:
tracing.on('example', e => { // e is a TracingEvent object
  if (e.name === 'info') {
    console.log('Got a tracing event: ' + e.args.message);
  }
});
tracing.emit('example', 'info', 'Hello tracing!');

// Multi-category listening and emitting:
// The listener receives events emitted for *any* matching category.
tracing.on(['example1', 'example2'] , e => { // e is a TracingEvent object
  if (e.name === 'info') {
    console.log('Got a tracing event: ' + e.args.message);
  }
});
tracing.emit(['example2', 'example3'], 'info', 'Hello tracing!');
```

### 3.2 Listener API specification
```typescript
interface Tracing extends CategoryEventEmitter ...

/**
 * Enables emitting and listening to events in multiple categories.
 *
 * This interface has methods that are similar to EventEmitter, with an important difference:
 * events may be emitted under multiple categories simultaneously instead of a single event name,
 * and listeners may listen to multiple categories while still only being called once per event.
 */
interface CategoryEventEmitter {
  /**
   * Emits an event for a category or categories.
   *
   * @param category Required category name or array of one or more category names.
   * @param args Optional arguments for the event to be emitted.
   * @returns True if the event was emitted; false if the event was not emitted because there
   *   were no listeners for any of the categories.
   */
  emit(category: string | string[], ...args: any[]): boolean;

  /**
   * Alias for addListener. Adds a listener for one or more categories of events.
   *
   * @param category Required category name or array of one or more category names to listen to.
   * @param listener Receives events of the requested category or categories. A listener is
   *   invoked only once per event, even for a multi-category event.
   */
  on(category: string | string[], listener: Function): void;

  /**
   * Adds a listener for one or more categories of events.
   *
   * @param category Required category name or array of one or more category names to listen to.
   * @param listener Receives events of the requested category. or categories. A listener is
   *   invoked only once per event, even for a multi-category event.
   */
  addListener(category: string | string[], listener: Function): void;

  /**
   * Removes a listener for one or more categories of events.
   *
   * @param category Required category name or array of one or more category names to stop
   * listening to.
   * @param listener The listener to remove.
   */
  removeListener(category: string | string[], listener: Function): void;

  /**
   * Removes all listeners for one or more categories of events, or removes all listeners
   * for all categories if no category argument is supplied.
   *
   * @param category Required category name or array of one or more category names to stop
   *   listening to.
   */
  removeAllListeners(category?: string | string[]): void;

  /**
   * Gets all the listeners for one or more categories of events.
   *
   * @param category Optional category name or array of one or more category names to retrieve
   *   listeners for.
   * @returns Array of listeners for the specified categories. Even if a listener is registered
   *   for more than one of the categories, it is only included in the list once.
   */
  listeners(category?: string | string[]): Function[];

  /**
   * Gets a count of all the listeners for one or more categories of events.
   *
   * @param category Optional category name or array of one or more category names to count
   *   listeners for.
   * @returns Count of listeners for the specified categories. Even if a listener is registered
   *   for more than one of the categories, it is only counted once.
   */
  listenerCount(category?: string | string[]): number;

  /**
   * Gets a list of all the categories having registered listeners.
   */
  listenerCategories(): string[];
}
```

## 4. Console tracing API extensions
The JavaScript `console` module (in Node.js and the web) already defines a set of basic tracing
methods that align well with the four tracing event types described above. These methods are NOT
part of any official standard, though there is
[a working group that maintains a spec](https://github.com/whatwg/console)
for the purpose of encouraging interoperability.

The existing Node.js implementation of those Console tracing methods simply logs tracing events to
the console. This proposal extends those methods with a new optional category parameter to enable
emitting categorized tracing events in a way that is backward-compatible with existing console APIs.

### 4.1 Example use
```javascript
// Emit { eventType: 'instant', category: 'console', name: 'myEvent'}
console.timeStamp('myEvent');
// Emit { eventType: 'instant', category: 'example', name: 'myEvent'}
console.timeStamp('myEvent', 'example');

// Emit { eventType: 'begin', category: 'example', name: 'myTask'}
console.time('myTask', 'example');
// Emit { eventType: 'end', category: 'example', name: 'myTask'}
console.timeEnd('myTask', 'example');

// Emit { eventType: 'count', category: 'console', name: 'myCounter', value = 1 }
console.count('myCounter');
// Emit { eventType: 'count', category: 'example', name: 'myCounter', value = 2 }
console.count('myCounter', 'example');
```

### 4.2 Console API specification
```typescript
/**
 * Partial interface for the Node.js Console module showing added and updated methods for tracing.
 * @see {@link https://nodejs.org/dist/latest-v7.x/docs/api/console.html}
 */
interface Console {

  /**
   * Keeps a count of the number of times count() has been invoked with the provided name,
   * and emits a "count" tracing event each time. Also logs the counter name and value to the
   * console, regardless of whether tracing is enabled for any category.
   *
   * @param name Required name of the counter.
   * @param [category] Optional category name or array of category names for the tracing event.
   *   If unspecified, then 'console' is used.
   *
   * NEW NODE.JS METHOD, EXISTING WEB METHOD + NEW OPTIONAL PARAMETERS FOR TRACING
   * @see {@link https://developer.mozilla.org/en-US/docs/Web/API/Console/count}
   * @see {@link https://console.spec.whatwg.org/#count}
   */
  count(name: string, category?: string | string[]): void;

  /**
   * Emits a "begin" tracing event. Nothing is logged to the console until the
   * corresponding timeEnd() is called.
   *
   * @param name Required name of the timer.
   * @param [category] Optional category name or array of category names for the tracing event.
   *   If unspecified, then 'console' is used.
   *
   * EXISTING NODE.JS & WEB METHOD + NEW OPTIONAL PARAMETERS FOR TRACING
   * @see {@link https://nodejs.org/dist/latest-v7.x/docs/api/console.html#console_console_time_label}
   * @see {@link https://developer.mozilla.org/en-US/docs/Web/API/Console/time}
   * @see {@link https://console.spec.whatwg.org/#time}
   */
  time(name: string, category?: string | string[]): void;


  /**
   * Emits an "end" tracing event. Also logs the event name and duration to the console, regardless
   * of whether tracing is enabled for any category.
   *
   * @param name Required name of the timer; must match the name from a prior call to time().
   * @param [category] Optional category name or array of category names for the tracing event.
   *   If unspecified, then 'console' is used.
   *
   * EXISTING NODE.JS & WEB METHOD + NEW OPTIONAL PARAMETERS FOR TRACING
   * @see {@link https://nodejs.org/dist/latest-v7.x/docs/api/console.html#console_console_timeend_label}
   * @see {@link https://developer.mozilla.org/en-US/docs/Web/API/Console/timeEnd}
   * @see {@link https://console.spec.whatwg.org/#timeend}
   */
  timeEnd(name: string, category?: string | string[]): void;

  /**
   * Emits an "instant" tracing event. Also logs the event name and timestamp to the console,
   * regardless of whether tracing is enabled for any category.
   *
   * @param name Required name of the time stamp.
   * @param [category] Optional category name or array of category names for the tracing event.
   *   If unspecified, then 'console' is used.
   *
   * NEW NODE.JS METHOD, EXISTING WEB METHOD + NEW OPTIONAL PARAMETERS FOR TRACING
   * @see {@link https://developer.mozilla.org/en-US/docs/Web/API/Console/timeStamp}
   */
  timeStamp(name: string, category?: string | string[]): void;

  /**
   * Writes a formatted message to the console with a stack trace.
   *
   * NOTE: This is not actually related to tracing, but is listed here because its name may
   * cause some confusion.
   *
   * EXISTING NODE.JS & WEB METHOD (NO CHANGE)
   * @see {@link https://nodejs.org/dist/latest-v7.x/docs/api/console.html#console_console_trace_message_args}
   * @see {@link https://developer.mozilla.org/en-US/docs/Web/API/Console/trace}
   * @see {@link https://console.spec.whatwg.org/#trace}
   */
  trace(message?: any, ...optionalParams: any[]): void;

  //
  // Other non-tracing-related Console methods are omitted here.
  //
}
```

## 5. Implementation status
### 5.1 Tracing fork
Work is in progress at https://github.com/jasongin/nodejs/tree/tracejs. At the time of writing,
tracing emitter APIs and console tracing API extensions are functional.

The `CategoryEventEmitter` base class is implemented (with tests), however the full tracing
listener implementation has a dependency on an "event callback" mode that is currently
unimplemented in the v8 tracing code.

### 5.2 Performance
Performance for emitting tracing events from JavaScript is looking pretty good so far; I've done
some basic optimization but not extensive tuning. On my dev system, a 5-year old desktop running
Windows, the benchmark I wrote shows each single-category JS trace statement takes around **650ns
with tracing enabled** (writing events to a JSON file), or around **45ns with tracing disabled**.
Multi-category events are somewhat slower. Yes those numbers are nano-seconds, not micro- or
milli-seconds.

A more detailed performance analysis should be performed around the time of the pull request.

## 6. Future work
The following related items are outside the scope of this proposal, at least for now.

### 6.1 Output serialization
Currently, logs are simply JSON-serialized to a file; this JSON format is
[understood by the Chrome dev tools](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool). The C++ layer of the
tracing framework supports pluggable exporters, for example Chromium has an
[ETW exporter for Windows](https://chromium.googlesource.com/chromium/src/+/master/base/trace_event/trace_event_etw_export_win.h),
which way may want to adapt to Node.js eventually. And there has been discussion of a more compact
binary format. This is an area that needs some further development, however for performance reasons
the tracing output probably would not be exposed to JavaScript anyway, beyond the listener APIs
that are proposed here.

### 6.2 Instrumenting core modules
To improve performance diagnostic capabilities, we may want to instrument some Node core modules
(both C++ and JavaScript code) with tracing. Of course, care should be taken to avoid any
performance penalty when tracing is disabled.

### 6.3 Polyfill
It could be possible to create a JavaScript polyfill for tracing, to enable libraries instrumented
with tracing statements to also work on older versions of Node.js. The polyfill might implement the
tracing APIs as no-ops, or might provide full functionality, but with poorer performance compared
to the C++ background-thread implementation, and no abiltity to listen to C++ trace events.

### 6.4 Automatic Instrumentation
It could be possible to automatically instrument JavaScript libraries with tracing events, so that
begin/end events are emitted for every function call. The [Google Web Tracing Framework](http://google.github.io/tracing-framework/instrumenting-code.html#automatic-instrumentation)
is an example of a JavaScript tracing framework with automatic instrumentation. This function would
be better done in a non-core library, therefore it is not covered in this document.

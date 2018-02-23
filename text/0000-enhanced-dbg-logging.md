- Feature Name: enhanched-dbg-loggig
- Start Date: 2018-02-22
- RFC PR: (leave this empty)
- Pony Issue: (leave this empty)

# Summary

This RFC provides for enhanced debug logging capability. The implementatipon will support debug logging in the compiler, runtime and in pony and controlled by an arbitrary set of bits. It will be able to use any mechanizm which can write via C format style function. The initial implemenation should support logging to in-memory circular buffers, to FILE\* and in pony code to OutSream's.


# Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with the language to understand, and for somebody familiar with the compiler to implement. This should get into specifics and corner-cases, and include examples of how the feature is used.

# How We Teach This

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Pony patterns, or as a wholly new one?

Would the acceptance of this proposal mean the Pony guides must be re-organized or altered? Does it change how Pony is taught to new users at any level?

How should this feature be introduced and taught to existing Pony users?

# How We Test This

How do we assure that the initial implementation works? How do we assure going forward that the new functionality works after people make changes? Do we need unit tests? Something more sophisticated? What's the scope of testing? Does this change impact the testing of other parts of Pony? Is our standard CI coverage sufficient to test this change? Is manual intervention required?

In general this section should be able to serve as acceptance criteria for any implementation of the RFC.

# Drawbacks

Why should we *not* do this? Things you might want to note:

* Breaks existing code
* Introduces instability into the compiler and or runtime which will result in bugs we are going to spend time tracking down
* Maintenance cost of added code

# Alternatives

What other designs have been considered? What is the impact of not doing this?
None is not an acceptable answer. There is always to option of not implementing the RFC.

# Unresolved questions

What parts of the design are still TBD?
-------
This PR provides enhanced debug capability. It supports debug logging in the compiler, runtime and in pony. It can write the debug information to in-memory circular buffers or to FILE\* and in pony code it can write it to OutSream's.

The basic design uses an arbitrarily sized bit map and any single bit can be used to trigger one or more debug print statements and these control bits can be set/cleared at runtime. There can be multiple instances of debug contexts and at this time there are no locks on the buffer so it's a requirement that each thread/actor have its own buffer. Additional design work and code is needed to dump the memory buffers when a program exists or faults.

Right now packages/dbg/dbg.pony definitely needs work, support for outputing the location information is needed and the DbgReadBuf class is using malloc/free to create read buffers and uses String.from_cstring to convert the a buffer to a Pony string, there must be a better way.

Also, on the Pony side it might be appropriate to enhance logger instead of having a separate class, but for now I didn't want to break any existing code, so I created packages/dbg. In examples/dbg/main.pony I do show using DC bits to control logger.log statements.

Looking forward to feed back.
-------
- dbg.[hc] is implemented in libpontyrt which allows the debug code to be used in the compiler and from Pony code.
- dbg.[hc] supports directly emitting code to FILE* or memory buffers, the supporting code in pony, packages/dbg/dbg.pony, adds support for OutStream.
- The fine control is achieved by using an array of bits and allowing any bit to be read and/or written at runtime.
  - I needed this while looking at src/libponyc/pass. I sprinkled printf and ast_print in the code and was innudated with output not associated with the single class that I was interested in. So using dbg.[hc] allowed me to significantly reduce the output and focus on answering the questions I had. As an example of this I created [add-dummy-pass](https://github.com/ponylang/ponyc/compare/master...winksaville:add-dummy-pass) which turns on debugging only when its processing 'class C'.
  - The array of bits can also be used with the logger class. An example is [here](https://github.com/ponylang/ponyc/compare/master...winksaville:add-dbg#diff-7d79e820d124b88617e5e1717683379bR10).
- The memory buffer is implemented as a circular buffer.
  - This allows a relatively small buffer to capture pontentially important information for a long running application in the event of crash or other important event. Additional work is needed to be able to "read" the memory buffer, but by logging to memory buffer it is possible.
  - I had initially used fmemopen, which allows a memory buffer to be used in the context of a FILE*, but this is not available on Windows or MacOSX.
  - The implementation of the circular buffer is such that the first N characters are preserved if output needs to be truncated. I did this as I felt that more important information tends to be at the beginning of a particular output, for instance "__loc" information.
